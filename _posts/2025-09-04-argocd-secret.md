---
title: ArgoCD 오픈소스 기여 - 웹훅 시크릿 참조가 사라지던 문제를 해결하기
description: ArgoCD 서버 재시작 시 웹훅 시크릿 참조가 실제 값으로 변환되어 External Secret 로테이션이 반영되지 않던 문제의 원인 분석과 해결 과정을 다룹니다.
date: 2025-09-04
categories: [DevOps, ArgoCD]
tags: [ArgoCD, Webhook, External Secret, Kubernetes, Bug Fix]
pin: false
math: false
mermaid: false
---

## 문제 발견

ArgoCD에 기여하고 싶어서 이슈를 탐색하던 중, 흥미로운 버그 리포트를 발견했다. External Secret을 사용하는 환경에서 **서버 파드가 재시작될 때마다 웹훅 시크릿 참조가 실제 값으로 바뀌는** 문제였다.

이슈를 읽어보니 꽤 심각한 문제였다. 한 번 참조가 실제 값으로 바뀌고 나면, 그 이후로는 시크릿 로테이션이 전혀 반영되지 않는다는 것이다. 보안상 시크릿은 주기적으로 로테이션되어야 하는데, 이 버그 때문에 그게 불가능했다.

예를 들어 이런 식이었다:

```yaml
# 서버 재시작 전 (정상 상태)
webhook.github.secret: $github-webhook-secret:token

# 서버 재시작 후 (문제 발생)
webhook.github.secret: actual-secret-value-123
```

원래는 `$github-webhook-secret:token` 형태로 External Secret을 참조하도록 설정했는데, 서버가 재시작되면 실제 시크릿 값인 `actual-secret-value-123`으로 바뀌어 버렸다. 그러면 External Secret에서 값을 업데이트해도, ArgoCD는 이미 고정된 값만 계속 사용하게 된다.

보안상 시크릿은 주기적으로 로테이션되어야 하는데, 이 문제 때문에 로테이션이 전혀 먹히지 않았다. 수동으로 ArgoCD 파드를 재시작하면 새 값이 반영되긴 했지만, 그러면 또 다시 참조가 실제 값으로 바뀌는 악순환이 반복됐다.

---

## 원인 분석

### 코드 읽기 시작

문제의 원인을 찾기 위해 ArgoCD 소스코드를 뜯어보기 시작했다. 웹훅 시크릿 관련 설정을 처리하는 부분은 [util/settings/settings.go](https://github.com/argoproj/argo-cd/blob/master/util/settings/settings.go) 파일에 있었다.

가장 먼저 눈에 들어온 건 `updateSettingsFromSecret()` 함수였다. 이 함수는 Kubernetes Secret에서 설정값을 읽어와서 ArgoCD 설정 구조체에 저장하는 역할을 한다.

```go
// 문제의 코드
settings.WebhookGitHubSecret = ReplaceStringSecret(
    string(argoCDSecret.Data[settingsWebhookGitHubSecretKey]),
    settings.Secrets,
)
```

여기서 문제를 발견했다. `ReplaceStringSecret` 함수는 `$secret-name:key` 형태의 참조를 실제 시크릿 값으로 치환하는 함수다. 그런데 이 함수를 **설정을 읽어오는 시점**에 바로 호출하고 있었다.

즉, Kubernetes Secret에 저장된 `$github-webhook-secret:token`이라는 참조를 읽어오자마자 실제 값으로 바꿔버리고, 그 바뀐 값을 다시 메모리에 저장한다. 그리고 설정이 업데이트되면 그 값이 다시 Kubernetes Secret에 저장되는 구조였다.

결과적으로 한 번 서버가 재시작되면:
1. 참조 형태(`$github-webhook-secret:token`)를 읽어온다
2. 즉시 실제 값(`actual-secret-value-123`)으로 치환한다
3. 치환된 값을 메모리에 저장한다
4. 설정이 업데이트될 때 치환된 값이 Secret에 다시 저장된다
5. 원본 참조는 영원히 사라진다

### 다른 설정들은 어떨까?

흥미로운 점은, OIDC나 DEX 같은 다른 설정들은 이런 문제가 없다는 것이었다. 그래서 해당 코드를 확인해봤다.

```go
// OIDC - 정상 동작하는 코드
func (a *ArgoCDSettings) OIDCConfig() *OIDCConfig {
    config := a.oidcConfig()
    return config.toExported()
}

func (a *ArgoCDSettings) oidcConfig() *oidcConfig {
    configMap := map[string]any{}
    yaml.Unmarshal([]byte(a.OIDCConfigRAW), &configMap)

    configMap = ReplaceMapSecrets(configMap, a.Secrets)
    // ... 이후 처리
}
```

OIDC 설정을 보니 구조가 완전히 달랐다. 여기서는:
1. 원본 설정을 그대로 `OIDCConfigRAW`에 저장한다
2. 설정을 **사용할 때** `oidcConfig()` 함수를 호출한다
3. 그 함수 안에서 `ReplaceMapSecrets`로 시크릿을 치환한다
4. 치환된 값은 메모리에만 있고, 원본은 그대로 유지된다

즉, **원본 보존 + 사용 시점 치환**이라는 패턴을 사용하고 있었다.

정리하면 차이점은 이렇다:
- **OIDC/DEX**: 원본은 그대로 두고, 사용할 때만 치환
- **Webhook**: 읽을 때 바로 치환해서 원본 손실

웹훅 설정도 같은 패턴으로 바꾸면 문제를 해결할 수 있을 것 같았다.

---

## 해결 방향

### 패턴 통일

결론은 간단했다. 웹훅 설정도 OIDC처럼 **원본을 보존하고, 실제 사용할 때만 시크릿을 치환**하도록 바꾸는 것이다.

구체적인 방향은 다음과 같았다:

1. **설정 읽기 단계**: Secret에서 읽을 때는 참조 형태 그대로 저장한다.
2. **설정 사용 단계**: 실제로 웹훅 검증할 때만 시크릿을 치환한다.
3. **접근 제어**: Getter 메서드를 만들어서 직접 필드에 접근하지 못하게 한다.

### 구현 방법 선택

처음에는 단순하게 사용하는 곳에서 `ReplaceStringSecret`를 직접 호출하는 방법을 생각했다. 하지만 OIDC처럼 Getter 메서드를 사용하는 패턴이 이미 있었고, 이 방식이 더 깔끔해 보였다.

Getter 메서드의 장점은:
- 필드에 직접 접근하는 걸 막을 수 있다
- 시크릿 치환 로직을 한 곳에 모을 수 있다
- 나중에 로직을 수정하기 쉽다

그래서 Getter 패턴을 따르기로 결정했다.

---

## 수정 내용

### 1. 설정 읽기 로직 변경

먼저 설정을 읽어오는 부분을 수정했다. `ReplaceStringSecret` 호출을 제거하고, 읽어온 값을 그대로 저장하도록 바꿨다.

```go
// 기존 코드 (문제가 있던 코드)
settings.WebhookGitHubSecret = ReplaceStringSecret(
    string(argoCDSecret.Data[settingsWebhookGitHubSecretKey]),
    settings.Secrets,
)

// 수정 후 (원본을 그대로 저장)
settings.WebhookGitHubSecret = string(argoCDSecret.Data[settingsWebhookGitHubSecretKey])
```

이제 `$github-webhook-secret:token` 같은 참조 형태가 그대로 `WebhookGitHubSecret` 필드에 저장된다. 서버가 재시작되어도 참조가 유지되는 것이다.

### 2. Getter 메서드 추가

다음으로 시크릿 값을 가져오는 Getter 메서드를 추가했다. 이 메서드가 호출될 때만 시크릿 치환이 일어난다.

```go
func (a *ArgoCDSettings) GetWebhookGitHubSecret() string {
    return ReplaceStringSecret(a.WebhookGitHubSecret, a.Secrets)
}
```

이 함수는:
1. 원본 필드(`WebhookGitHubSecret`)의 값을 읽는다
2. `ReplaceStringSecret`로 참조를 실제 값으로 치환한다
3. 치환된 값을 반환한다
4. 원본 필드는 변경되지 않는다

GitLab, Bitbucket 등 다른 웹훅도 동일한 패턴으로 Getter를 추가했다:

```go
func (a *ArgoCDSettings) GetWebhookGitLabSecret() string {
    return ReplaceStringSecret(a.WebhookGitLabSecret, a.Secrets)
}

func (a *ArgoCDSettings) GetWebhookBitbucketSecret() string {
    return ReplaceStringSecret(a.WebhookBitbucketSecret, a.Secrets)
}

func (a *ArgoCDSettings) GetWebhookBitbucketUUIDSecret() string {
    return ReplaceStringSecret(a.WebhookBitbucketUUIDSecret, a.Secrets)
}

func (a *ArgoCDSettings) GetWebhookGogsSecret() string {
    return ReplaceStringSecret(a.WebhookGogsSecret, a.Secrets)
}
```

### 3. 사용처 수정

이제 웹훅 시크릿을 사용하는 모든 곳에서 직접 필드에 접근하지 않고 Getter를 사용하도록 바꿨다.

```go
// 기존 코드 (필드 직접 접근)
githubHandler, err := github.New(github.Options.Secret(settings.WebhookGitHubSecret))

// 수정 후 (Getter 사용)
githubHandler, err := github.New(github.Options.Secret(settings.GetWebhookGitHubSecret()))
```

이런 식으로 `server.go`, `webhook.go`, `applicationset/webhook/webhook.go` 등 여러 파일에서 사용하는 부분을 모두 수정했다.

### 4. 서버 재시작 감지 로직 보완

서버에는 웹훅 시크릿이 변경되면 자동으로 재시작하는 로직이 있다. 이 부분도 문제가 있었다.

```go
// 기존 코드
prevGitHubSecret := server.settings.WebhookGitHubSecret
// ... 이후에 설정 리로드 ...
if prevGitHubSecret != server.settings.WebhookGitHubSecret {
    log.Infof("github secret modified. restarting")
}
```

이 코드는 참조 형태와 실제 값을 비교하는 문제가 있었다. 예를 들어:
- 이전 값: `$github-webhook-secret:token` (참조)
- 새 값: `actual-secret-value-123` (치환된 실제 값)

이렇게 비교하면 값이 달라 보이지만, 실제로는 같은 시크릿을 가리키는 것일 수 있다. 반대로 참조는 같지만 실제 값이 바뀐 경우를 놓칠 수도 있다.

그래서 비교할 때도 Getter를 사용하도록 수정했다:

```go
// 수정 후
prevGitHubSecret := server.settings.GetWebhookGitHubSecret()
// ... 이후에 설정 리로드 ...
if prevGitHubSecret != server.settings.GetWebhookGitHubSecret() {
    log.Infof("github secret modified. restarting")
}
```

이제 실제 시크릿 값끼리 비교하기 때문에, 정확하게 변경 여부를 감지할 수 있다.

---
## 검증 과정

코드를 수정한 후에는 기존 테스트가 모두 통과하는지 확인했다. 다행히 모든 테스트가 정상적으로 통과했고, 시나리오를 직접 테스트해봤다.

**참조 형태 보존 확인**

서버 재시작 후에도 `$secret:key` 형태가 유지되는지 확인했다.

```bash
# 참조 형태로 설정
kubectl patch secret argocd-secret -p '{"data":{"webhook.github.secret":"JGdpdGh1Yi13ZWJob29rLXNlY3JldDp0b2tlbg=="}}'

# 서버 재시작
kubectl rollout restart deployment argocd-server

# 재시작 후 확인
kubectl get secret argocd-secret -o jsonpath='{.data.webhook\.github\.secret}' | base64 -d
# 결과: $github-webhook-secret:token 
```

예전에는 재시작 후 실제 값으로 바뀌었지만, 이제는 참조 형태가 그대로 유지됐다.

----


## 배운 점

이 문제를 해결하면서 몇 가지 중요한 교훈을 얻었다.

### 1. 일관성의 중요성

같은 기능(시크릿 참조 처리)인데도 OIDC는 "사용 시점 치환", 웹훅은 "읽기 시점 치환" 방식을 쓰고 있었다. 이런 불일치가 예상치 못한 버그를 만들어냈다.

코드베이스가 크면 클수록 일관된 패턴을 유지하는 게 중요하다는 걸 다시 한 번 느꼈다. 새로운 코드를 추가할 때는 기존 코드가 어떤 패턴을 사용하는지 반드시 확인해야 한다.

### 2. 지연 평가의 유용함

"필요할 때 계산한다"는 Lazy Evaluation 개념이 여기서도 적용됐다. 시크릿 치환을 설정 로드 시점이 아니라 실제 사용 시점까지 미루면:

- 원본 값이 보존된다
- 외부 의존성 변경이 즉시 반영된다
- 재시작 없이도 새 값을 사용할 수 있다

특히 외부 시스템과 연동되는 값일수록 지연 평가가 안정적이다.

### 3. 코드 패턴 분석의 중요성

문제를 해결하기 전에 "왜 OIDC는 괜찮은데 웹훅만 문제일까?"라는 질문부터 시작했다. 이미 잘 동작하는 코드의 패턴을 분석하니, 해결 방법이 명확해졌다.

---
## PR 정보

이 수정사항은 [ArgoCD 공식 저장소](https://github.com/argoproj/argo-cd)에 PR을 제출하여 머지되었다.

- **Issue**: [#22588](https://github.com/argoproj/argo-cd/issues/22588)
- **PR**: [fix(server): preserve webhook secret references on server restart](https://github.com/argoproj/argo-cd/pull/23905) 

---

## 마무리

지금까지 여러 번 오픈 소스 기여를 해봤지만, 이번이 가장 의미 있는 기여였던 것 같다. PR을 올리고 나서 생각보다 빠르게 승인을 받았다. 별다른 수정 사항 없이 한 번에 머지가 되어서 더 뿌듯했다. 코드베이스를 직접 뜯어보고, 문제의 원인을 찾아내고, 기존 패턴을 분석해서 해결책을 만들어내는 과정이 정말 재미있었다.
