---
title: Redis Streams Pending 메시지 자동 복구 시스템 구현하기
description: 예외 발생으로 ACK되지 않은 Pending 메시지를 자동으로 재시도하고, 최종 실패 시 DLQ로 이동하는 복구 시스템을 구현합니다.
date: 2025-10-19
categories: [Backend, Redis]
tags: [Redis, Streams, Pending Messages, Message Recovery, DLQ, Scheduler]
pin: false
math: false
mermaid: false
---

## 1. 문제 상황: Pending 메시지가 쌓이고 있다

[이전 포스트](https://somln.github.io/posts/redis-streams1/)에서 Redis Streams를 활용한 채팅 시스템을 구현했다. Consumer Group과 ACK 메커니즘으로 메시지 순서 보장과 영구 저장을 달성했지만, 중요한 문제가 남아있었다.

**예외가 발생하면 메시지가 영원히 Pending 상태로 남는다.**

```redis
127.0.0.1:6379> XPENDING chat:stream:abc-123 chat-consumer-group
1) (integer) 3                    # Pending 메시지 3개
2) "1760861533323-0"              # 가장 오래된 메시지 ID
3) "1760861542040-0"              # 가장 최근 메시지 ID
4) 1) 1) "chat-consumer"          # Consumer 이름
      2) "3"                     
```

위 결과는 의도적으로 장애 상황을 만들어 ACK를 보내지 않았을 때 XPENDING 조회 결과이다. 해당 메시지들은 현재 재처리 로직이 없기 때문에 이 상태로 계속 남아있게 된다.

---

## 2. 설계 고민: 언제, 어떻게 재시도할 것인가?

### 고민 1: 스케줄러 실행 주기

**너무 짧으면:**
- Redis 부하 증가
- 정상 처리 중인 메시지까지 간섭할 위험

**너무 길면:**
- 메시지 지연 발생
- 사용자 경험 저하

채팅은 실시간성이 중요하므로 **30초 주기**로 결정했다. 대부분의 일시적 오류는 30초 내에 복구되고, Redis 부하도 감당 가능한 수준이다.

```java
@Scheduled(initialDelay = 0, fixedDelay = 30000)
public void retryPendingMessages() {
    // 30초마다 실행
}
```

### 고민 2: Pending 임계값 설정

메시지를 읽은 지 얼마나 지나야 "재시도가 필요한 상태"로 볼 것인가?

**일반적인 채팅 메시지 처리 시간:**
```
메시지 읽기 (XREADGROUP): 1~5ms
↓
처리 로직 (WebSocket 전송 + DB 저장): 50~200ms
↓
ACK: 1~5ms
```

정상적이라면 200ms 내외로 처리된다.

하지만 너무 짧은 임계값은 정상 처리 중인 메시지까지 재시도하게 만들 수 있다. 일시적인 네트워크 지연이나 DB 부하로 1~2초 정도 지연될 수 있기 때문이다.

고민 끝에 **5초**로 설정했다. 일반적인 처리 시간(200ms)보다 충분히 길고, 실패한 메시지를 빠르게 감지할 수 있는 균형점이다.

```java
private static final long PENDING_THRESHOLD_MILLIS = 5000; // 5초
```

**실제 복구 타임라인:**
```
메시지 처리 실패: 0초
↓
Pending idle time 5초 도달: 5초
↓
스케줄러 실행: 최대 30초 (평균 15초)
↓
총 복구 시간: 5~35초 (평균 20초)
```

채팅 메시지가 20초 정도 지연되는 것은 일시적 장애 상황에서 허용 가능한 수준이라고 생각한다.

### 고민 3: XAUTOCLAIM의 부재

Redis 6.2부터 `XAUTOCLAIM` 명령어가 추가되어, `XPENDING`과 `XCLAIM`을 한 번에 처리할 수 있다.

```redis
# Redis 6.2+
XAUTOCLAIM mystream mygroup myconsumer 3600000 0-0 COUNT 10
```

하지만 Spring Data Redis에는 `XAUTOCLAIM`에 대응하는 메서드가 없었다. 결국 `XPENDING`으로 조회 → 필터링 → `XCLAIM`으로 재할당하는 2단계 접근을 선택했다.

```java
// 1단계: XPENDING으로 모든 Pending 메시지 조회
PendingMessages pendingMessages = redisTemplate.opsForStream()
    .pending(streamKey, CONSUMER_GROUP, Range.unbounded(), 100L);

// 2단계: 조건에 맞는 메시지만 필터링
for (PendingMessage pending : pendingMessages) {
    if (idleTime >= PENDING_THRESHOLD_MILLIS) {
        retryableIds.add(RecordId.of(pending.getIdAsString()));
    }
}

// 3단계: XCLAIM으로 재할당
List<MapRecord<String, Object, Object>> claimed = redisTemplate.opsForStream()
    .claim(streamKey, CONSUMER_GROUP, CONSUMER_NAME, 
           Duration.ofMillis(PENDING_THRESHOLD_MILLIS),
           retryableIds.toArray(new RecordId[0]));
```

이 방식은 추가 네트워크 호출이 필요하지만, 필터링 로직을 세밀하게 제어할 수 있다는 장점이 있다.

---

## 3. 구현: Pending 메시지 자동 복구 시스템

### 전체 구조

```
[스케줄러 - 30초마다 실행]
    ↓
[1] 모든 채팅 스트림 조회
    ↓
[2] 각 스트림의 Pending 메시지 조회 (XPENDING)
    ↓
[3] 조건별 분류
    ├─ 5초 이상 & 2번 미만 재시도 → XCLAIM으로 재처리
    └─ 2번 이상 재시도 → DLQ로 이동
    ↓
[4] DLQ 처리
```

### Step 1: 모든 채팅 스트림 찾기

30초마다 실행되는 스케줄러가 Redis에 존재하는 모든 채팅 스트림을 찾는다.

```java
@Scheduled(initialDelay = 0, fixedDelay = 30000)
public void retryPendingMessages() {
    // 모든 채팅 스트림 키 조회
    Set<String> streamKeys = findAllChatStreams();
    if (streamKeys.isEmpty()) {
        return;
    }

    // 각 스트림의 Pending 메시지 처리
    for (String streamKey : streamKeys) {
        processPendingMessages(streamKey);
    }
}

private Set<String> findAllChatStreams() {
    Set<String> keys = redisTemplate.keys("chat:stream:*");
    return (keys != null) ? keys : Collections.emptySet();
}
```

**실행되는 Redis 명령:**
```redis
KEYS chat:stream:*
→ 1) "chat:stream:room-000"
  2) "chat:stream:room-123"
  3) "chat:stream:room-456"
```

### Step 2: Pending 메시지 조회

각 스트림에 대해 `XPENDING` 명령으로 처리되지 않은 메시지를 확인한다.

```java
private List<MapRecord<String, String, String>> claimRetryableMessages(String streamKey) {
    // Pending 메시지 조회
    PendingMessages pendingMessages = redisTemplate.opsForStream()
            .pending(streamKey, CONSUMER_GROUP, Range.unbounded(), 100L);

    if (pendingMessages == null || pendingMessages.isEmpty()) {
        return Collections.emptyList();
    }
    
    // ... 필터링 로직
}
```

**실행되는 Redis 명령:**
```redis
XPENDING chat:stream:abc-123 chat-consumer-group - + 100
→ 1) 1) "1760861533323-0"     # 메시지 ID
     2) "chat-consumer"       # Consumer 이름
     3) 12000                 # Idle time (12초)
     4) 1                     # 전달 횟수 (1번)
  2) 1) "1760861537875-0"
     2) "chat-consumer"
     3) 10000                 # 10초
     4) 1
  3) 1) "1760861542040-0"
     2) "chat-consumer"
     3) 8000                  # 8초
     4) 1
```

**이때 Redis 상태:**
```redis
# Consumer Group 상태
XINFO GROUPS chat:stream:abc-123
→ pending: 3                  # Pending 메시지 3개

# Consumer 상태
XINFO CONSUMERS chat:stream:abc-123 chat-consumer-group
→ 1) name: "chat-consumer"
  2) pending: 3               # 처리 중인 메시지 3개
  3) idle: 12000              # 마지막 활동 후 12초 경과
```

### Step 3: 재시도 대상 필터링

조회된 Pending 메시지를 조건에 따라 분류한다.

```java
private static final int MAX_RETRY_COUNT = 1  // 최대 1번 재시도

List<RecordId> retryableIds = new ArrayList<>();

for (PendingMessage pending : pendingMessages) {
    long deliveryCount = pending.getTotalDeliveryCount();
    long idleTime = pending.getElapsedTimeSinceLastDelivery().toMillis();

    // 최대 재시도 횟수 초과 시 DLQ로 이동
    if (deliveryCount > MAX_RETRY_COUNT) {
        moveToDLQ(streamKey, pending);
    } 
    // 5초 이상 idle이고 1번 이하면 재시도
    else if (idleTime >= PENDING_THRESHOLD_MILLIS) {
        retryableIds.add(RecordId.of(pending.getIdAsString()));
    }
}
```

> **deliveryCount와 재시도 횟수:**
> - deliveryCount 1: 최초 읽기
> - deliveryCount 2: 1차 재시도 → 이후 DLQ
>
> 즉, `MAX_RETRY_COUNT = 1`이면 `deliveryCount > 1` (즉, 2 이상)일 때 DLQ로 이동한다.

**분류 결과 (로그):**
```
Found 3 pending messages in stream: chat:stream:abc-123
- 1760861533323-0: idle=12000ms, deliveryCount=1 → 재시도 대상
- 1760861537875-0: idle=10000ms, deliveryCount=1 → 재시도 대상
- 1760861542040-0: idle=8000ms, deliveryCount=1 → 재시도 대상
```

### Step 4: XCLAIM으로 메시지 재할당

재시도 대상 메시지들을 `XCLAIM`으로 가져온다.

```java
if (!retryableIds.isEmpty()) {
    List<MapRecord<String, Object, Object>> claimed = redisTemplate.opsForStream()
            .claim(streamKey, CONSUMER_GROUP, CONSUMER_NAME,
                    Duration.ofMillis(PENDING_THRESHOLD_MILLIS),
                    retryableIds.toArray(new RecordId[0]));
    
    return claimed.stream()
            .map(this::convertToStringRecord)
            .toList();
}
```

**실행되는 Redis 명령:**
```redis
XCLAIM chat:stream:abc-123 
       chat-consumer-group 
       chat-consumer 
       5000  # min-idle-time: 5000ms 이상 idle인 메시지만 가져옴
       1760861533323-0 1760861537875-0 1760861542040-0

→ 1) 1) "1760861533323-0"
     2) {roomUUID: "abc-123", senderId: "1", message: "안녕"}
  2) 1) "1760861537875-0"
     2) {roomUUID: "abc-123", senderId: "2", message: "반가워"}
  3) 1) "1760861542040-0"
     2) {roomUUID: "abc-123", senderId: "1", message: "ㅎㅇ"}
```

> **min-idle-time 파라미터:**
> `Duration.ofMillis(PENDING_THRESHOLD_MILLIS)` (5000ms)는 최소 idle time을 의미한다.
> 이 시간보다 짧게 idle 상태인 메시지는 claim하지 않아, 정상 처리 중인 메시지와의 충돌을 방지한다.

**XCLAIM 후 Redis 상태:**
```redis
# Pending 상태 유지 (아직 ACK 안 함)
XPENDING chat:stream:abc-123
→ pending: 3                  # 여전히 3개 (소유권만 재할당됨)

# 하지만 idle time과 deliveryCount가 업데이트됨
XPENDING chat:stream:abc-123 chat-consumer-group - + 100
→ 1) 1) "1760861533323-0"
     2) "chat-consumer"       # 동일한 Consumer (단일 Consumer 구조)
     3) 100                   # idle time 리셋 (거의 0)
     4) 2                     # deliveryCount 증가 (1 → 2)
```

> 현재는 단일 서버 환경이므로 Consumer 이름이 동일하다.
> 여러 서버로 확장할 경우 각 서버마다 다른 Consumer 이름을 사용해야 하며, 
> 그 경우 XCLAIM은 다른 Consumer가 실패한 메시지를 가져가는 방식으로 동작한다.

### Step 5: 메시지 재처리 및 ACK

Claim된 메시지를 RedisSubscriber에게 전달하여 재처리한다.

```java
private void processPendingMessages(String streamKey) {
    List<MapRecord<String, String, String>> messages = claimRetryableMessages(streamKey);
    messages.forEach(redisSubscriber::onMessage);
}
```

**RedisSubscriber에서 재처리:**
```java
@Override
public void onMessage(MapRecord<String, String, String> message) {
    try {
        // WebSocket으로 전송 (이번엔 성공)
        messagingTemplate.convertAndSend(destination, chatMessage);

        // ACK 처리
        redisTemplate.opsForStream().acknowledge(streamKey, CONSUMER_GROUP, messageId);
        
    } catch (Exception e) {
        // 또 실패하면 ACK 안 함 → 다음 스케줄러 실행 때 다시 시도
    }
}
```

**실행되는 Redis 명령 (각 메시지마다):**
```redis
# 메시지 1 ACK
XACK chat:stream:abc-123 chat-consumer-group 1760861533323-0
→ 1

# 메시지 2 ACK
XACK chat:stream:abc-123 chat-consumer-group 1760861537875-0
→ 1

# 메시지 3 ACK
XACK chat:stream:abc-123 chat-consumer-group 1760861542040-0
→ 1
```

### Step 6: 복구 완료 확인

**재시도 성공 후 (스케줄러 실행 이후):**

모든 메시지가 성공적으로 처리되면 Pending이 비워진다.

```redis
127.0.0.1:6379> XPENDING chat:stream:abc-123 chat-consumer-group
1) (integer) 0                # Pending 메시지 0개!
2) (nil)
3) (nil)
4) (nil)
```

**Consumer Group 상태:**
```redis
XINFO GROUPS chat:stream:abc-123
→ pending: 0                  # 모두 처리됨
→ last-delivered-id: "1760861542040-0"  # 마지막 처리된 메시지

XINFO CONSUMERS chat:stream:abc-123 chat-consumer-group
→ 1) name: "chat-consumer"
  2) pending: 0               # 처리 중인 메시지 없음
  3) idle: 100ms              # 방금 활동함
```

---

## 4. DLQ (Dead Letter Queue) 처리

총 2번 시도해도 실패하는 메시지는 DLQ로 이동한다.

### DLQ 이동 로직

```java
private void moveToDLQ(String streamKey, PendingMessage pending) {
    String messageId = pending.getIdAsString();

    try {
        // 1. 메시지 Claim (내용 가져오기)
        List<MapRecord<String, Object, Object>> claimed = redisTemplate.opsForStream()
                .claim(streamKey, CONSUMER_GROUP, CONSUMER_NAME, 
                       Duration.ZERO, RecordId.of(messageId));

        // 2. DLQ 데이터 준비 (메타데이터 추가)
        Map<String, Object> dlqData = new HashMap<>(claimed.get(0).getValue());
        dlqData.put("_originalMessageId", messageId);
        dlqData.put("_failedAt", System.currentTimeMillis());
        dlqData.put("_attempts", pending.getTotalDeliveryCount());

        // 3. DLQ 스트림에 추가
        String dlqKey = streamKey + ":dlq";
        redisTemplate.opsForStream().add(dlqKey, dlqData);

        // 4. 원본 스트림에서 ACK (Pending 제거)
        redisTemplate.opsForStream().acknowledge(streamKey, CONSUMER_GROUP, messageId);

    } catch (Exception e) {
        log.error("Failed to move message {} to DLQ", messageId, e);
    }
}
```

> ACK 후에도 원본 Stream에는 메시지가 남아있다.
> 
> ```redis
> # ACK 후에도 원본 메시지는 유지됨
> XRANGE chat:stream:abc-123 - +
> → 1) "1760861999999-0" {roomUUID: "abc-123", ...}
> ```
> 
> 만약 원본 메시지도 삭제하려면 `XDEL`을 추가로 실행해야 한다.
> ```java
> // 선택: 원본 메시지 삭제
> redisTemplate.opsForStream().delete(streamKey, messageId);
> ```

**DLQ 이동 시나리오:**
```
메시지 ID: 1760861999999-0
deliveryCount: 2 (이미 2번 시도 실패)

↓

[1] XCLAIM으로 메시지 내용 가져오기
XCLAIM chat:stream:abc-123 chat-consumer-group chat-consumer 0 1760861999999-0
→ {roomUUID: "abc-123", senderId: "5", message: "테스트"}

[2] 메타데이터 추가
{
  "roomUUID": "abc-123",
  "senderId": "5",
  "message": "테스트",
  "_originalMessageId": "1760861999999-0",
  "_failedAt": 1697876543210,
  "_attempts": 4
}

[3] DLQ 스트림에 추가
XADD chat:stream:abc-123:dlq * <위 데이터>
→ "1697876999999-0" (새 DLQ 메시지 ID)

[4] 원본 스트림에서 ACK
XACK chat:stream:abc-123 chat-consumer-group 1760861999999-0
→ 1
```

**DLQ 조회:**
```redis
# DLQ 메시지 확인
XRANGE chat:stream:abc-123:dlq - +
→ 1) "1697876999999-0"
     {
       roomUUID: "abc-123",
       senderId: "5",
       message: "테스트",
       _originalMessageId: "1760861999999-0",
       _failedAt: "1697876543210",
       _attempts: "4"
     }

# DLQ 개수 확인
XLEN chat:stream:abc-123:dlq
→ 1
```

### DLQ 모니터링

운영 중에는 주기적으로 DLQ를 모니터링하여 시스템 상태를 확인한다.

```java
@Scheduled(cron = "0 */10 * * * *") // 10분마다
public void monitorDLQ() {
    Set<String> dlqKeys = redisTemplate.keys("chat:stream:*:dlq");
    
    for (String dlqKey : dlqKeys) {
        Long size = redisTemplate.opsForStream().size(dlqKey);
        
        if (size != null && size > 0) {
            log.warn("DLQ {} has {} failed messages!", dlqKey, size);
            
            // 임계값 초과 시 알림
            if (size > 100) {
                sendAlert("DLQ overflow: " + dlqKey);
            }
        }
    }
}
```

---

## 5. 전체 흐름 요약

### 정상 메시지 흐름
```
[메시지 발행]
    ↓
XADD chat:stream:abc-123
    ↓
[Consumer 읽기]
    ↓
XREADGROUP (메시지 → Pending)
    ↓
[처리 성공]
    ↓
XACK (Pending → 제거)
    ↓
완료 
```

### 실패 메시지 복구 흐름
```
[메시지 발행]
    ↓
XADD chat:stream:abc-123
    ↓
[Consumer 읽기]
    ↓
XREADGROUP (메시지 → Pending)
    ↓
[처리 실패] 
    ↓
ACK 안 함 (Pending 유지)
    ↓
[5초 경과]
    ↓
[스케줄러 발견 (30초마다 실행)]
    ↓
XPENDING (Pending 메시지 조회)
    ↓
XCLAIM (메시지 재할당)
    ↓
[재처리]
    ↓
[성공] → XACK 
[또 실패] → 다음 스케줄러 실행 때 재시도 
[2번째 실패] → DLQ 이동 
```

---

## 6. 결론

### 실제 검증 결과

**장애 발생 시 (재처리 전):**
```redis
127.0.0.1:6379> XPENDING chat:stream:abc-123 chat-consumer-group
1) (integer) 3                # Pending 메시지 3개
2) "1760861533323-0"          
3) "1760861542040-0"          
4) 1) 1) "chat-consumer"
      2) "3"
```

**스케줄러 실행 로그:**
```
[PendingRetryScheduler] Found 3 pending messages
[PendingRetryScheduler] Retrying message: 1760861533323-0 
[PendingRetryScheduler] Successfully retried and acknowledged message: 1760861533323-0
[PendingRetryScheduler] Retrying message: 1760861537875-0
[PendingRetryScheduler] Successfully retried and acknowledged message: 1760861537875-0
[PendingRetryScheduler] Retrying message: 1760861542040-0 
[PendingRetryScheduler] Successfully retried and acknowledged message: 1760861542040-0
```

**복구 완료 후:**
```redis
127.0.0.1:6379> XPENDING chat:stream:abc-123 chat-consumer-group
1) (integer) 0                # Pending 메시지 0개
2) (nil)
3) (nil)
4) (nil)
```

**Pending 상태가 지속되어 유실될 뻔한 메시지 3개가 모두 정상적으로 재처리되고 ACK 되었다.** 

기존 Redis Pub/Sub에서 메시지가 유실될 수 있는 문제를 해결하고 싶었고, Redis Streams를 도입하게 되었다. Redis Streams를 채팅에 도입한 예시는 거의 없어서 고민이 되었는데, 오히려 그래서 공식 문서를 더 찾아보고 깊게 공부할 수 있게 되어 좋았다.

재처리 로직을 완벽하게 설계했는지는 확실하지 않지만, 목표로 했던 **메시지 유실 문제 해결**은 충분히 달성한 것 같다. Redis Streams와 자동 재처리 메커니즘을 통해 채팅 시스템의 안정성을 크게 향상시킬 수 있었고, 더 이상 일시적인 장애로 인한 메시지 유실을 걱정하지 않아도 되게 되었다.