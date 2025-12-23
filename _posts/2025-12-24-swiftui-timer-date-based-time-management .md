---

title: "[SwiftUI] Timer를 제거하고 Date 기반으로 시간 관리하기 (Swift 6 & 백그라운드 대응)"
date: 2025-12-24
categories:
  - iOS
  - SwiftUI
  - Architecture
tags: [SwiftUI, SwiftConcurrency, Timer, Date, MainActor, Sendable, iOS, 상태기반설계, 백그라운드처리]
slug: swiftui-timer-date-based-time-management
permalink: /posts/swiftui-timer-date-based-time-management/

---

# SwiftUI 타이머 설계 개선기  
### Timer 제거와 Date 기반 시간 보정으로 Swift 6 & 백그라운드 이슈 해결하기

운동(라이딩) 기능을 구현하면서 **타이머 로직**과 **백그라운드/포그라운드 전환 시 시간 보정** 문제를 동시에 마주하게 되었다.  
특히 Swift 6 언어 모드에서 `Timer`를 사용하는 기존 방식이 더 이상 안전하지 않다는 점이 핵심 이슈였다.

이 글에서는 다음 흐름으로 문제를 정리한다.

- 문제 (Swift 6 Warning)
- 원인 (Sendable / MainActor)
- 해결 전략 (Date 기반 설계)
- 구현 코드 (Timer 제거)
- 트레이드오프 및 정리

---

## 1. 문제: Swift 6에서 발생한 Timer Warning

`@MainActor`로 격리된 ViewModel에서 `Timer`를 사용해 시간을 증가시키는 구조였다.

```swift
timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
    self.elapsedSeconds += 1
}
````

Swift 6 언어 모드에서 아래 경고(사실상 에러)가 발생했다.

> Main actor-isolated property 'elapsedSeconds' can not be mutated from a Sendable closure

이는 단순한 경고가 아니라,
**Swift Concurrency 모델과 기존 Timer 기반 설계가 충돌하고 있음을 의미**한다.

---

## 2. 원인: Sendable 클로저와 MainActor 격리

### 2-1. Timer의 본질적인 문제

* `Timer`의 콜백 클로저는 **Sendable 컨텍스트**로 취급될 수 있다.
* 반면 ViewModel은 `@MainActor`로 격리되어 있다.
* 즉, **다른 액터에서 MainActor 상태를 직접 변경하려는 구조**가 된다.

### 2-2. 기술적으로 가능한 우회 방법

```swift
Task { @MainActor in
    self.elapsedSeconds += 1
}
```

이 방식으로 경고를 없앨 수는 있지만,

* Timer + Task + Actor hop이 섞이며
* 코드 가독성과 설계 명확성이 떨어진다.

더 근본적인 접근이 필요했다.

---

## 3. 해결 전략: “틱 기반”이 아닌 “Date 기반”으로 전환

### 3-1. 기존 사고 방식 (문제 있음)

* 1초마다 타이머가 울린다
* 울릴 때마다 `elapsedSeconds += 1`
* ❌ 백그라운드에서 Timer는 정확하지 않다

### 3-2. 새로운 사고 방식 (해결)

> **시간은 흐른다. 우리는 그 차이를 계산할 뿐이다.**

* “몇 번 tick이 울렸는가” ❌
* “지금 시각 - 시작 시각” ✅

### 핵심 아이디어

* **진짜 시간의 기준(source of truth)** 을 `Date`로 둔다
* Timer는 제거
* UI 업데이트는 `Task.sleep` 기반 ticker로 처리

---

## 4. 구현 코드: Timer 제거 + Date 기반 시간 계산

### 4-1. 핵심 상태 정의

```swift
@MainActor
final class CyclingRideViewModel: ObservableObject {

    enum RideStatus {
        case idle
        case countingDown(Int)
        case running
        case paused
    }

    @Published private(set) var status: RideStatus = .idle
    @Published private(set) var elapsedSeconds: Int = 0

    private var startDate: Date?
    private var accumulatedSeconds: Int = 0

    private var tickerTask: Task<Void, Never>?
}
```

* `startDate`: 현재 러닝이 시작된 시각
* `accumulatedSeconds`: 일시정지 전까지 누적된 시간
* `elapsedSeconds`: UI 표시용 값

---

### 4-2. Date 기반 경과시간 계산

```swift
private func currentElapsedSeconds(now: Date) -> Int {
    let running = startDate.map { Int(now.timeIntervalSince($0)) } ?? 0
    return accumulatedSeconds + running
}
```

* running 상태: `now - startDate`
* paused 상태: 누적 시간만 사용

---

### 4-3. Timer 제거: Task 기반 ticker

```swift
private func startTicker() {
    stopTicker()
    tickerTask = Task {
        while !Task.isCancelled {
            try? await Task.sleep(nanoseconds: 1_000_000_000)
            if status == .running {
                elapsedSeconds = currentElapsedSeconds(now: Date())
            }
        }
    }
}

private func stopTicker() {
    tickerTask?.cancel()
    tickerTask = nil
}
```

* `Task.sleep`은 Swift Concurrency와 자연스럽게 동작
* `@MainActor` 안에서 상태 변경 → 경고 없음
* ticker는 **UI 갱신용**일 뿐, 시간의 진실은 아니다

---

## 5. 백그라운드 / 포그라운드 시간 보정 원리

### 문제 상황

* 앱이 백그라운드로 가면 ticker는 멈춘다
* 하지만 **시간은 계속 흐른다**

### 해결 방식

* ticker가 멈춰도 상관없다
* 복귀 시 `Date()` 기준으로 다시 계산하면 된다

```swift
elapsedSeconds = currentElapsedSeconds(now: Date())
```

### scenePhase 활용

* `.active`로 돌아오는 순간 한 번 즉시 계산
* UX적으로 “숫자가 늦게 갱신되는 느낌” 제거

---

## 6. 트레이드오프와 결론

### 장점

* Swift 6 actor 규칙과 완전히 호환
* 백그라운드/포그라운드 시간 정확
* Timer 의존성 제거
* 테스트 및 유지보수 용이

### 단점

* 구조가 단순한 `Timer + 1초 증가`보다 복잡
* Date 기반 사고가 필요

### 결론

운동/라이딩처럼 **정확한 시간 추적이 중요한 기능**에서는
Timer 중심 설계보다 **Date 기반 설계가 장기적으로 훨씬 안전하고 확장성 있다**.

Swift Concurrency 시대에는

> “시간을 세지 말고, 계산하자”
> 라는 관점이 더 적합하다고 느꼈다.****
