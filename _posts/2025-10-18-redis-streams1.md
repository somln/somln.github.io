---
title: Redis Pub/Sub의 메시지 유실 문제를 Redis Streams로 해결하기
description:  구독자 연결이 끊기면 메시지가 사라지는 Redis Pub/Sub의 한계를 극복하고, Consumer Group과 ACK 메커니즘으로 안정적인 채팅 시스템을 구축한 과정을 공유합니다.
date: 2025-10-18
categories: [Backend, Redis]
tags: [Redis, Streams, Chat, Pending Messages, Message Recovery]
pin: false
math: false
mermaid: false
---

## 1. 문제 정의 및 기술 선택
채팅 시스템을 처음 구현할 때는 Redis Pub/Sub을 사용했다. 구조가 단순하고 구현이 쉬웠기 때문이다. 하지만 Pub/Sub는 몇가지 단점이 있다.

- 메시지 유실: 구독자가 일시적으로 끊기면 그 사이 메시지는 저장되지 않고 사라진다.
- 복구 불가: 재연결 후에도 이전 메시지를 다시 받을 수 없다.
- 확장 한계: 여러 서버로 확장 시 모든 서버가 같은 메시지를 받아 중복 처리가 발생한다.

따라서 아래와 같은 이유로 기존 Redis Pub/Sub 기반을 Redis Streams로 마이그레이션했다.

- 기존 Redis 인프라를 그대로 활용할 수 있다.
- 메시지 순서 보장과 영구 저장이 가능하면서도, 메모리 기반이라 지연이 매우 낮다.
- Consumer Group을 통해 자연스럽게 부하 분산이 가능하다.
- 비슷한 메시지 큐 방식인 Kafka를 도입하는 것은 오버엔지니어링이라고 판단했다.

## 2. Redis Streams 핵심 개념

### Stream: 로그 기반 메시지 저장소

Redis Stream은 **Append-Only 로그 구조**로 메시지를 저장한다. 메시지를 추가하면 고유한 ID가 자동으로 생성되고, 시간순으로 정렬된다.

```redis
# 메시지 추가
XADD chat:stream:room-1 * message "안녕" senderId 1
→ "1704123456789-0"  # Entry ID (타임스탬프-시퀀스)

# 메시지 조회
XRANGE chat:stream:room-1 - +
→ 1) "1704123456789-0" {message: "안녕", senderId: "1"}
  2) "1704123456790-0" {message: "반가워", senderId: "2"}
```

**Entry ID 구조:**
```
1704123456789-0
     ↑        ↑
  밀리초    시퀀스번호
```

- Redis가 현재 시간으로 자동 생성 (`*` 사용 시)
- 같은 밀리초에 여러 메시지가 들어오면 시퀀스로 구분 (0, 1, 2, ...)
- 시간순 정렬이 자동으로 보장됨

**Stream 특징:**

- **영구 저장**: 명시적으로 삭제하기 전까지 유지
- **순서 보장**: Entry ID로 자동 정렬
- **다중 소비**: 여러 Consumer가 같은 메시지를 독립적으로 읽기 가능

---

### 메시지 읽기: XREAD vs XREADGROUP

**1) XREAD: 단순 읽기**

가장 기본적인 메시지 읽기 방법. Stream에서 메시지를 그냥 가져온다.

```redis
# 마지막 ID 이후 메시지 읽기
XREAD STREAMS chat:stream:room-1 1704123456789-0

# 새 메시지 대기 (Long Polling)
XREAD BLOCK 5000 STREAMS chat:stream:room-1 $
```

- 간단하지만 **메시지 분산 처리나 장애 복구 기능이 없음**
- 여러 Consumer가 읽으면 모두 같은 메시지를 받음

**2) XREADGROUP: 분산 처리**

Consumer Group을 사용한 고급 읽기 방법. 메시지를 여러 Consumer에게 분산하고, 처리 실패 시 복구가 가능하다.

```redis
# Consumer Group 생성
XGROUP CREATE chat:stream:room-1 chat-group $

# 메시지 읽기
XREADGROUP GROUP chat-group consumer-1 
           COUNT 10 
           BLOCK 3000 
           STREAMS chat:stream:room-1 >
```

---

### Consumer Group: 메시지 분산 처리

**핵심 원리:**

하나의 메시지는 Group 내 **한 Consumer에게만** 전달된다.

```
Stream: [메시지1] [메시지2] [메시지3] [메시지4]
           ↓        ↓        ↓        ↓
        Consumer1 Consumer2 Consumer1 Consumer2
```

**ReadOffset 종류:**

```java
// Consumer Group 생성 시
ReadOffset.latest()       // "$" - 이 시점 이후 메시지만 처리
                          // 채팅방 입장 시점부터 새 메시지만 보여주고 싶을 때

// 메시지 읽기 시 (XREADGROUP)
ReadOffset.lastConsumed() // ">" - Group이 아직 처리 안 한 메시지
                          // 일반적으로 항상 이것을 사용

// 과거 메시지 재처리 시
ReadOffset.from("0-0")    // "0" - 처음부터 모두
                          // 거의 사용하지 않음
```

**여러 Group 생성 가능:**

같은 Stream에 여러 용도의 Group을 만들 수 있다. 각 Group은 독립적인 offset을 가진다.

```
Stream: [메시지1] [메시지2] [메시지3]
          ↓  ↓  ↓
          ↓  ↓  └─→ Group C (분석용)
          ↓  └────→ Group B (알림용)
          └───────→ Group A (채팅 전송용)
```

---

### ACK와 Pending List: 메시지 유실 방지

**메시지 처리 흐름:**

```redis
# 1. 메시지 읽기 (자동으로 Pending 상태가 됨)
XREADGROUP GROUP chat-group consumer-1 STREAMS chat:stream:room-1 >
→ [1704123456789-0] {message: "안녕"}

# 2. 처리 로직 실행
# ...

# 3. 처리 완료 확인 (ACK)
XACK chat:stream:room-1 chat-group 1704123456789-0
```

**Pending List란?**

읽었지만 **아직 ACK되지 않은 메시지 목록**. Consumer가 죽어도 메시지를 잃어버리지 않도록 보호한다.

```redis
# Pending 메시지 확인
XPENDING chat:stream:room-1 chat-group
→ 1) "1704123456789-0"  # Message ID
  2) "consumer-1"       # 어느 Consumer가 읽었는지
  3) 30000             # 30초간 처리 안 됨
  4) 1                 # 전달 횟수
```

**재처리 (XCLAIM):**

Consumer가 죽어서 처리 못한 메시지를 다른 Consumer가 가져간다.

```redis
# 10분 이상 Pending인 메시지를 consumer-2가 가져가기
XCLAIM chat:stream:room-1 chat-group consumer-2 600000 1704123456789-0
```

---

## 3. 채팅 전송부터 수신까지 전체 흐름

### Step 1: 채팅방 생성 및 Stream 구독

**Stream 구독 로직:**
```java
public void joinChatRoom(String streamKey) {
    // 중복 구독 방지
    if (subscribedStreams.putIfAbsent(streamKey, true) != null) {
        return;
    }
    
    try {
        // 1. Consumer Group 생성 (Stream이 없으면 자동 생성)
        // XGROUP CREATE chat:stream:abc-123 chat-consumer-group $ MKSTREAM
        createConsumerGroupIfNotExists(streamKey);
        
        // 2. Stream 리스너 등록 - 백그라운드에서 메시지를 읽어올 설정
        streamMessageListenerContainer.receive(
            Consumer.from(CONSUMER_GROUP, CONSUMER_NAME),  // 어떤 Consumer Group과 이름으로 읽을지
            StreamOffset.create(streamKey, ReadOffset.lastConsumed()),  // 어느 위치부터 읽을지 (>)
            redisSubscriber  // 메시지를 받았을 때 실행할 리스너
        );
        
        log.info("Successfully subscribed to stream: {}", streamKey);
    } catch (Exception e) {
        subscribedStreams.remove(streamKey);
        throw e;
    }
}

private void createConsumerGroupIfNotExists(String streamKey) {
    try {
        redisTemplate.execute((RedisCallback<Object>) connection -> {
            connection.streamCommands().xGroupCreate(
                streamKey.getBytes(),   // Stream 키
                CONSUMER_GROUP,         // Consumer Group 이름
                ReadOffset.latest(),    // "$" - 지금부터 들어오는 메시지만 읽기
                true                   // MKSTREAM: Stream이 없으면 자동 생성
            );
            return null;
        });
    } catch (RedisBusyException e) {
        // Consumer Group이 이미 존재 - 정상 케이스
        log.debug("Consumer group already exists: {}", CONSUMER_GROUP);
    }
}
```

**이때 Redis 상태:**
```redis
# Stream 생성됨 (비어있음)
XLEN chat:stream:abc-123
→ 0

# Consumer Group 생성됨
XINFO GROUPS chat:stream:abc-123
→ 1) name: "chat-consumer-group"
  2) consumers: 1
  3) pending: 0
  4) last-delivered-id: "0-0"

# Consumer가 대기 중
XINFO CONSUMERS chat:stream:abc-123 chat-consumer-group
→ 1) name: "chat-consumer"
  2) pending: 0
  3) idle: 0ms
```

---

### Step 2: 메시지 전송

**클라이언트:**
```javascript
stompClient.send('/app/message', {}, JSON.stringify({
    roomUUID: 'abc-123',
    senderId: 1,
    message: '안녕하세요'
}));
```

**서버 - RedisPublisher:**
```java
public void publish(String streamKey, ChatMessage chatMessage) {
    try {
        Map<String, String> messageMap = new HashMap<>();
        messageMap.put("roomUUID", chatMessage.getRoomUUID());
        messageMap.put("senderId", String.valueOf(chatMessage.getSenderId()));
        messageMap.put("message", chatMessage.getMessage());
        
        redisTemplate.opsForStream().add(streamKey, messageMap);
        
        // Stream 크기 제한 (최근 1만 개만 유지)
        redisTemplate.opsForStream().trim(streamKey, 10000, true);
        
    } catch (Exception e) {
        log.error("Failed to publish message", e);
        throw new ChatPublishException("메시지 전송 실패", e);
    }
}
```

**실제 실행되는 Redis 명령:**
```redis
XADD chat:stream:abc-123 * roomUUID abc-123 senderId 1 message "안녕하세요"
→ "1704123456789-0"

# 메시지 추가 후 자동 정리
XTRIM chat:stream:abc-123 MAXLEN ~ 10000
```

**이때 Redis 상태:**
```redis
# Stream에 메시지 추가됨
XLEN chat:stream:abc-123
→ 1

XRANGE chat:stream:abc-123 - +
→ 1) "1704123456789-0"
     {roomUUID: "abc-123", senderId: "1", message: "안녕하세요"}

# Consumer Group은 아직 읽지 않음
XINFO GROUPS chat:stream:abc-123
→ last-delivered-id: "0-0"  # 아직 업데이트 안 됨
→ pending: 0
```
메시지 발행 시마다 자동으로 trim하므로, Stream 크기가 항상 일정하게 유지된다. 만약 별도 배치 작업으로 관리하고 싶다면 스케줄러를 사용할 수도 있다.

---

### Step 3: 메시지 감지 및 읽기

**StreamMessageListenerContainer가 백그라운드에서 폴링:**
```java
    @Bean
    public StreamMessageListenerContainer<String, ?> streamMessageListenerContainer(RedisConnectionFactory connectionFactory) {
        StreamMessageListenerContainerOptions<String, ?> options = StreamMessageListenerContainerOptions
                .builder()
                .pollTimeout(Duration.ofMillis(3000))
                .build();

        StreamMessageListenerContainer<String, ?> container =
                StreamMessageListenerContainer.create(connectionFactory, options);
        container.start();
        return container;
    }
```
pollTimeout(3000ms)는 Redis에 "새 메시지 있으면 줘, 없으면 3000ms 기다려"라고 요청하는 주기이다. 


내부적으로 3초마다 실행되는 Redis 명령:
```redis
XREADGROUP GROUP chat-consumer-group chat-consumer 
           COUNT 10 BLOCK 3000 
           STREAMS chat:stream:abc-123 >
→ [1704123456789-0] {roomUUID: "abc-123", senderId: "1", message: "안녕하세요"}
```

**이때 Redis 상태:**
```redis
# Consumer Group offset 업데이트됨
XINFO GROUPS chat:stream:abc-123
→ last-delivered-id: "1704123456789-0"  # 업데이트!
→ pending: 1  # 메시지 읽었지만 아직 ACK 안 함

# Pending List에 추가됨
XPENDING chat:stream:abc-123 chat-consumer-group
→ 1) "1704123456789-0"    # Message ID
  2) "chat-consumer"      # Consumer 이름
  3) 0                    # idle time (방금 읽음)
  4) 1                    # delivery count

# Consumer 상태
XINFO CONSUMERS chat:stream:abc-123 chat-consumer-group
→ 1) name: "chat-consumer"
  2) pending: 1  # 처리 중인 메시지 1개
  3) idle: 50ms
```

---

### Step 4: 메시지 처리 및 전달

**서버 - RedisSubscriber:**
```java
@Override
public void onMessage(MapRecord<String, String, String> message) {
    String streamKey = message.getStream();
    String messageId = message.getId().getValue();

    try {
        // 메시지 데이터 추출
        Map<String, String> messageMap = message.getValue();

        // ChatMessage 생성
        ChatMessage chatMessage = new ChatMessage();
        chatMessage.setRoomUUID(messageMap.get("roomUUID"));
        chatMessage.setSenderId(Long.parseLong(messageMap.get("senderId")));
        chatMessage.setMessage(messageMap.get("message"));

        // WebSocket 전송
        String destination = "/sub/chat/room/" + chatMessage.getRoomUUID();
        messagingTemplate.convertAndSend(destination, chatMessage);

        // ACK 처리
        redisTemplate.opsForStream().acknowledge(streamKey, CONSUMER_GROUP, messageId);

    } catch (Exception e) {
        // → Pending 유지 (재시도 가능)
        log.error("[RedisSubscriber] Processing failed, will retry: {}", messageId, e);
    }
}
```

**실행되는 Redis 명령:**
```redis
# ACK 처리
XACK chat:stream:abc-123 chat-consumer-group 1704123456789-0
→ 1  # ACK 성공
```

**이때 Redis 상태:**
```redis
# Pending 상태 해제
XINFO GROUPS chat:stream:abc-123
→ last-delivered-id: "1704123456789-0"
→ pending: 0  # ACK 완료!

# Pending List 비어있음
XPENDING chat:stream:abc-123 chat-consumer-group
→ (empty)

# Consumer의 pending도 0
XINFO CONSUMERS chat:stream:abc-123 chat-consumer-group
→ pending: 0

# 메시지는 여전히 Stream에 저장됨
XLEN chat:stream:abc-123
→ 1

XRANGE chat:stream:abc-123 - +
→ 1) "1704123456789-0"
     {roomUUID: "abc-123", senderId: "1", message: "안녕하세요"}
```

---

### Step 5: 클라이언트 수신

**클라이언트:**
```javascript
stompClient.subscribe('/sub/chat/room/abc-123', function(message) {
    const chatMessage = JSON.parse(message.body);
    displayMessage(chatMessage);
});
```

화면에 "안녕하세요" 메시지가 표시됨.

---

## 4. 결론
Redis Streams를 활용해 실시간 채팅 시스템의 기본 구조를 구현해봤다. Consumer Group과 ACK 메커니즘을 통해 메시지 순서 보장과 영구 저장이라는 두 조건을 모두 만족할 수 있었고, 기존 Redis Pub/Sub 대비 안정성도 크게 향상되었다.

하지만 현재 구현에는 중요한 부분이 빠져있다. 바로 예외 발생으로 ACK를 보내지 못한 Pending 메시지들을 어떻게 처리할 것인가 하는 문제다. 일시적인 네트워크 장애, DB 타임아웃, WebSocket 연결 끊김 등으로 메시지 처리에 실패하면, 해당 메시지는 Pending 상태로 남게 된다.

정상적인 경우라면 메시지 처리 후 즉시 ACK가 되어 Pending에서 제거되지만, 실패한 메시지는 계속 Pending 리스트에 쌓이게 된다. 이를 방치하면 사용자는 자신이 보낸 메시지가 전송되지 않은 것처럼 보이고, 시스템은 처리되지 않은 메시지로 가득 차게 된다.

다음 포스트에서는 Pending 메시지 자동 복구 시스템을 구현하는 과정을 다뤄볼 예정이다. 스케줄러를 통한 주기적인 Pending 메시지 감지, 재시도 전략, 그리고 최종 실패 시 DLQ 처리까지, Redis Streams를 활용한 메시지 유실 없는 안정적인 채팅 시스템을 구축하는 과정을 작성해보려 한다.