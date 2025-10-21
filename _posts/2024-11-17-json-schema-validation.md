---
title: Argument Resolver로 JSON Schema 검증하기
description: Spring MVC에서 Argument Resolver를 활용하여 JSON Schema 검증을 효율적으로 구현하는 방법을 소개합니다
date: 2024-11-17
categories: [Backend, Spring, 데이터 검증]
tags: [Spring MVC, JSON Schema, Argument Resolver, 데이터 검증, Java]
pin: false
math: false
mermaid: false
---
## JSON Schema 도입 이유
SpringBoot로 RESTful API를 개발하다 보면 클라이언트한테 받는 데이터를 검증할 때 보통 `@Valid` 어노테이션이랑 Bean Validation을 많이 쓴다. 간단한 검증에는 충분하지만, 복잡한 구조와 필드 간 관계를 검증하기엔 한계가 있다. 

예를 들어 "A 필드가 있을 때만 B 필드가 필수"라거나, "특정 조건에서만 C 필드 형식을 체크"하는 조건부 검증은 Bean Validation만으로 구현하기 까다롭다. 물론 커스텀 Validator를 만들면 되긴 하는데, 검증 로직이 복잡해질수록 관리가 어려워진다.

JSON Schema를 사용하면 이런 복잡한 검증 규칙을 선언적으로 깔끔하게 정의할 수 있다. 그래서 JSON Schema를 활용해 데이터를 검증하고, 이걸 Argument Resolver로 구현하는 방법을 정리해봤다.

---

## JSON Schema란?

JSON Schema는 JSON 데이터의 구조를 명확하게 정의하는 표준이다. 데이터의 청사진 같은 역할을 하며, 각 속성의 타입과 제약 조건을 상세하게 지정할 수 있다.

예를 들어, 게시판 데이터를 검증하는 스키마는 다음과 같이 작성할 수 있다:


```json
{
  "type": "object",
  "properties": {
    "boardTitle": {
      "type": "string",
      "minLength": 3,
      "maxLength": 70,
      "description": "게시글 제목"
    },
    "boardContent": {
      "type": "string",
      "minLength": 1,
      "description": "게시글 내용"
    },
    "boardGroupId": {
      "type": "string",
      "pattern": "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$",
      "description": "게시판 그룹 ID (UUID 형식)"
    },
    "tags": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "maxItems": 10,
      "description": "게시글 태그 목록"
    }
  },
  "required": ["boardTitle", "boardContent", "boardGroupId"],
  "additionalProperties": false
}
```

이 스키마는:
- 제목은 3~70자 사이여야 한다.
- 내용은 최소 1자 이상이어야 한다.
- 그룹 ID는 UUID 형식이어야 한다.
- 태그는 선택 사항이지만, 있다면 최대 10개까지만 허용한다/
- 정의되지 않은 추가 필드는 허용하지 않는다.

JSON Schema를 사용하면 API 요청에서 발생할 수 있는 다양한 오류를 사전에 방지할 수 있다. 타입 오류, 길이 제한, 필수 필드 누락 등을 명확하게 검증할 수 있기 때문이다.

---

## Argument Resolver란?

**Argument Resolver는 Spring MVC에서 컨트롤러 메서드의 파라미터를 자동으로 바인딩해주는 컴포넌트다.**

Spring의 `@RequestBody`, `@PathVariable`, `@RequestParam` 같은 어노테이션들도 내부적으로는 모두 Argument Resolver로 구현되어 있다. 각각 `RequestResponseBodyMethodProcessor`, `PathVariableMethodArgumentResolver`, `RequestParamMethodArgumentResolver`가 담당한다.

동작 원리는 다음과 같다.

1. **요청이 들어오면** Spring이 해당 컨트롤러 메서드를 찾는다
2. **메서드의 각 파라미터를 순회하며** 어떤 Resolver가 처리할 수 있는지 `supportsParameter()`로 확인한다
3. **적절한 Resolver를 찾으면** 그 Resolver의 `resolveArgument()`를 호출해서 실제 값을 생성한다
4. **생성된 값을** 메서드 파라미터로 전달해서 컨트롤러를 실행한다

직접 `HandlerMethodArgumentResolver`를 구현하면, 이 흐름에 커스텀 로직을 끼워넣을 수 있다. 파라미터를 받기 전에 검증, 변환, 가공 등 원하는 처리를 추가할 수 있는 것이다.

---

## 왜 Argument Resolver가 필요할까?

만약 JSON Schema를 컨트롤러에서 직접 검증하면 다음과 같은 코드가 반복된다.

```java
@PostMapping("/boards")
public ResponseDto createBoard(@RequestBody String jsonBody) {
    // JSON 파싱
    JsonNode jsonNode = objectMapper.readTree(jsonBody);
    
    // 스키마 로드
    JsonSchema schema = getJsonSchema("BoardModel.json");
    
    // 검증
    ProcessingReport report = schema.validate(jsonNode);
    if (!report.isSuccess()) {
        throw new ValidationException("Invalid data");
    }
    
    // 객체 변환
    BoardRequest request = objectMapper.treeToValue(jsonNode, BoardRequest.class);
    
    // 실제 비즈니스 로직
    // ...
}
```

이렇게 하면 모든 엔드포인트마다 같은 코드를 반복해야 하고, 검증 로직을 수정하려면 여러 파일을 고쳐야 한다. 컨트롤러가 검증까지 책임지면서 관심사가 섞이는 것도 문제다.

Argument Resolver를 사용하면 이런 문제를 해결할 수 있다. 검증 로직을 한 곳에 모아두고, 컨트롤러는 이미 검증된 데이터만 받을 수 있다. 코드도 훨씬 깔끔해진다.

---

## 구현 과정

### 1. JsonRequestBody 어노테이션 정의

먼저 JSON Schema의 경로를 지정하기 위한 커스텀 어노테이션을 만들었다.

```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface JsonRequestBody {
    String value();
}
```

각 어노테이션 메타데이터의 의미는 다음과 같다:

- `@Target(ElementType.PARAMETER)`: 이 어노테이션은 메서드의 파라미터에만 사용할 수 있다. 클래스나 메서드에는 붙일 수 없다.
- `@Retention(RetentionPolicy.RUNTIME)`: 런타임까지 어노테이션 정보가 유지된다. 이래야 실행 중에 리플렉션으로 어노테이션 정보를 읽을 수 있다.
- `String value()`: 스키마 파일의 경로를 지정하는 속성이다. 예를 들어 `@JsonRequestBody("BoardModel.json")`처럼 사용할 수 있다.

### 2. CustomJsonSchemaResolver 구현

핵심이 되는 Argument Resolver를 구현했다. Spring MVC에서 컨트롤러의 메서드 파라미터를 커스텀 방식으로 처리하려면 `HandlerMethodArgumentResolver` 인터페이스를 구현해야 한다.

```java
public class CustomJsonSchemaResolver implements HandlerMethodArgumentResolver {

    private final ObjectMapper objectMapper;
    private final ResourcePatternResolver resourcePatternResolver;
    private final JsonSchemaFactory schemaFactory;

    public CustomJsonSchemaResolver(ObjectMapper objectMapper, ResourcePatternResolver resourcePatternResolver) {
        this.objectMapper = objectMapper;
        this.resourcePatternResolver = resourcePatternResolver;
        this.schemaFactory = JsonSchemaFactory.byDefault();
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(JsonRequestBody.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        
        String schemaPath = parameter.getParameterAnnotation(JsonRequestBody.class).value();
        JsonSchema schema = getJsonSchema(schemaPath);
        JsonNode jsonNode = objectMapper.readTree(getJsonPayload(webRequest));
        ProcessingReport report = schema.validate(jsonNode);

        if (report.isSuccess()) {
            return objectMapper.treeToValue(jsonNode, parameter.getParameterType());
        }

        throw new CustomValidationException("JSON validation failed.");
    }

    private String getJsonPayload(NativeWebRequest webRequest) throws IOException {
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        
        if (request == null) {
            throw new CustomValidationException("Invalid request.");
        }

        return StreamUtils.copyToString(request.getInputStream(), StandardCharsets.UTF_8);
    }

    private JsonSchema getJsonSchema(String schemaPath) throws IOException {
        Resource resource = resourcePatternResolver.getResource("classpath:schemas/" + schemaPath);

        if (!resource.exists()) {
            throw new CustomValidationException("Schema not found: " + schemaPath);
        }

        try (InputStream stream = resource.getInputStream()) {
            return schemaFactory.getJsonSchema(
                JsonLoader.fromReader(new InputStreamReader(stream, StandardCharsets.UTF_8))
            );
        }
    }
}
```

각 메서드의 역할을 살펴보면:

**supportsParameter**

Spring이 "이 Resolver가 해당 파라미터를 처리할 수 있는가?"를 물어볼 때 호출된다. `@JsonRequestBody` 어노테이션이 붙어 있는 파라미터만 처리한다.

**resolveArgument**

실제 변환 작업이 일어나는 메서드다. 어노테이션에서 스키마 경로를 읽고, 스키마 파일을 로드한 다음, 요청 본문의 JSON을 검증한다. 검증이 성공하면 파라미터 타입에 맞는 객체로 변환해서 반환한다.

**getJsonPayload**

HTTP 요청의 본문을 읽어서 문자열로 반환한다. InputStream은 한 번만 읽을 수 있는데, Argument Resolver에서 한 번만 읽고 변환까지 완료하므로 문제없다.

**getJsonSchema**

지정된 경로에서 JSON 스키마 파일을 로드한다. `classpath:schemas/` 디렉토리를 기준으로 파일을 찾는다. 스키마 파일을 매번 읽는 게 비효율적일 수 있어서, 실제 운영에서는 캐싱을 추가하는 것도 좋다.

### 3. Argument Resolver 등록

만든 Resolver를 Spring에 등록해야 실제로 동작한다. `WebMvcConfigurer`를 구현해서 `addArgumentResolvers` 메서드를 오버라이드한다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private ObjectMapper objectMapper;
    // JSON 직렬화/역직렬화를 위한 ObjectMapper

    @Autowired
    private ResourcePatternResolver resourcePatternResolver;
    // 리소스 파일을 읽기 위한 Resolver

    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        // 우리가 만든 CustomJsonSchemaResolver를 등록
        resolvers.add(new CustomJsonSchemaResolver(objectMapper, resourcePatternResolver));
    }
}
```

`@Configuration` 어노테이션으로 이 클래스를 설정 클래스로 만들고, `WebMvcConfigurer`를 구현해서 Spring MVC 설정을 커스터마이즈한다. 

`addArgumentResolvers` 메서드에 우리가 만든 Resolver를 추가하면, Spring이 컨트롤러 메서드를 호출할 때 자동으로 이 Resolver를 사용하게 된다.

### 4. 컨트롤러에서 사용하기

이제 컨트롤러에서는 아주 간단하게 사용할 수 있다.

```java
@PostMapping()
public ResponseDto<BoardResponse> addBoard(
    @JsonRequestBody("BoardModel.json") BoardRequest request,
    @Principal String adminId) {
    
    BoardResponse boardResponse = boardService.addBoard(request, adminId);
    return ResponseDto.okWithData(boardResponse);
}
```

이전에 비교했던 복잡한 검증 코드가 완전히 사라졌다. `@JsonRequestBody` 어노테이션 하나만 붙이면:

1. 요청 본문이 자동으로 파싱된다
2. `BoardModel.json` 스키마로 검증된다
3. 검증이 성공하면 `BoardRequest` 객체로 변환된다
4. 컨트롤러는 이미 검증된 객체를 받는다

검증이 실패하면 컨트롤러 메서드가 실행되기 전에 예외가 발생한다. 이 예외는 `@ControllerAdvice`로 전역적으로 처리할 수 있다.

---


## 마무리

Spring MVC의 Argument Resolver를 활용해서 JSON Schema 기반의 요청 데이터 검증을 구현해봤다.

복잡한 검증 규칙이 많은 프로젝트라면 JSON Schema를 고려해볼 만하다. 특히 조건부 검증이나 동적 JSON 구조를 다뤄야 할 때 유용하다. 검증 로직을 DTO에서 분리할 수 있다는 점도 괜찮았다.

다만 JSON Schema가 익숙하지 않다면 오히려 러닝 커브가 생길 수 있다. 간단한 검증만 필요한 경우에는 @Valid + Bean Validation이 훨씬 직관적이다. 상황에 맞게 선택하면 될 것 같다.

---

## 참고

- [JSON Schema 공식 문서](https://json-schema.org/)
- [Spring ArgumentResolver 가이드](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html)
