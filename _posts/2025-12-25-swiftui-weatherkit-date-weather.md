---
title: "[SwiftUI] WeatherKit으로 현재 날짜·날씨 구현하기 (SwiftUI + CoreLocation + Swift 6)"
date: 2025-12-25
categories:
  - iOS
  - SwiftUI
  - Architecture
tags: [SwiftUI, WeatherKit, CoreLocation, SwiftConcurrency, iOS, 위치기반]
slug: swiftui-weatherkit-date-weather
permalink: /posts/swiftui-weatherkit-date-weather/
---

# SwiftUI에서 WeatherKit으로 날짜 · 날씨 구현하기  
### (SwiftUI + CoreLocation + Swift 6 기준 정리)

대시보드 화면 상단에  
**현재 날짜**, **오늘 여부**, **현재 위치 기반 날씨와 온도**를 표시하는 기능은  
운동 앱, 헬스케어 앱, 라이프로그 앱에서 거의 필수에 가깝다.

iOS 16부터 Apple은 `WeatherKit`을 공식 제공하며  
외부 API 없이도 **신뢰도 높은 날씨 데이터**를 사용할 수 있게 되었다.

이 글에서는 다음에 집중한다.

- SwiftUI에서 WeatherKit을 사용하는 전체 흐름
- CoreLocation과 결합하는 구조
- Swift 6 기준에서 안전한 설계 포인트
- 실제 구현 시 반드시 체크해야 할 사항

---

## 1. 전체 설계 개요

### 핵심 역할 분리

| 구성 요소 | 역할 |
|--------|------|
| View (`CDDateWeatherBarView`) | 날짜 · 날씨 UI 표시 |
| ViewModel | 날짜 계산, 위치 요청, 날씨 요청 |
| CoreLocation | 사용자 현재 위치 제공 |
| WeatherKit | 위치 기반 날씨 데이터 제공 |

핵심 원칙은 단순하다.

> **View는 표시만 담당하고  
> 날짜·날씨 계산과 네트워크 로직은 ViewModel에 둔다**

---

## 2. View: 날짜·날씨 표시 전용 컴포넌트

```swift
import SwiftUI

struct CDDateWeatherBarView: View {
    @StateObject private var vm = DashboardWeatherViewModel()

    var body: some View {
        HStack {
            Text(vm.dateText)
                .font(.subheadline)
                .fontWeight(.medium)

            Text(vm.todayText)
                .font(.subheadline)
                .foregroundStyle(.secondary)

            Spacer()

            Text(vm.weatherText)
                .font(.subheadline)
                .foregroundStyle(.secondary)
        }
        .task {
            vm.requestLocation()
        }
    }
}
````

### View의 책임

* 날짜 문자열 표시
* 날씨 문자열 표시
* ViewModel 상태 바인딩

👉 **위치 요청, WeatherKit 호출 로직은 없음**

---

## 3. ViewModel: 날짜 계산 + 위치 + 날씨

```swift
import Foundation
import CoreLocation
import WeatherKit
import Observation

@Observable
final class DashboardWeatherViewModel: NSObject {

    var dateText: String = ""
    var todayText: String = ""
    var weatherText: String = "날씨 불러오는 중…"

    private let locationManager = CLLocationManager()
    private let weatherService = WeatherService()

    override init() {
        super.init()
        locationManager.delegate = self
        updateDate()
    }

    func requestLocation() {
        locationManager.requestWhenInUseAuthorization()
        locationManager.requestLocation()
    }

    private func updateDate() {
        let formatter = DateFormatter()
        formatter.locale = Locale(identifier: "ko_KR")
        formatter.dateFormat = "M월 d일"

        dateText = formatter.string(from: Date())
        todayText = "오늘"
    }

    private func fetchWeather(for location: CLLocation) async {
        do {
            let weather = try await weatherService.weather(for: location)
            let temp = Int(weather.currentWeather.temperature.value.rounded())
            let condition = weather.currentWeather.condition.koreanText

            await MainActor.run {
                self.weatherText = "\(condition) \(temp)°C"
            }
        } catch {
            await MainActor.run {
                self.weatherText = "날씨 실패"
            }
        }
    }
}
```

---

## 4. CoreLocation Delegate 처리 (Swift 6 기준)

```swift
extension DashboardWeatherViewModel: CLLocationManagerDelegate {

    nonisolated func locationManagerDidChangeAuthorization(
        _ manager: CLLocationManager
    ) {
        if manager.authorizationStatus == .authorizedWhenInUse ||
           manager.authorizationStatus == .authorizedAlways {
            manager.requestLocation()
        }
    }

    nonisolated func locationManager(
        _ manager: CLLocationManager,
        didUpdateLocations locations: [CLLocation]
    ) {
        guard let location = locations.last else { return }

        Task {
            await self.fetchWeather(for: location)
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

### 설계 포인트

* Delegate 메서드는 **nonisolated**
* UI 상태 변경은 **MainActor에서만 수행**
* 외부 시스템 콜백과 UI 상태 변경을 명확히 분리

---

## 5. WeatherCondition → 한글 문자열 변환

WeatherKit의 `WeatherCondition`은
UI에 바로 쓰기엔 적절한 문자열을 제공하지 않는다.

따라서 확장(extension)을 통해 변환 계층을 둔다.

```swift
import WeatherKit

extension WeatherKit.WeatherCondition {
    var koreanText: String {
        switch self {
        case .clear: return "맑음"
        case .mostlyClear: return "대체로 맑음"
        case .partlyCloudy: return "구름 조금"
        case .mostlyCloudy: return "대체로 흐림"
        case .cloudy: return "흐림"
        case .foggy: return "안개"
        case .rain: return "비"
        case .snow: return "눈"
        case .thunderstorms: return "천둥번개"
        default: return "날씨"
        }
    }
}
```

### 체크 포인트

* 반드시 `WeatherKit.WeatherCondition`에 확장
* 동일한 이름의 커스텀 enum 사용 금지

---

## 6. 반드시 체크해야 할 설정들

### 6-1. Info.plist

```text
NSLocationWhenInUseUsageDescription
→ "현재 위치의 날씨 정보를 표시하기 위해 필요합니다."
```

### 6-2. Signing & Capabilities

* WeatherKit ✅
* iOS 16 이상

### 6-3. 테스트 환경

* **실제 기기 테스트 권장**
* 시뮬레이터 사용 시 위치 설정 필수

---

## 7. 정리

SwiftUI에서 WeatherKit을 사용할 때 핵심은 다음이다.

* 날짜·날씨는 ViewModel 책임
* 위치 요청 → WeatherKit → UI 업데이트 흐름 명확화
* 외부 콜백과 UI 변경의 Actor 경계 분리
* Apple 제공 타입과 이름 충돌 방지

> WeatherKit은 단순히 “날씨 API”가 아니라
> **Swift Concurrency와 잘 맞물리도록 설계된 시스템 프레임워크**다.

올바른 구조로 접근하면
외부 API 없이도 안정적인 날씨 기능을 구현할 수 있다.
