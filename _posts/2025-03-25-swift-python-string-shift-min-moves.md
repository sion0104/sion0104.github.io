---
title: "[Algorithm] 문자열 밀기: 최소 이동 횟수 찾기" 
date: 2025-03-25
categories:
  - Development
  - Algorithm
  - Python
  - Swift
  - Problem Solving
tags: [string manipulation, algorithm, python, swift, coding text, problem solving, 문자열 밀기, 프로그래머스]
description: "프로그래머스 Level 0 문자열 밀기 문제 풀이"
link: https://school.programmers.co.kr/learn/courses/30/lessons/120921?language=python3
slug: swift-python-string-shift-min-moves
---

# 문자열 밀어서 비교하기: 최소 이동 횟수 찾기

## 문제 출처
[프로그래머스 Level 0 - 문자열 밀기](https://school.programmers.co.kr/learn/courses/30/lessons/120921?language=python3)

## 문제 설명
주어진 문자열 A를 오른쪽으로 밀어서 문자열 B로 만들 수 있는 최소 횟수를 찾는 알고리즘입니다.
## 문제 조건
- 문자열 A와 B의 길이는 동일해야 합니다.
- 0 < 문자열 길이 < 100
- 알파벳 소문자로만 구성됩니다.

## Python 솔루션

```python
from collections import Counter

def solution(A, B):
    # 이미 동일한 문자열인 경우
    if A == B:
        return 0
    
    # 문자 구성 확인
    if Counter(A) != Counter(B):
        return -1
    
    # 문자열 밀기
    for count in range(1, len(A)):
        A = A[-1] + A[:-1]
        if A == B:
            return count
    
    return -1
```

## Swift 솔루션

```swift
func solution(_ A: String, _ B: String) -> Int {
    // 이미 동일한 문자열인 경우
    if A == B {
        return 0
    }
    
    // 문자 구성 확인
    if Set(A.sorted()) != Set(B.sorted()) {
        return -1
    }
    
    // 문자열 밀기
    var newA = A
    for count in 1..<A.count {
        newA = String(newA.last!) + newA.dropLast()
        if newA == B {
            return count
        }
    }
    
    return -1
}
```

## 알고리즘 설명

1. 먼저 A와 B가 완전히 동일한지 확인합니다.
2. 문자 구성(Counter)을 비교하여 변환 가능성을 판단합니다.
3. 문자열을 오른쪽으로 밀면서 B와 일치하는지 확인합니다.
4. 최대 문자열 길이만큼 반복하며 일치 여부를 검사합니다.

## 시간 복잡도
- O(n²), n은 문자열의 길이

## 주의사항
- 문자열 밀기는 순환적으로 이루어집니다.
- 문자 구성이 다르면 변환 불가능합니다.

## 예시
- "hello" → "ohell": 1회 이동
- "apple" → "elppa": 불가능 (-1)
