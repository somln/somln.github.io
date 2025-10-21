---
title: Spring Boot에서 Casbin으로 도메인 기반 RBAC + ACL 구현하기
description: 역할 기반 권한에 사용자별 예외 처리를 결합한 도메인 RBAC 구현, PostgreSQL 어댑터로 정책을 동적 관리하고 우선순위 기반 deny-override를 적용한 실전 사례를 다룹니다.
date: 2024-10-28   
categories: [Backend, 권한 관리, Casbin]   
tags: [Casbin, PostgreSQL, 권한 관리, 정책 저장]  
pin: false
math: false  
mermaid: false  
---

## 왜 RBAC만으로는 부족했을까
프로젝트를 진행하면서 역할 기반 권한 관리와 함께 사용자별 세밀한 조회가 필요한 요구사항이 있었다. "특정 사용자만 예외로 허용하기", "특정 리소스나 경로만 접근 금지하기" 같은 세밀한 제어가 필요한 상황이 자주 발생했기 때문이다. 

특히 멀티테넌트 환경에서는 더 복잡했다. 도메인별로 권한을 분리해야 하는 동시에, 역할에 따른 기본 권한 설정과 개별 사용자에 대한 예외 처리를 함께 다뤄야 했다. 단순히 `ADMIN`, `USER` 같은 역할만으로는 이런 요구사항을 충족시키기 어려웠다.

Casbin은 모델과 정책을 완전히 분리하는 구조를 제공해서 이런 문제를 깔끔하게 해결할 수 있었다. 도메인 기반 RBAC은 Casbin에서 공식적으로 지원하고 있고, 정책 우선순위나 deny-override 같은 세밀한 조정도 가능하다는 점이 큰 장점이었다. Casbin에 대한 설명은 [여기에](https://somln.github.io/posts/casbin/) 정리했다.

---

## 모델 설계: 도메인 + 역할 + 개인 예외 + 우선순위

핵심 아이디어는 정책에 **우선순위**와 **효과**를 추가하고, 매처에서 역할 기반 매칭과 사용자 직접 매칭을 모두 지원하는 것이다.

```ini
[request_definition]
r = sub, obj, act, dom

[policy_definition]
; priority 숫자가 작을수록 우선순위가 높음
; eft는 allow 또는 deny
p = priority, sub, obj, act, dom, eft

[role_definition]
; 도메인 스코프의 역할 상속
g = _, _, _

[policy_effect]
; 우선순위 기반으로 첫 번째 매칭의 eft를 따름 (deny-override 구현)
e = priority(p.eft) || deny-override

[matchers]
; (1) 도메인 스코프 역할 상속   (2) 사용자 직접 정책(ACL)
; (3) 경로 패턴 매칭            (4) 액션 정규식 매칭
m = (g(r.sub, p.sub, r.dom) || r.sub == p.sub)
    && keyMatch2(r.obj, p.obj)
    && regexMatch(r.act, p.act)
    && r.dom == p.dom
```

각 섹션을 하나씩 살펴보면:

**request_definition**: 권한 검사 시 전달되는 요청 인자를 정의한다. 여기서는 주체(sub), 객체(obj), 행동(act), 도메인(dom) 네 가지를 사용한다.

**policy_definition**: 정책의 구조를 정의한다. 우선순위를 맨 앞에 두고, 주체, 객체, 행동, 도메인, 그리고 효과를 순서대로 정의했다.

**role_definition**: `g = _, _, _` 형태로 역할 링크에 도메인 차원을 추가했다. 이렇게 하면 같은 사용자가 도메인별로 다른 역할을 가질 수 있다.

**policy_effect**: 우선순위를 기준으로 첫 번째로 매칭된 정책의 효과를 따른다. 이를 통해 deny-override 방식을 구현할 수 있다.

**matchers**: 실제 권한 검사 로직이다. `g(r.sub, p.sub, r.dom)`으로 도메인 스코프 역할 상속을 확인하고, `r.sub == p.sub`로 사용자 직접 매칭도 처리한다. 또한 `keyMatch2`로 경로 패턴 매칭을, `regexMatch`로 액션 정규식 매칭을 수행한다.

몇 가지 중요한 포인트를 짚어보자면:

- **도메인 스코프 RBAC**: 역할 링크에 도메인을 포함시켜서 멀티테넌트 환경을 지원한다.
- **우선순위**: 숫자가 작을수록 우선순위가 높다. 예를 들어 priority가 5인 정책이 priority가 100인 정책보다 먼저 평가된다.
- **deny-override**: deny 정책이 하나라도 매칭되면 허용을 덮어쓰는 방식을 간단하게 구현할 수 있다.
- **keyMatch2, regexMatch**: REST API 경로나 HTTP 메서드를 패턴으로 매칭할 수 있는 내장 함수들이다.

---

## 정책 예시: 역할은 기본, ACL은 예외

역할 기반 정책을 넓게 설정하고, 개인별 예외는 우선순위를 높게 설정하는 방식으로 구현할 수 있다.

```csv
; p, priority,     sub,            obj,                 act,            dom,           eft
; 역할 기본 권한
p, 100,            content_editor, /news/*,            (create|edit),  journalism,    allow
p, 100,            viewer,         /news/*,            read,           journalism,    allow

; 개인 예외(ACL) - 특정 사용자 허용
p, 10,             dave,           /confidential/*,    read,           journalism,    allow

; 개인 예외(ACL) - 특정 사용자 금지(deny-override)
p, 5,              bob,            /news/*,            edit,           journalism,    deny
```

이 정책을 해석해보면:

**역할 기본 권한 (priority 100)**:
- `content_editor` 역할을 가진 사용자는 journalism 도메인의 `/news/*` 경로에 대해 create와 edit 권한을 가진다.
- `viewer` 역할을 가진 사용자는 journalism 도메인의 `/news/*` 경로에 대해 read 권한만 가진다.

**개인 허용 예외 (priority 10)**:
- `dave`는 viewer 역할만 가지고 있지만, `/confidential/*` 경로는 읽을 수 있다. 이는 역할 권한보다 우선순위가 높아서 개인 예외로 처리된다.

**개인 금지 예외 (priority 5)**:
- `bob`은 content_editor 역할을 가지고 있어도 `/news/*` 경로의 편집이 차단된다. 가장 높은 우선순위(가장 작은 숫자)를 가진 deny 정책이기 때문이다.

이런 "역할로 기본 권한 설정, 개인별로 예외 처리"하는 방식이 실용적이다. 대부분의 사용자는 역할에 따른 권한을 그대로 따르고, 특별한 케이스만 개별적으로 처리할 수 있기 때문이다.

---

## Spring Boot 통합: PostgreSQL 어댑터

정책을 파일로 관리하면 수정할 때마다 재배포해야 하는 문제가 있다. 데이터베이스에 정책을 저장하고 동적으로 로드하는 방식이 훨씬 효율적이다. Casbin은 JDBC 어댑터를 제공해서 PostgreSQL 같은 데이터베이스와 쉽게 연동할 수 있다.

```java
@Configuration
public class CasbinConfig {

    @Value("${POSTGRESQL_ADDR}")     private String url;
    @Value("${POSTGRESQL_USERNAME}") private String username;
    @Value("${POSTGRESQL_PASSWORD}") private String password;

    @Bean
    public Enforcer enforcer() throws Exception {
        var modelResource = new ClassPathResource("casbin/model.conf");

        // PostgreSQL DataSource 설정
        var ds = new org.postgresql.ds.PGSimpleDataSource();
        ds.setURL(url); 
        ds.setUser(username); 
        ds.setPassword(password);

        // JDBC Adapter 생성
        var adapter = new org.casbin.adapter.JDBCAdapter(ds);

        // Enforcer 생성
        var e = new org.casbin.jcasbin.main.Enforcer(modelResource.getPath(), adapter);

        // 예시: 도메인 스코프 역할 링크 추가
        // g, alice, content_editor, journalism
        e.addNamedGroupingPolicy("g", List.of("alice", "content_editor", "journalism"));

        // 예시: 개인 deny(ACL) 우선 정책 추가
        // p, 5, bob, /news/*, edit, journalism, deny
        e.addPolicy("5", "bob", "/news/*", "edit", "journalism", "deny");

        // 정책을 DB에 저장
        e.savePolicy();
        return e;
    }
}
```
JDBC Adapter는 자동으로 `casbin_rule` 테이블을 생성한다. 테이블 구조는 다음과 같다:

```sql
CREATE TABLE casbin_rule (
    id SERIAL PRIMARY KEY,
    ptype VARCHAR(100),  -- 'p' (정책) 또는 'g' (역할 링크)
    v0 VARCHAR(100),     -- priority (p) 또는 user (g)
    v1 VARCHAR(100),     -- sub (p) 또는 role (g)
    v2 VARCHAR(100),     -- obj (p) 또는 domain (g)
    v3 VARCHAR(100),     -- act (p) 또는 빈값 (g)
    v4 VARCHAR(100),     -- dom (p) 또는 빈값 (g)
    v5 VARCHAR(100)      -- eft (p) 또는 빈값 (g)
);
```

**실제 저장 예시:**

정책과 역할 링크가 다음과 같이 저장된다:

| id | ptype | v0 | v1 | v2 | v3 | v4 | v5 |
|----|-------|----|----|----|----|----|----|
| 1 | p | 5 | bob | /news/* | edit | journalism | deny |
| 2 | p | 100 | content_editor | /news/* | (create\|edit) | journalism | allow |
| 3 | p | 100 | viewer | /news/* | read | journalism | allow |
| 4 | g | alice | content_editor | journalism | | | |
| 5 | g | bob | viewer | journalism | | | |

**데이터 매핑 규칙:**

`model.conf`의 `policy_definition` 순서대로 v0~v5에 매핑된다:

```ini
; model.conf에서 정의한 순서
p = priority, sub, obj, act, dom, eft

; 실제 코드
e.addPolicy("5", "bob", "/news/*", "edit", "journalism", "deny");

; DB 저장 결과
ptype='p', v0='5', v1='bob', v2='/news/*', v3='edit', v4='journalism', v5='deny'
```

---

## 권한 검사

Enforcer를 설정했으면 실제 서비스 코드에서 권한 검사를 수행할 수 있다. 사용법은 매우 간단하다.

```java
if (!enforcer.enforce(username, "/news/123", "edit", "journalism")) {
    throw new PermissionDeniedException("권한이 없습니다.");
}
```

`enforce()` 메서드는 주체(username), 객체(/news/123), 행동(edit), 도메인(journalism)을 인자로 받아서 권한 여부를 boolean으로 반환한다. 내부적으로는 앞서 정의한 matcher 로직에 따라 정책을 평가하고 결과를 반환한다.

정책만 수정하면 재배포 없이 바로 반영된다는 게 큰 장점이다. 역할, 도메인, 개인 예외가 모두 한 번에 적용된다.

---

## 실제 동작 검증

이론적인 설명만으로는 실제로 어떻게 동작하는지 감이 잘 안 올 수 있다. 구체적인 시나리는 아래와 같다.

### 시나리오 1: 역할 기반 권한

**설정:**
```java
// alice에게 content_editor 역할 부여
e.addNamedGroupingPolicy("g", List.of("alice", "content_editor", "journalism"));

// content_editor는 /news/* 경로에 edit 가능
e.addPolicy("100", "content_editor", "/news/*", "edit", "journalism", "allow");
```

**테스트:**
```java
// 성공 - alice는 content_editor 역할로 edit 가능
enforcer.enforce("alice", "/news/123", "edit", "journalism");  // true

// 실패 - alice는 delete 권한 없음
enforcer.enforce("alice", "/news/123", "delete", "journalism");  // false

// 실패 - 다른 도메인에서는 권한 없음
enforcer.enforce("alice", "/news/123", "edit", "other-domain");  // false
```

---

### 시나리오 2: ACL 예외 

**설정:**
```java
// bob은 viewer 역할 (read만 가능)
e.addNamedGroupingPolicy("g", List.of("bob", "viewer", "journalism"));
e.addPolicy("100", "viewer", "/news/*", "read", "journalism", "allow");

// 하지만 bob에게 특정 경로 edit 허용 (priority 10 - 더 높은 우선순위)
e.addPolicy("10", "bob", "/news/special/*", "edit", "journalism", "allow");
```

**테스트:**
```java
// 성공 - viewer 역할로 read 가능
enforcer.enforce("bob", "/news/general", "read", "journalism");  // true

// 성공 - ACL 예외로 특정 경로 edit 허용
enforcer.enforce("bob", "/news/special/exclusive", "edit", "journalism");  // true

// 실패 - 일반 경로는 viewer 권한만 (edit 불가)
enforcer.enforce("bob", "/news/general", "edit", "journalism");  // false
```

---

### 시나리오 3: deny-override 

**설정:**
```java
// charlie에게 content_editor 역할 부여
e.addNamedGroupingPolicy("g", List.of("charlie", "content_editor", "journalism"));
e.addPolicy("100", "content_editor", "/news/*", "edit", "journalism", "allow");

// 하지만 charlie의 edit는 명시적으로 차단 (priority 5 - 가장 높은 우선순위)
e.addPolicy("5", "charlie", "/news/*", "edit", "journalism", "deny");
```

**테스트:**
```java
// 실패 - deny가 allow보다 우선 (priority 5 < 100)
enforcer.enforce("charlie", "/news/123", "edit", "journalism");  // false

// 성공 - read는 차단되지 않음
enforcer.enforce("charlie", "/news/123", "read", "journalism");  // true
```

**우선순위 동작 원리:**

1. matcher가 매칭되는 모든 정책을 찾음
2. priority가 가장 작은 정책(우선순위 높음)을 먼저 평가
3. 첫 번째 매칭의 eft(allow/deny)를 최종 결과로 반환

```
charlie의 /news/123 edit 요청:

매칭된 정책:
- priority 5:   charlie, /news/*, edit → deny   ← 이게 선택됨!
- priority 100: content_editor, /news/*, edit → allow

결과: false (deny)
```

---

## 마무리

도메인별 권한 분리, RBAC, ACL 예외 처리를 모두 지원해야 하는 요구사항을 마주했을 때, 어떻게 구현해야 할지 감이 안 잡혔다. Casbin이라는 라이브러리를 알게 됐는데, 처음 보는 기술이라 쉽지 않았다.

그래도 공식 문서랑 예제를 보면서 하나씩 이해하고 나니, 생각보다 구조가 명확했다. 복잡한 권한 로직이 model.conf 몇 줄로 선언적으로 표현되는 게 꽤 인상적이었다.

정책만 수정하면 재배포 없이 바로 반영되는 점도 좋았다. 처음 접하는 기술이라 러닝 커브가 있긴 했지만, 충분히 투자할 만한 가치가 있었던 것 같다.