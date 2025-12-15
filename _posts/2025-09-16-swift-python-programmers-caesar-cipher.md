---
title: "[Algorithm/Python/Swift] 시저 암호 최적화"
date: 2025-09-16
categories: 
  - Development
  - Python
  - Algorithm
tags: [Python, 파이썬, 프로그래머스, 시저 암호, Swift, 스위프트]
slug: swift-python-programmers-caesar-cipher
permalink: /posts/swift-python-programmers-caesar-cipher/

---

# 프로그래머스: 시저 암호 문제 풀이 및 최적화

문제 링크 👉 [프로그래머스 12926 - 시저 암호](https://school.programmers.co.kr/learn/courses/30/lessons/12926)

---

## 접근 방법

처음에는 단순히 **문자를 하나씩 증가**시키며 구현했지만, 다음과 같은 문제가 있었습니다.

- 이동 횟수 `n`이 클 경우, `for i in range(n)`처럼 매번 루프를 돌게 됨  
- 문자열을 `answer += ...`로 계속 더하면, Python의 **문자열 불변(immutable)** 특성상 비효율 발생  

즉, 불필요하게 **시간 복잡도와 메모리 사용량이 커지는 문제**가 있었습니다.  

---

## 아이디어

곰곰이 생각해보니 굳이 `n`번 반복문을 돌 필요가 없었습니다.  
**모듈러 연산(`% 26`)** 을 사용하면 한 번의 수식으로도 결과를 구할 수 있고, 문자열도 `list`에 모았다가 `join` 하면 효율적일 거라 판단했습니다.  

---

## 파이썬 풀이 (최적화 전)

```python
def solution(s, n):
    answer = ''
    for char in s:
        if char == ' ':
            answer += ' '
        elif char.isupper():
            num = ord(char)
            for i in range(n):
                num += 1
                if num > ord('Z'):
                    num = ord('A')
            answer += chr(num)
        else:
            num = ord(char)
            for i in range(n):
                num += 1
                if num > ord('z'):
                    num = ord('a')
            answer += chr(num)
                    
    return answer
````

### 문제점

1. **`for i in range(n)` 반복문** → `O(n)` 비효율
2. **문자열 누적** → 매번 새 문자열 객체 생성

---

## 파이썬 풀이 (최적화 후)

```python
def solution(s, n):
    answer = []
    for char in s:
        if char == ' ':
            answer.append(' ')
        elif char.isupper():
            answer.append(chr((ord(char) - ord('A') + n) % 26 + ord('A')))
        else:
            answer.append(chr((ord(char) - ord('a') + n) % 26 + ord('a')))
    return ''.join(answer)
```

### 최적화 포인트

1. **모듈러 연산 `% 26`** → 반복문 없이 한 번의 계산으로 이동 처리
2. **리스트 + join** → 문자열 결합 시 불필요한 메모리 낭비 방지

---

## Swift 풀이

```swift
func solution(_ s: String, _ n: Int) -> String {
    var result = ""

    for char in s {
        if char == " " {
            result.append(" ")
        } else if char.isUppercase {
            if let ascii = char.asciiValue {
                let shifted = (Int(ascii) - Int(Character("A").asciiValue!) + n) % 26
                result.append(Character(UnicodeScalar(shifted + Int(Character("A").asciiValue!))!))
            }
        } else if char.isLowercase {
            if let ascii = char.asciiValue {
                let shifted = (Int(ascii) - Int(Character("a").asciiValue!) + n) % 26
                result.append(Character(UnicodeScalar(shifted + Int(Character("a").asciiValue!))!))
            }
        }
    }

    return result
}
```

---

## 핵심 포인트

* **Python**

  * 불필요한 루프 제거 → `O(n)` → `O(1)` 개선
  * 문자열 결합 최적화 (`list` + `join`)

* **Swift**

  * `asciiValue` → `Character` 변환을 통해 Python과 동일한 ASCII 기반 로직 구현
  * `isUppercase`, `isLowercase`로 대소문자 판별
  * `UnicodeScalar`를 활용해 ASCII 값을 다시 문자로 변환

---

## 실행 예시

### Python

```python
print(solution("AB", 1))    # "BC"
print(solution("z", 1))     # "a"
print(solution("a B z", 4)) # "e F d"
```

### Swift

```swift
print(solution("AB", 1))     // "BC"
print(solution("z", 1))      // "a"
print(solution("a B z", 4))  // "e F d"
```

---

## ✅ 정리

* Python: 모듈러 연산 + `join`으로 효율 개선
* Swift: `asciiValue`, `UnicodeScalar` 등을 활용해 동일 로직 구현 가능
* 두 언어 모두 문자열 처리에서 **반복 최소화, 메모리 효율성 확보**가 핵심
