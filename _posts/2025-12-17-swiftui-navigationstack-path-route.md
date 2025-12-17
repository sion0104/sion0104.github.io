---

title: "[SwiftUI] NavigationStack과 NavigationPath를 제대로 써야 하는 이유 (Route 기반 네비게이션 정리)"
date: 2025-12-15
categories:
  - iOS
  - SwiftUI
  - Architecture
tags: [SwiftUI, NavigationStack, NavigationPath, Route, iOS, 상태기반네비게이션]
slug: swiftui-navigationstack-path-route
permalink: /posts/swiftui-navigationstack-path-route/

---

# NavigationStack과 NavigationPath를 제대로 써야 하는 이유

SwiftUI에서 화면 이동을 구현하다 보면  
`NavigationLink`만으로는 설명되지 않는 이상한 동작을 마주하게 된다.

이번에 겪은 문제는 다음과 같았다.

> **회원정보 수정 플로우에서  
> → 수정 완료 후 루트로 돌아가야 하는데  
> → 계속 NavigationLink를 타고 엉뚱한 화면으로 이동하는 현상**

이 글에서는 이 문제가 **왜 발생했는지**,  
그리고 **NavigationStack + NavigationPath + Route** 개념을 통해  
어떻게 구조적으로 해결했는지를 정리한다.

---

## 1. 문제 상황 요약

구현하고자 했던 플로우는 단순했다.

```text
마이페이지
 → 비밀번호 인증
 → 회원정보 수정
 → 완료 후 마이페이지로 복귀
````

하지만 실제 동작은 달랐다.

* 수정 완료 후
* `dismiss()`를 호출했음에도
* 예상하지 못한 NavigationLink를 거쳐
* RootView로 튀어버리는 현상이 발생했다

---

## 2. NavigationLink 기반 네비게이션의 한계

초기 구현은 대부분 이런 형태였다.

```swift
NavigationLink {
    PersonalInfoPasswordVerifyView()
} label: {
    Text("회원정보 변경")
}
```

이 방식의 문제점은 명확하다.

* 화면 이동 로직이 **UI에 묶여 있음**
* 중간 단계를 건너뛰거나
* “완료 시 루트로 복귀” 같은 플로우 제어가 어려움
* `dismiss()` 호출 시, **어디로 돌아갈지 예측하기 어려움**

즉, **복잡한 플로우를 표현하기에는 부적합**하다.

---

## 3. NavigationStack의 핵심 개념

### 상태 기반 네비게이션

`NavigationStack`의 핵심 철학은 이것이다.

> **“화면 이동은 링크가 아니라 상태다.”**

```swift
NavigationStack(path: $path) { ... }
```

여기서 `path`는 단순한 배열이 아니라,
**현재 네비게이션 스택 전체를 표현하는 상태**다.

---

## 4. NavigationPath란 무엇인가?

`NavigationPath`는 다음과 같은 역할을 한다.

* 현재 어떤 화면들이 스택에 쌓여 있는지 저장
* `append()` → push
* `removeLast()` → pop
* `removeLast(path.count)` → pop to root

```swift
@State private var path = NavigationPath()

path.append(MyRoute.editInfo)
path.removeLast()
path.removeLast(path.count)
```

이제 화면 이동은 **함수 호출로 제어 가능**해진다.

---

## 5. Route(enum)가 필요한 이유

`NavigationPath`에 들어가는 값은 반드시 `Hashable`이어야 한다.

이때 가장 명확한 방법이 **Route enum**이다.

```swift
enum MyPageRoute: Hashable {
    case passwordVerify
    case editInfo
}
```

이 방식의 장점:

* 화면 이동이 “문자열”이나 “뷰 타입”이 아님
* **의미 있는 화면 상태**로 관리됨
* 플로우가 코드 상에서 한눈에 보임

---

## 6. navigationDestination의 역할

`navigationDestination`은 다음 질문에 답한다.

> “이 Route가 path에 들어오면,
> 어떤 View를 보여줘야 하는가?”

```swift
.navigationDestination(for: MyPageRoute.self) { route in
    switch route {
    case .passwordVerify:
        PasswordVerifyView(...)
    case .editInfo:
        EditInfoView(...)
    }
}
```

여기서 중요한 규칙이 있다.

> `path.append()`에 넣는 타입과
> `navigationDestination(for:)`의 타입은
> **반드시 정확히 일치해야 한다.**

---

## 7. 실제 해결 구조

### 핵심 전략

* NavigationStack은 **부모가 소유**
* 자식 View는 path를 직접 만지지 않음
* 대신 **이벤트(closure)만 위로 전달**

---

## 8. 전체 예시 코드

아래는 실제로 문제를 해결한 구조를 단순화한 예시다.

```swift
enum MyPageRoute: Hashable {
    case passwordVerify
    case editInfo
}
```

```swift
struct MypageSettingView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            Button("회원정보 변경") {
                path.append(MyPageRoute.passwordVerify)
            }
            .navigationDestination(for: MyPageRoute.self) { route in
                switch route {
                case .passwordVerify:
                    PasswordVerifyView {
                        path.append(.editInfo)
                    }
                case .editInfo:
                    EditInfoView {
                        path.removeLast(path.count)
                    }
                }
            }
        }
    }
}
```

---

## 9. 이 경험에서 얻은 교훈

이번 이슈를 통해 확실히 느낀 점은 다음과 같다.

* NavigationLink는 **단순한 화면 이동용**
* 플로우가 있는 화면은 반드시 NavigationPath로 관리
* dismiss()에 의존하면 구조가 무너진다
* 네비게이션은 UI가 아니라 **상태로 설계해야 한다**

---

## 10. 정리

SwiftUI의 네비게이션은
“어디로 가는가”보다
**“현재 어떤 상태인가”**가 더 중요하다.

> **NavigationStack + NavigationPath + Route 조합은
> 복잡한 플로우를 다루기 위한 사실상 표준 구조다.**

한 번 이 구조에 익숙해지면,
네비게이션 버그의 절반은 사라진다.

---
