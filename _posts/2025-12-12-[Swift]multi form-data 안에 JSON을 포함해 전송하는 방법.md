---
title: "[iOS/HTTP] multipart/form-data 안에 JSON을 포함해 전송하는 방법 정리"
date: 2025-12-12
categories:
  - iOS
  - Network
  - HTTP
  - Swift
tags: [iOS, Swift, HTTP, multipart, form-data, JSON, 네트워크, 파일업로드]
---

# multipart/form-data 안에 JSON을 포함해 전송하는 방법 정리

HTTP 요청에서 파일과 JSON 데이터를 함께 보내야 하는 경우가 많다. 대표적으로 회원가입 시 프로필 이미지와 사용자 정보를 동시에 보내는 상황이 있다. 이때 가장 흔히 사용되는 방식이 multipart/form-data이다. 이 글에서는 multipart와 form-data가 무엇인지, 어떤 상황에서 사용하는지, 그리고 multipart 안에 JSON을 안전하게 넣는 규칙을 설명한다. 마지막에는 Swift에서 직접 구현하는 예시도 포함한다.

---

## 1. multipart란 무엇인가

HTTP 요청 본문에는 데이터를 여러 방식으로 포함할 수 있다. 그중 multipart는 본문을 여러 조각(part)으로 나누어 전송하는 방식이다. 하나의 HTTP 요청 안에 여러 개의 데이터 조각을 전달할 수 있으며, 각 조각은 고유한 헤더와 본문을 가진다.

대표적인 예시는 다음과 같다.

* 사진 파일과 텍스트를 함께 전송할 때
* 여러 개의 파일을 한 번에 업로드할 때
* JSON 데이터와 바이너리를 함께 보내야 할 때

multipart는 이메일 프로토콜(MIME)에서 기원했으며 HTTP에서도 같은 구조를 사용한다.

---

## 2. form-data란 무엇인가

multipart는 여러 부분으로 이루어진 데이터를 전달하는 형식이고, 그중 form-data는 HTML 폼을 통해 전송되는 데이터를 표현하는 서브타입이다.

즉, Content-Type이 다음과 같을 때:

```
Content-Type: multipart/form-data; boundary=----ABC123
```

이 값은 서버가 "multipart 형식이며, 각 조각은 form-data 구조로 되어 있다"는 사실을 알려준다.
HTML `form` 태그에서 파일 업로드를 할 때 `enctype="multipart/form-data"`를 설정하는 이유도 여기에 있다.

---

## 3. multipart/form-data를 사용하는 이유

일반적인 JSON 요청으로는 파일 전송이 불가능하다. 파일은 바이너리 데이터이기 때문에 JSON 문자열로 변환하기 어렵고, 변환하더라도 용량 증가와 성능 저하 문제가 발생한다.

따라서 다음 상황에서는 multipart/form-data가 가장 적합하다.

* 파일과 텍스트를 동시에 전송해야 할 때
* JSON 데이터와 이미지 데이터를 한 요청에서 처리해야 할 때
* 클라이언트가 서버에 프로필 사진, 첨부 파일 등을 업로드할 때

---

## 4. multipart 안에 JSON을 넣을 때 반드시 지켜야 하는 규칙

multipart 안에 JSON을 포함하려면 아래 규칙들을 반드시 지켜야 한다. 이는 RFC 7578 표준에 기반한 규칙이며, 하나라도 틀리면 서버가 JSON을 읽지 못하거나 파일을 감지하지 못한다.

### (1) JSON 파트는 Content-Type을 명시해야 한다

JSON은 단순 텍스트가 아니기 때문에 반드시 다음과 같은 헤더를 포함해야 한다.

```
Content-Disposition: form-data; name="param"
Content-Type: application/json; charset=utf-8
```

Content-Type을 생략하면 서버는 해당 데이터를 일반 문자열로 판단하여 JSON 디코딩에 실패할 가능성이 높다.

---

### (2) boundary는 전체 multipart를 구분하는 기준

요청 헤더의 Content-Type에 boundary가 포함된다.

```
Content-Type: multipart/form-data; boundary=----ABC123
```

본문은 이 boundary를 기준으로 여러 파트로 나뉜다.

---

### (3) 각 파트는 boundary로 시작해야 한다

```
----ABC123
Content-Disposition: form-data; name="param"
...
```

---

### (4) 헤더와 본문 사이에는 빈 줄(\r\n)이 반드시 필요하다

multipart 문법에서는 헤더와 본문을 구분하는 빈 줄이 필수이다.

```
Content-Disposition: ...
Content-Type: ...

{ JSON 본문 }
```

---

### (5) JSON 본문 뒤에도 줄바꿈이 필요하다

JSON이 끝난 후 다음 파트나 종료 시 반드시 줄바꿈을 넣어야 한다.

```
{ JSON }\r\n
```

---

### (6) 파일 파트는 Content-Type을 반드시 명시해야 한다

파일은 반드시 MIME 타입을 지정해야 한다.

```
Content-Disposition: form-data; name="profileImg"; filename="profile.jpg"
Content-Type: image/jpeg
```

---

### (7) 마지막 boundary는 반드시 "--"로 닫아야 한다

multipart가 끝났음을 알리는 규칙이다.

```
----ABC123--
```

---

## 5. multipart 안에 JSON과 파일을 함께 넣은 전체 예시

```
POST /v1/auth/sign-up HTTP/1.1
Content-Type: multipart/form-data; boundary=----ABC123

----ABC123
Content-Disposition: form-data; name="param"
Content-Type: application/json; charset=utf-8

{
  "username": "gildong",
  "password": "1234",
  "name": "홍길동",
  "birthDate": "1999-01-01",
  "gender": "M"
}
----ABC123
Content-Disposition: form-data; name="profileImg"; filename="profile.jpg"
Content-Type: image/jpeg

<binary image data>
----ABC123--
```

---

## 6. Swift에서 multipart + JSON 조합을 직접 구현하는 예시

아래는 Swift에서 URLSession을 이용해 JSON과 이미지를 multipart로 묶어 전송하는 코드이다.

```swift
func postMultipartWithJSON<T: Decodable, Param: Encodable>(
    url: URL,
    param: Param,
    imageData: Data?
) async throws -> T {

    let boundary = "Boundary-\(UUID().uuidString)"

    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("multipart/form-data; boundary=\(boundary)",
                     forHTTPHeaderField: "Content-Type")

    var body = Data()

    // JSON part
    let encoder = JSONEncoder()
    let jsonData = try encoder.encode(param)

    body.append("--\(boundary)\r\n")
    body.append("Content-Disposition: form-data; name=\"param\"\r\n")
    body.append("Content-Type: application/json; charset=utf-8\r\n\r\n")
    body.append(jsonData)
    body.append("\r\n")

    // Image part
    if let imageData {
        body.append("--\(boundary)\r\n")
        body.append("Content-Disposition: form-data; name=\"profileImg\"; filename=\"profile.jpg\"\r\n")
        body.append("Content-Type: image/jpeg\r\n\r\n")
        body.append(imageData)
        body.append("\r\n")
    }

    // Finish
    body.append("--\(boundary)--\r\n")

    request.httpBody = body

    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(T.self, from: data)
}
```

---

## 7. 정리

multipart/form-data는 텍스트와 파일을 함께 전송해야 할 때 가장 적합한 방식이다. 특히 JSON과 파일을 함께 보내는 경우 아래 규칙들이 반드시 지켜져야 한다.

1. multipart/form-data + boundary 명시
2. JSON 파트에는 Content-Type: application/json
3. boundary로 파트 구분
4. 헤더와 본문 사이에 빈 줄
5. JSON 끝에도 개행
6. 파일 파트에 filename과 MIME 타입 명시
7. 마지막 boundary는 `--` 로 닫기

Swift에서는 URLSession으로 직접 multipart를 구성할 수 있으며, 위 규칙을 정확히 따르면 서버는 JSON과 이미지를 정상적으로 수신할 수 있다.


