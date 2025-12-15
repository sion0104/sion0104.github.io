---

title: "[iOS/HTTP] 400 Bad Request 오류가 발생하는 이유와 해결 과정 정리"
date: 2025-12-15
categories:
  - iOS
  - Network
  - HTTP
  - Swift
tags: [iOS, Swift, HTTP, "400", BadRequest, 네트워크, API]
slug: ios-http-400-bad-request-debugging
permalink: /posts/ios-http-400-bad-request-debugging/

---

# 400 Bad Request 오류가 발생하는 이유와 해결 과정 정리

API를 연동하다 보면 가장 자주 마주치는 오류 중 하나가 **400 Bad Request**이다.
이번 글에서는 iOS 앱에서 로그인 API를 연동하는 과정에서 실제로 겪었던 400 오류를 기준으로,
**오류가 발생한 원인, 해결 방법, 그리고 HTTP 이론**을 정리했습니다.

---

## 1. 400 Bad Request란 무엇인가

HTTP 상태 코드 **400 Bad Request**는 다음을 의미한다.

> **클라이언트가 보낸 요청이 서버의 요구 조건을 만족하지 못했다**

즉,

* 요청 문법이 잘못되었거나
* 필수 파라미터가 누락되었거나
* 서버가 기대하는 형식과 다른 방식으로 데이터를 전달했을 때
  서버는 400 응답을 반환한다.

중요한 점은 **서버 로직까지 도달하기 전에 요청 검증 단계에서 실패**했다는 것이다.

---

## 2. 실제 발생한 400 오류 상황

로그인 API 연동 중 다음과 같은 오류가 발생했다.

```http
POST /v1/auth/sign-in
→ 400 Bad Request
message: "must not be blank"
```

Swagger 문서에는 다음과 같이 명시되어 있었다.

```
POST /v1/auth/sign-in
username (query)
password (query)
```

하지만 iOS 앱에서는 다음과 같이 요청을 보내고 있었다.

```http
POST /v1/auth/sign-in
Content-Type: application/json

{
  "username": "user1111",
  "password": "a1234"
}
```

---

## 3. 왜 400 오류가 발생했는가 (이론적 원인)

### (1) 서버와 클라이언트의 요청 형식 불일치

서버는 다음 중 하나를 기대하고 있었다.

* query parameter
* application/x-www-form-urlencoded (form 방식)

하지만 클라이언트는 **JSON body**로 데이터를 전달했다.

서버 입장에서는:

```text
username == null
password == null
```

이 되었고, 서버의 유효성 검증 로직에서 다음과 같은 판단이 내려졌다.

```java
if (username == null || username.isBlank()) {
    throw ValidationException("must not be blank");
}
```

이 단계에서 요청은 **비즈니스 로직으로 진입하지 못하고 즉시 차단**되었으며,
그 결과 400 Bad Request가 반환되었다.

---

## 4. 400 오류의 핵심 특징

이번 사례를 통해 정리할 수 있는 400 오류의 특징은 다음과 같다.

* 요청은 서버에 도달했다
* 하지만 **요청 형식 자체가 서버 규칙을 위반**
* 서버는 데이터를 읽지 못하거나 검증 단계에서 실패
* 비즈니스 로직은 실행되지 않음

---

## 5. 해결 방법

### ✔ 핵심 해결 전략

> **서버가 기대하는 요청 형식에 맞춰 데이터를 전달한다**

이번 경우에는 JSON이 아니라
`application/x-www-form-urlencoded` 방식으로 요청을 보내야 했다.

---

### ✔ 수정 후 요청 예시

```http
POST /v1/auth/sign-in
Content-Type: application/x-www-form-urlencoded

username=user1111&password=a1234
```

Swift 코드에서는 다음과 같이 구현했다.

```swift
func postForm<T: Decodable>(
    _ path: String,
    form: [String: String]
) async throws -> T {

    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue(
        "application/x-www-form-urlencoded; charset=utf-8",
        forHTTPHeaderField: "Content-Type"
    )

    let body = form
        .map { "\($0.key)=\($0.value)" }
        .joined(separator: "&")

    request.httpBody = body.data(using: .utf8)

    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(T.self, from: data)
}
```

---

## 6. 400 오류를 예방하는 체크리스트

로그인·회원가입 API 연동 시 아래 항목을 먼저 확인하면 400 오류를 크게 줄일 수 있다.

1. 서버 문서에서 **query / form / body(JSON)** 중 어떤 방식을 요구하는지
2. Content-Type 헤더가 올바르게 설정되어 있는지
3. 필수 파라미터가 실제로 서버에 전달되고 있는지
4. 공백(trim) 처리로 인해 값이 비어 있지 않은지
5. Swagger Try it out과 동일한 방식으로 요청을 보내고 있는지

---

## 7. 정리

400 Bad Request는 단순한 “에러”가 아니라,
**서버와 클라이언트 간의 계약(API 스펙)이 어긋났다는 신호**이다.

이번 사례에서는:

* JSON body로 요청을 보냈지만
* 서버는 query/form 파라미터를 기대했고
* 그 결과 요청 검증 단계에서 400 오류가 발생했다.

> **400 오류를 만났다면 먼저 “요청 형식”부터 의심해야 한다.**

이 원칙만 기억해도 네트워크 디버깅 속도가 크게 빨라진다.
