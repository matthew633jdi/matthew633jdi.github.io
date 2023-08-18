---
title: "[Spring] 단일 URL에 다양한 값 받기"
date: 2023-01-12
categories: Spring
---

# [Spring] 단일 URL에서 여러 모델 처리

Spring REST Docs 작업을 진행 중 Spring Security의 Acess Token에 대한 API 명세를 작성 중 겪은 이슈

**POST [baseURL]/token**에서 처리하는 API

- Authorization Code Grant
- Client Credentials Grant
- Resource Owner Password Credentials Grant

각각의 방식에는 각각의 입력 파라미터가 존재한다.

3가지 방식에 대해서 아래와 같이 Controllerd의 3가지 메서드를 만들었다.

```java
@RestController
public class AuthorizationController {

  @PostMapping(value="/token", consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
  public void authorize(@RequestBody AuthReq authReq) {
    //
  }

  @PostMapping(value="/token", consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
  public void clientAuthorize(@RequestBody ClientAuthReq clientAuthReq) {
    //
  }

  @PostMapping(value="/token", consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
  public void authorize(@RequestBody PwdAuthReq pwdAuthReq) {
    //
  }
}
```

각 메서드에 대한 Test Case를 실행했을 때, 아래와 같은 오류가 발생하였다.

> java.lang.IllegalStateException: Ambiguous mapping found. Cannot map ‘AuthorizationController ’ bean method

DispatcherServlet이 요청으로 들어온 URL을 보고 Controller의 메서드를 실행하는데, 동일한 URL이 있어 구분할 수 없어서 발생시킨 오류이다.

Spring Security에서는 분명히 /token에서 각각의 방식을 모두 처리하고 있는데, 어떻게 처리해야할까???

spring-security-oauth2 라이브러리의 TokenEndpoint.class를 참고하였다.

```java
...
@RequestMapping(value = "/oauth/token", method=RequestMethod.POST)
public ResponseEntity<OAuth2AccessToken> postAccessToken(Principal principal, @RequestParam
	Map<String, String> parameters) throws HttpRequestMethodNotSupportedException {
        //
}
...

```

위 코드에서 /token에서 @RequestParam Map으로 여러 값을 받은 후 Map에 있는 값을 통해 분기하여 각각을 처리하고 있었다.

이를 통해 위 문제를 해결하였다.

```java
@PostMapping(value = "/token", consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
public ResponseEntity<byte[]> token(@RequestParam Map<String, String> parameters) throws IOException {
            // 분기 로직

            }
```