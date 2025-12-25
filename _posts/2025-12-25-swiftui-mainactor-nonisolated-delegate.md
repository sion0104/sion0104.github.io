---
title: "[SwiftUI] @MainActor와 nonisolated Delegate 충돌 이해하기 (Swift 6 동시성 이론 정리)"
date: 2025-12-25
categories:
  - iOS
  - SwiftUI
  - Architecture
tags: [SwiftUI, SwiftConcurrency, MainActor, nonisolated, Delegate, Actor, iOS, 상태격리]
slug: swiftui-mainactor-nonisolated-delegate
permalink: /posts/swiftui-mainactor-nonisolated-delegate/
---

# SwiftUI 동시성 설계 개선기  
### @MainActor ViewModel과 nonisolated Delegate 충돌을 이론적으로 이해하기 (Swift 6)

날씨 기능을 구현하며 `CLLocationManagerDelegate`를 ViewModel에서 직접 처리하던 중,  
Swift 6 언어 모드에서 **기존에는 보지 못했던 에러**를 마주하게 되었다.

이 에러는 단순한 문법 문제가 아니라  
**Swift Concurrency의 “Actor 격리 규칙”을 정확히 이해하지 못하면 피할 수 없는 구조적 충돌**이었다.

이 글에서는 다음 흐름으로 문제를 정리한다.

- 문제 (Swift 6 에러)
- 원인 (Actor isolation & Protocol requirement)
- 해결 전략 (nonisolated + Actor hop)
- 구현 코드
- 트레이드오프 및 정리

---

## 1. 문제: Swift 6에서 발생한 Delegate 격리 에러

`@MainActor`로 격리된 ViewModel에서  
`CLLocationManagerDelegate`를 직접 구현하고 있었다.

```swift
@MainActor
final class DateWeatherViewModel: NSObject, ObservableObject {
    ...
}

extension DateWeatherViewModel: CLLocationManagerDelegate {
    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) { ... }
}
````

Swift 6 언어 모드에서 아래 에러가 발생했다.

> Main actor-isolated instance method
> cannot be used to satisfy nonisolated protocol requirement
> (this is an error in Swift 6 language mode)

즉, **컴파일 자체가 불가능**해졌다.

---

## 2. 원인: Actor 격리와 Protocol 요구사항의 충돌

### 2-1. @MainActor의 의미

```swift
@MainActor
final class ViewModel { ... }
```

이는 단순한 “메인 스레드에서 실행” 선언이 아니다.

* 해당 인스턴스의 **모든 메서드와 프로퍼티는**
* **MainActor에 격리(isolated)** 된다
* 즉, **다른 Actor / 스레드에서 직접 접근 불가**

이것이 Swift Concurrency가 데이터 레이스를 막는 핵심 장치다.

---

### 2-2. Delegate 프로토콜의 본질

`CLLocationManagerDelegate`는 Objective-C 기반 프로토콜이다.

이런 Delegate의 특징은:

* 콜백이 **어느 스레드에서 호출될지 보장되지 않음**
* Swift Concurrency 관점에서는 **nonisolated 요구사항**으로 해석됨
* 즉, “어떤 Actor에서도 호출 가능해야 함”

---

### 2-3. 왜 충돌이 발생했는가?

정리하면 구조는 이랬다.

| 항목            | 격리 수준               |
| ------------- | ------------------- |
| ViewModel     | @MainActor (격리됨)    |
| Delegate 요구사항 | nonisolated (격리 없음) |

Swift는 이렇게 판단한다.

> “nonisolated 메서드를 요구하는 프로토콜을
> MainActor에 격리된 메서드로 구현할 수 없다”

Swift 5.x에서는 경고 수준이었지만,
**Swift 6에서 언어 규칙으로 에러 처리**되었다.

---

## 3. 해결 전략: Delegate는 nonisolated, UI는 MainActor

핵심 전략은 단 하나다.

> **외부 콜백은 nonisolated로 받고,
> UI 상태 변경만 MainActor에서 수행한다**

이를 위해 다음 원칙을 세웠다.

### 전략 요약

1. ViewModel 전체를 `@MainActor`로 만들지 않는다
2. Delegate 메서드는 `nonisolated`로 유지
3. `@Published` 상태 변경은 `MainActor`에서만 수행

---

## 4. 구현 코드: Swift 6에서 안전한 Delegate 처리

### 4-1. ViewModel (Actor 비격리)

```swift
final class DateWeatherViewModel: NSObject, ObservableObject {

    @Published var weatherText: String = "날씨 불러오는 중…"

    private let locationManager = CLLocationManager()

    override init() {
        super.init()
        locationManager.delegate = self
    }
}
```

---

### 4-2. Delegate 구현 (nonisolated)

```swift
extension DateWeatherViewModel: CLLocationManagerDelegate {

    nonisolated func locationManager(
        _ manager: CLLocationManager,
        didUpdateLocations locations: [CLLocation]
    ) {
        Task { @MainActor in
            self.weatherText = "날씨 갱신됨"
        }
    }

    nonisolated func locationManager(
        _ manager: CLLocationManager,
        didFailWithError error: Error
    ) {
        Task { @MainActor in
            self.weatherText = "위치 오류"
        }
    }
}
```

### 핵심 포인트

* Delegate 메서드는 **nonisolated** → 프로토콜 요구사항 충족
* UI 상태 변경은 `Task { @MainActor in }`로 **Actor hop**
* Swift 6 규칙을 정확히 만족

---

## 5. 왜 ViewModel 전체를 @MainActor로 두지 않았는가?

UI 전용 ViewModel이라면 `@MainActor`가 편할 수 있다.

하지만:

* Delegate
* Completion handler
* Notification
* System callback

처럼 **호출 컨텍스트가 불확실한 외부 진입점**이 있다면,

> ViewModel 전체를 MainActor에 묶는 순간
> 프로토콜 격리 충돌 위험이 급격히 커진다

이번 문제가 그 대표적인 예였다.

---

## 6. 트레이드오프와 결론

### 장점

* Swift 6 동시성 규칙과 완전 호환
* Delegate / Actor 격리 개념이 명확
* 구조적 이유가 분명한 코드

### 단점

* `@MainActor`를 무조건 붙이던 방식보다 사고 비용 증가
* Actor hop에 대한 이해 필요

### 결론

Swift 6 시대의 설계 핵심은 다음 문장으로 요약된다.

> **“외부 입력은 nonisolated로 받고,
> UI 변경은 명시적으로 MainActor에서 수행하자”**

이는 단순한 문법 대응이 아니라,
**Swift Concurrency 철학에 맞는 구조적 설계 전환**이다.

---

이 글은
Timer → Date 기반 설계 전환과 마찬가지로,

> “Swift 6은 기존 패턴을 금지하는 것이 아니라
> 더 명확한 사고를 요구한다”

는 점을 다시 한 번 느끼게 해준 사례였다.

