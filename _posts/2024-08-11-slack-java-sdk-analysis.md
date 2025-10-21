---
title: Slack Java SDK 분석하기 - HTTP 클라이언트부터 API 호출까지
description: Slack Java SDK의 Status API를 분석하면서 OkHttp와 Gson을 활용한 HTTP 통신 패턴을 학습합니다.
date: 2024-08-28
categories: [Backend, Java, SDK]
tags: [Java, Slack SDK, OkHttp, Gson, HTTP, REST API]
pin: false
math: false
mermaid: false
---


오픈소스 컨트리뷰션 아카데미(OSSCA)에 참여하면서 LitmusChaos의 Java SDK 개발에 참여할 기회가 생겼다. 팀원 모두 SDK를 개발해본 경험이 없었기 때문에, 먼저 다른 오픈소스의 Java SDK를 분석하며 학습하기로 했다.

특히 Slack의 Status API 모듈이 비교적 단순하면서도 SDK의 핵심 패턴을 잘 보여줘서, 이를 중심으로 분석한 내용을 정리해봤다.

---

## Slack Java SDK란?

Slack Java SDK는 Slack의 공식 Java 라이브러리로, Web API, RTM API, Events API 등 다양한 Slack 기능을 제공한다.

이번에 살펴볼 Status API는 Slack 서비스의 상태를 조회하는 역할을 한다. 서비스가 정상 작동 중인지, 장애가 발생했는지 등의 정보를 가져올 수 있다.

**관련 링크**
- [Slack API Client 패키지](https://github.com/slackapi/java-slack-sdk/tree/main/slack-api-client)
- [Status API 문서](https://slack.dev/java-slack-sdk/guides/status-api)
- [Slack Status API 공식 문서](https://api.slack.com/apis/slack-status)

---

## 전체 구조 살펴보기

Status API의 구조는 생각보다 단순했다. 크게 세 가지 요소로 이루어져 있다:

| 클래스명 | 역할 |
|---------|------|
| `SlackHttpClient` | HTTP 요청을 처리하는 클라이언트 (OkHttp 기반) |
| `StatusClientImpl` | Status API 엔드포인트를 호출하는 구현체 |
| `CurrentStatus`, `SlackIssue` | API 응답을 담는 데이터 모델 |

`SlackHttpClient`가 HTTP 통신의 모든 책임을 지고, `StatusClientImpl`은 이를 활용해서 실제 API를 호출한다. 응답 데이터는 `CurrentStatus`나 `SlackIssue` 같은 모델 객체로 변환된다.

여기서 핵심 라이브러리가 두 가지 등장한다:
- **OkHttp**: HTTP 요청을 보내는 클라이언트
- **Gson**: JSON 데이터를 Java 객체로 변환

이 두 라이브러리를 먼저 이해하고 나면, SDK의 동작 방식이 훨씬 명확해진다.

---

## OkHttp 

### OkHttp란?

OkHttp는 Square에서 만든 HTTP 클라이언트 라이브러리다. Java 표준 라이브러리의 `HttpURLConnection`보다 훨씬 사용하기 편하고 기능도 많다.

**주요 특징:**
- **연결 재사용**: 같은 서버에 여러 요청을 보낼 때 연결을 재사용해서 성능을 높인다
- **인터셉터**: 요청/응답을 가로채서 로깅이나 헤더 추가 같은 작업을 할 수 있다
- **비동기 처리**: 별도 스레드에서 요청을 처리해서 메인 로직이 멈추지 않는다
- **HTTP/2 지원**: 최신 프로토콜로 빠른 통신이 가능하다

### 기본 사용법

```java
OkHttpClient client = new OkHttpClient();

Request request = new Request.Builder()
    .url("https://api.example.com/users")
    .build();

try (Response response = client.newCall(request).execute()) {
    if (response.isSuccessful()) {
        String body = response.body().string();
        System.out.println("응답: " + body);
    }
}
```

핵심은 이렇다:
1. `OkHttpClient` 인스턴스를 만든다 (한 번만 만들어서 재사용하는 게 좋다)
2. `Request`로 요청을 정의한다
3. `newCall().execute()`로 동기 요청을 보낸다
4. `response.body().string()`로 응답 본문을 읽는다

### POST 요청 보내기

데이터를 보내야 할 때는 `RequestBody`를 사용한다.

```java
String json = "{\"name\":\"홍길동\",\"age\":30}";
RequestBody body = RequestBody.create(
    json, 
    MediaType.parse("application/json")
);

Request request = new Request.Builder()
    .url("https://api.example.com/users")
    .post(body)
    .build();

try (Response response = client.newCall(request).execute()) {
    if (response.isSuccessful()) {
        System.out.println("성공!");
    }
}
```

JSON 문자열을 `RequestBody`로 감싸서 POST 요청으로 보내면 된다.

### 비동기 요청

`.execute()` 대신 `.enqueue()`를 쓰면 비동기로 요청을 보낼 수 있다.

```java
client.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {
        System.out.println("요청 실패: " + e.getMessage());
    }

    @Override
    public void onResponse(Call call, Response response) {
        if (response.isSuccessful()) {
            System.out.println("응답 받음!");
        }
    }
});

System.out.println("요청 보냄! (대기 중...)");
```

요청이 백그라운드에서 처리되기 때문에, "요청 보냄!"이 먼저 출력되고 나중에 응답을 받게 된다.

---

## Gson 

### Gson이란?

Gson은 Google에서 만든 JSON 라이브러리다. JSON 문자열과 Java 객체를 서로 변환해준다.

**주요 특징:**
- **간단한 API**: 몇 줄의 코드로 변환 가능
- **커스터마이징**: 필요하면 변환 규칙을 직접 정의할 수 있다
- **네이밍 전략**: camelCase, snake_case 등 다양한 규칙 지원

### 기본 사용법

**객체를 JSON으로 변환하기:**

```java
class User {
    String name;
    int age;
}

User user = new User();
user.name = "홍길동";
user.age = 30;

Gson gson = new Gson();
String json = gson.toJson(user);
// 결과: {"name":"홍길동","age":30}
```

**JSON을 객체로 변환하기:**

```java
String json = "{\"name\":\"홍길동\",\"age\":30}";

Gson gson = new Gson();
User user = gson.fromJson(json, User.class);

System.out.println(user.name); // 홍길동
System.out.println(user.age);  // 30
```

정말 간단하다. `toJson()`과 `fromJson()` 두 개만 기억하면 된다.

### 특정 필드 제외하기

민감한 정보를 JSON에 포함시키고 싶지 않을 때는 `transient` 키워드를 사용한다.

```java
class User {
    String name;
    transient String password; // JSON 변환에서 제외됨
}

User user = new User();
user.name = "홍길동";
user.password = "secret123";

Gson gson = new Gson();
String json = gson.toJson(user);
// 결과: {"name":"홍길동"}
```

`password` 필드는 JSON에 포함되지 않는다.

---

## OkHttp + Gson 조합하기

이제 이 둘을 합쳐보자. API 응답을 받아서 바로 객체로 변환하는 코드다.

```java
class Post {
    int userId;
    int id;
    String title;
    String body;
}

OkHttpClient client = new OkHttpClient();
Request request = new Request.Builder()
    .url("https://jsonplaceholder.typicode.com/posts/1")
    .build();

try (Response response = client.newCall(request).execute()) {
    if (response.isSuccessful()) {
        String json = response.body().string();
        
        Gson gson = new Gson();
        Post post = gson.fromJson(json, Post.class);
        
        System.out.println("제목: " + post.title);
        System.out.println("내용: " + post.body);
    }
}
```

OkHttp로 API를 호출하고, 받은 JSON을 Gson으로 객체에 매핑했다. 이 패턴이 Slack SDK 전반에 사용되는 핵심 구조다.

---

## Slack SDK에서는 어떻게 쓰일까?

이제 본격적으로 Slack SDK 코드를 살펴보자.

### SlackHttpClient 

`SlackHttpClient`는 Slack API와의 모든 HTTP 통신을 담당한다. 내부적으로 OkHttp를 사용하고, GET, POST, PATCH, DELETE 같은 다양한 요청을 지원한다.

**주요 메서드:**

| 메서드 | 역할 |
|-------|------|
| `get()` | GET 요청 (데이터 조회) |
| `postJsonBody()` | JSON 데이터를 담은 POST 요청 |
| `postMultipart()` | 파일 업로드용 POST 요청 |
| `postForm()` | 폼 데이터 POST 요청 |
| `delete()` | DELETE 요청 |
| `patchCamelCaseJsonBodyWithBearerHeader()` | PATCH 요청 (OAuth 토큰 포함) |
| `close()` | 리소스 해제 |

#### GET 요청 예시

```java
public Response get(String url, Map<String, String> query, String token) 
    throws IOException {
    
    // 쿼리 파라미터 추가
    if (query != null) {
        HttpUrl.Builder urlBuilder = HttpUrl.parse(url).newBuilder();
        for (Map.Entry<String, String> entry : query.entrySet()) {
            urlBuilder.addQueryParameter(entry.getKey(), entry.getValue());
        }
        url = urlBuilder.build().toString();
    }
    
    // 요청 생성
    Request.Builder rb = new Request.Builder()
        .url(url)
        .get();
    
    // OAuth 토큰이 있으면 추가
    if (token != null) {
        rb.header("Authorization", "Bearer " + token);
    }
    
    return okHttpClient.newCall(rb.build()).execute();
}
```

쿼리 파라미터를 URL에 붙이고, 필요하면 인증 토큰도 헤더에 추가한다. 기본적인 GET 요청의 전형적인 패턴이다.

#### POST 요청 (JSON)

```java
public Response postJsonBody(String url, Object obj) throws IOException {
    // 객체를 snake_case JSON으로 변환
    String json = toSnakeCaseJsonString(obj);
    RequestBody body = RequestBody.create(
        json, 
        MEDIA_TYPE_APPLICATION_JSON
    );
    
    Request request = new Request.Builder()
        .url(url)
        .post(body)
        .build();
    
    return okHttpClient.newCall(request).execute();
}
```

Java 객체를 받아서 JSON으로 변환하고 POST 요청으로 보낸다. 여기서 주목할 점은 `toSnakeCaseJsonString()` 메서드다. Slack API는 snake_case를 사용하기 때문에, Java의 camelCase 네이밍을 변환해준다.

#### 파일 업로드

```java
public Response postMultipart(String url, String token, 
                              MultipartBody multipartBody) 
    throws IOException {
    
    Request.Builder rb = new Request.Builder()
        .url(url)
        .post(multipartBody)
        .header("Authorization", "Bearer " + token);
    
    return okHttpClient.newCall(rb.build()).execute();
}
```

파일을 업로드할 때는 `multipart/form-data` 형식을 사용한다. OkHttp의 `MultipartBody`가 이를 처리해준다.

#### 리소스 정리

```java
@Override
public void close() throws Exception {
    // 스레드 풀 종료
    okHttpClient.dispatcher().executorService().shutdown();
    
    // 연결 풀 비우기
    okHttpClient.connectionPool().evictAll();
    
    // 캐시 닫기
    if (okHttpClient.cache() != null) {
        okHttpClient.cache().close();
    }
}
```

`SlackHttpClient`는 `AutoCloseable`을 구현해서 `try-with-resources`로 사용할 수 있다. 더 이상 사용하지 않을 때는 반드시 리소스를 정리해야 메모리 누수를 방지할 수 있다.

### StatusClientImpl - API 호출 구현

`StatusClientImpl`은 `SlackHttpClient`를 활용해서 실제 Status API를 호출한다.

```java
public class StatusClientImpl implements StatusClient {
    
    private String endpointUrlPrefix = "https://slack-status.com/api/v2.0.0/";
    private final SlackHttpClient slackHttpClient;

    public StatusClientImpl(SlackHttpClient slackHttpClient) {
        this.slackHttpClient = slackHttpClient;
    }

    @Override
    public CurrentStatus current() throws IOException, StatusApiException {
        // GET 요청 보내기
        Response response = slackHttpClient.get(
            endpointUrlPrefix + "current", 
            null, 
            null
        );
        
        String body = response.body().string();
        slackHttpClient.runHttpResponseListeners(response, body);
        
        // 응답 성공 시 JSON을 객체로 변환
        if (response.isSuccessful()) {
            return GsonFactory.createSnakeCase(slackHttpClient.getConfig())
                .fromJson(body, CurrentStatus.class);
        } else {
            throw new StatusApiException(response, body);
        }
    }

    @Override
    public List<SlackIssue> history() throws IOException, StatusApiException {
        Response response = slackHttpClient.get(
            endpointUrlPrefix + "history", 
            null, 
            null
        );
        
        // List 타입 정의
        Type listType = new TypeToken<ArrayList<SlackIssue>>() {}.getType();
        
        String body = response.body().string();
        slackHttpClient.runHttpResponseListeners(response, body);
        
        if (response.isSuccessful()) {
            return GsonFactory.createSnakeCase(slackHttpClient.getConfig())
                .fromJson(body, listType);
        } else {
            throw new StatusApiException(response, body);
        }
    }
}
```

동작 과정을 정리하면:

1. `SlackHttpClient`로 HTTP 요청을 보낸다
2. 응답 본문을 문자열로 읽는다
3. 응답 리스너를 실행한다 
4. 성공하면 JSON을 객체로 변환해서 반환한다
5. 실패하면 예외를 던진다

여기서 `GsonFactory.createSnakeCase()`를 사용하는 걸 볼 수 있다. Slack API가 snake_case를 사용하기 때문에, Gson에 네이밍 전략을 설정해서 자동으로 변환하도록 한 것이다.

이렇게 `SlackHttpClient`와 `StatusClientImpl`의 역할을 분리하니 코드가 훨씬 명확해진다. HTTP 통신은 `SlackHttpClient`가 전담하고, `StatusClientImpl`은 비즈니스 로직에만 집중할 수 있다. 이런 구조 덕분에 새로운 API를 추가할 때도 HTTP 클라이언트를 재사용할 수 있어서 효율적이다.


---

## 정리

Slack SDK를 분석하면서 HTTP 통신과 JSON 처리의 실전 패턴을 많이 배울 수 있었다.

**핵심 패턴:**
- SlackHttpClient가 HTTP 통신의 모든 책임을 담당하고, 다양한 요청 타입을 지원한다
- OkHttp와 Gson을 조합해서 API 호출부터 객체 매핑까지 처리한다
- snake_case와 camelCase 변환을 Gson의 네이밍 전략으로 자동화한다
- AutoCloseable을 구현해서 리소스 관리를 명확하게 한다

SDK를 만들 때는 이런 패턴을 참고하면 좋을 것 같다. 다음에는 다른 API 모듈도 분석해보면서 더 복잡한 패턴들을 살펴봐야겠다.

---

## 참고 자료

- [Slack Java SDK GitHub](https://github.com/slackapi/java-slack-sdk)
- [OkHttp 공식 문서](https://square.github.io/okhttp/)
- [Gson 공식 문서](https://github.com/google/gson)
- [Slack Status API 문서](https://api.slack.com/apis/slack-status)