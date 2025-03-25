---
title: "[Python] collections.Counter" 
date: 2025-03-24
categories: 
  - Development
  - Python
  - Libraries/Collections
tags: [Python, Counter, dictionary, collections, PythonTips, DataProcessing]
---

### Counter란?
`Counter`는 딕셔너리의 서브클래스로, **요소의 개수를 세는 데 특화된 자료구조** 입니다.

### Counter의 주요 기능

#### 1. 요소 카운팅
```python
from collections import Counter

text = "hello"
counter = Counter(text)
print(counter)  # {'l': 2, 'h': 1, 'e': 1, 'o': 1}
```

#### 2. 가장 흔한 요소 찾기
```python
print(counter.most_common(2))  # [('l', 2), ('h', 1)]
```

#### 3. 안전한 키 접근
```python
print(counter['z'])  # 0 (KeyError 없음)
```

#### 4. 산술 연산
```python
c1 = Counter(['a', 'b', 'c'])
c2 = Counter(['b', 'c', 'd'])
print(c1 + c2)  # {'b': 2, 'c': 2, 'a': 1, 'd': 1}
```
