---
title: “[Algorithm/Python/Swift] 달리기 경주" 
date: 2025-08-27
categories: 
  - Development
  - Python
  - Algorithm

tags: [Python, 파이썬, 프로그래머스, 달리기 경주, Swift, 스위프트]
---

# 프로그래머스: 달리기 경주 문제 풀이

문제 링크 👉 [프로그래머스 178871 - 달리기 경주](https://school.programmers.co.kr/learn/courses/30/lessons/178871)

---

## 접근 방법

처음에는 단순히 **swap**으로 구현할까 생각했지만, 다음과 같은 문제가 있었습니다.

- swap 횟수: 최대 **1,000,000 번**
- 배열 길이: 최대 **50,000**
- 인덱스 탐색 비용: `1,000,000 × 50,000`

즉, **시간 초과**가 날 게 뻔했습니다. ⏱️

---

## 아이디어

레벨 1 문제라서 어려운 해결법은 아닐 거라 생각했습니다.  
고민하다 보니, 각 선수의 **현재 인덱스를 저장**하면 효율적으로 해결할 수 있겠다 싶었습니다.  

👉 그래서 **딕셔너리(Dictionary)** 를 이용했습니다.

---

## 파이썬 풀이

```python
def solution(players, callings):
    answer = {player: i for i, player in enumerate(players)}
    
    for i in callings:
        index = answer[i]
        answer[i] -= 1
        answer[players[index-1]] += 1
        players[index - 1], players[index] = players[index], players[index-1]
        
    return players
```

### 풀이 설명
1. 각 선수 이름과 인덱스를 **dict**에 저장합니다.  
2. `callings`에서 불린 선수를 찾습니다.  
3. 해당 선수를 **앞 칸으로 이동**시킵니다.  
4. 앞에 있던 선수와 **자리를 교체**합니다.  

---

## Swift 풀이

```swift
import Foundation

func solution(_ players:[String], _ callings:[String]) -> [String] {
    var players = players
    
    var answer: [String: Int] = [:]
    for (i, player) in players.enumerated() {
        answer[player] = i
    }
    
    for name in callings {
        if let index = answer[name], index > 0 {
            players.swapAt(index, index - 1)
            
            answer[name] = index - 1
            
            let frontPlayer = players[index] 
            answer[frontPlayer] = index
        }
    }
    
    return players
}
```

---

## 핵심 포인트

- **시간 초과 방지** → 매번 인덱스를 찾는 대신, `dict`에 현재 인덱스를 저장해 활용.  
- **교체 방식** → `swap` 사용으로 두 선수의 위치를 효율적으로 변경.  
- Python과 Swift 모두 **O(N)** 수준의 효율성을 확보.  
