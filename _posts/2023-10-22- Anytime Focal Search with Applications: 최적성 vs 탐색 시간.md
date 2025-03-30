---
title:  "Anytime Focal Search with Applications: 최적성 vs 탐색 시간"
excerpt: "Algorithm"

categories:
  - Path_Finding

tags:
  - [Algorithm, Path Finding, Searching]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2023-10-22
last_modified_at: 2023-10-22
---

# **Anytime Focal Search with Applications: 최적성 vs 탐색시간**
---
이번에는 경로탐색 알고리즘에서 자주쓰이는 $A^\*$알고리즘의 변형인 Focal Search에 대해 자세히 알아보려고 한다. 회사에서는 로봇 관제를 하기위해 [EECBS](https://arxiv.org/abs/2010.01367)알고리즘을 사용하고 있었는데, 해당 코드가 내가 해결해야하는 다이나믹하고, 비정형 환경에 맞지 않는 문제점이 있었다. 그래서 해당 문제를 해결하고자 [CBS](https://reofard.github.io/path_finding/2023/05/20/CBS-MAPF%EA%B3%84%EC%9D%98-%EC%84%B1%EA%B2%BD.html)알고리즘을 직접 변형 및 구현하여, 그래프맵에서 비정형 충돌과 로봇의 감가속을 고려하여 Continuous하게 알고리즘을 새로 개발했었는데, 그 과정에서 EECBS에서 적용되었던 bounded-suboptimal search를 사용하지 못했었다. 단순하게 내가 해당 알고리즘에 대한 이해도가 부족해서였는데, 이번 논문에서 **bounded-suboptimal search란 무엇이고, 어떤 이점이 있어서 사용하는지**를 알아보기 위해서 해당 논문을 리뷰해보고 공부하려고 한다.

<br>

# $A^\*$ **알고리즘이란?**
---
우선 간단하게 $A^\*$ 알고리즘에 대해 알아보도록하자. $A^\*$ 알고리즘은 최선 우선 탐색(best-first search) 알고리즘의 일종으로 $OPEN-List$에서 평가값이 가장 낮은 상태롤 확장하여 새롭게 $OPEN-List$에 넣어 상태를 확장하고 탐색하는 알고리즘이다. 이때 특정 상태 $n$의 평가값은 $f(n)=g(n)+h(n)$이고, $g(n)$과 $h(n)$의 정의는 아래와 같다.

- $g(n)$: 시작 상태에서 $n$이 되는데 걸린 시간 혹은 비용
- $f(n)$: $n$에서 목표 상태가 되는데 필요한 예상(heuristics)비용

하지만 완벽하지 않은 휴리스틱이 사용되지 않는경우 $A^\*$는 비효율적인 경우가 있고, 이에 따라, 최적해를 반드시 찾을 필요 없이 탐색 속도를 높이는 다양한 $A^\*$의 변형 알고리즘이 제안되었고, 그중 하나가 Focal Search이다. $A^\*$ 탐색은 admissible 휴리스틱을 사용하지만, 최적 솔루션을 찾는 데 시간이 오래 걸릴 수 있다고 한다. 이 말은, $A^\*$는 $f(n)=f_{min}$​인 상태만 확장하는데, 이는 **좋은 상태의 판정 기준이 1개뿐이라는 뜻**이 된다. 그렇기 때문에 $A^\*$는 유연성이 떨어진다고 한다.

<br>

# BSS(Bounded-Suboptimal Search)와 BCS(Bounded-Cost Search)
---
FS(Focal Search)는 **"완벽하게 최적은 아니지만, 충분히 좋은 해(solution)를 빠르게 찾는 탐색 방법"**이다. 이를 이해하기 위해 필요한 BSS(Bounded-Suboptimal Search)와 BCS(Bounded-Cost Search)의 개념을 먼저 정리해보려고 한다.

먼저 **BSS (Bounded-Suboptimal Search)란** **사용자가 "해결책이 최적보다 최대 w배 비싸도 괜찮다"**라고 지정하는 탐색 방식이다. 즉 최적 경로의 비용이 $P_{opt}$​라면, BSS는 최대 $wP_{opt}$ 이내의 경로를 보장한다. 
두번째로 **BCS(Bounded-Cost Search)란** **사용자가 "비용이 C 이하인 해결책만 찾아줘"** 라고 지정하는 방식이다. 즉 경로의 비용이 C를 초과하지 않는 해만 허용한다. 아래는 각각의 방법론 별 예시이다.

1. **BSS(Bounded-Suboptimal Search):** 최적 경로가 10이라면, w=1.5w=1.5이면 비용이 최대 15까지 허용
2. **BCS(Bounded-Cost Search):** 비용이 100을 초과하면 안 된다면, BCS는 비용 100 이하의 해결책을 찾는 것만 허용

이러한 기준들은 Focal Search에서 Focal List를 구축하는 과정에서 유용하게 사용된다고 한다.

<br>

# **Focal Search(FS) 알고리즘**
---

이제 Focal Search(FS)에 대해서 본격적으로 알아보자. Focal Search(FS)는 **(1) 어떤 상태를 FOCAL list에 넣을지**, 그리고 **(2) 그 안에서 어떤 순서로 확장할지**라는 두 가지 독립적인 요소로 구성된다. 각각의 요소가 무슨 의미이고, 어떤 역할을 하는지 자세히 알아보자.

### **Definition of FOCAL List**
FS의 핵심은 Focal List이다. Focal List는 **"현재 확장할 수 있는 상태($n \in OPEN$)들 중, 충분히 좋은 것들만 모아놓은 리스트"** 라고 생각하면 된다. 이때 **충분히 좋은것**의 기준은 OPEN List의 데이터중 가장 평가값이 낮은 값($f_{min}$)이 된다. 그렇기 때문에 Focal List는 아래와 같은 식을 통해 정의할 수 있다.

1. **BSS(Bounded-Suboptimal Search) 기반**

    >$FOCAL=\{n \in OPEN \mid f(n) \le wf_{min}\}$


2. **BCS(Bounded-Cost Search) 기반**

    >$FOCAL=\{n \in OPEN \mid f(n) \le C\}$

즉, BSS는 "적당히 좋은 해를 빠르게 찾자", BCS는 "비용을 넘지 않는 해만 찾자" 정도로 해석할 수 있다.

<br>

## **Priority Function**
---
FS를 BSS관점에서 볼 때 $A^\*$의 한계를 극복하기 위한 방법으로 FOCAL List가 사용된다. FOCAL List는 OPEN List와 별개의 우선순위를 가질 수 있는데, 이는 FOCAL List를 이용하여 **"충분히 좋은" 솔루션을 찾기 위해 FOCAL 리스트에 있는 어떤 상태라도 확장**할 수 있음을 의미한다. 이 유연성 덕분에 **FS는 A보다 더 빠르면서, 준최적성 경계를 보장**할 수 있다. 이렇게 FOCAL List에서 어떤 상태를 우선적으로 확장할지에 대한 기준을 $h_{FOCAL}(n)$ 함수로 나타낸다.

### 1. $w$-admissible $h_{FOCAL}$

우선순위 함수 $h_{FOCAL}$이 $w$-admissible이라고 하면 모든 확장된 상태 n에 대해 $g(n) \le wg^\*(n)$을 만족한다. 이때 $g^\*(n)$은 시작상태에서 n까지의 최적 비용이다. 하지만 모든 $h_{FOCAL}$가 이 속성을 만족하는 것은 아니다.



예를 들어, 아래 그림과 같은 그래프를 고려해보자. ```S```는 시작 상태이고, ```G```는 목표 상태이다. ```w = 2```라고 하고, 우선순위 함수 $h_{FOCAL}$은 알파벳 역순으로 정렬한다고 가정하고 아래와 같은 상황이라고 해보자.

![w-admissible 비허용](/assets/img/w-admissible_invalid.png)

**예제 상황 설명**

1. ```S```를 확장하면 ```A```와 ```C```가 OPEN List에 추가

    - $g(A) = 1, f(A) = 11$
    - $g(C) = 8, f(C) = 16$

2. $f_{min} = 11$이므로 FOCAL List에는 $f(n) \leq wf_{min} = 2 \times 11 = 22$를 만족하는 상태들이 들어간다.

    - ```A```와 ```C``` 모두 FOCAL에 포함

3. $h_{FOCAL}$이 알파벳 역순이므로 ```C```가 먼저 확장

4. $g(C) = 8, g^*(C) = 3$이므로 $g(C) > wg^\*(C) = 2 \times 3 = 6$

5. 즉, $g(C)$가 $wg^\*(C)$보다 크므로 $w$-admissibility 조건을 위반

결과론 적으로 위 예제와 같이 $f_{FOCAL}$이 $w$-admissible하지 않으면 불필요한 상태를 확장하게 되어, 탐색 효율이 떨어지고 경우에 따라 w-bounded optimality 보장을 깨뜨릴 수도 있다. 그렇기 때문에 $f_{FOCAL}$를 설정할 때는 $w$-admissibility 속성을 고려하는 것이 중요하다고 한다.

### 2. Efficiently reusable $h_{FOCAL}$

이번 속성은 기본적인 FS에서 다루는 속성이 아닌, 이번 논문에서 저자가 제안하는 Anytime Focal Search에서 사용하는 개념에 가깝다. $w$나 $C$같이 FOCAL List의 정렬 순서에 영향을 미치는 값을 유동적으로 할수 있는지에 대한 내용이다. 우선순위 함수 $h_{FOCAL}$이 **efficiently reusable** 하다는 것은 이 함수가 suboptimality bound $w$ 또는 cost bound $C$에 의존하지 않는다는 의미이다. 즉, $w$나 $C$가 변경되더라도 FOCAL List를 다시 정렬할 필요가 없다.

하지만 모든 $h_{FOCAL}$가 이 속성을 가지는 것은 아니다. 예를 들어, **wA***와 **Potential Search (PS)**의 경우, 우선순위 함수가 $w$나 $C$에 직접적으로 의존하기 때문에, 이 값들이 변경될 때마다 FOCAL List를 다시 정렬해야 한다. 이러한 과정은 연산 비용이 크기 때문에 이번 논문에서 제안하는 **anytime search**와 같은 반복적인 탐색 환경에서는 비효율적일 수 있다고 한다.

<br>

## **FOCAL Search Pseudocode**
---
결과적으로 FS는 **Bounded-Suboptimal Search(BSS)**와 **Bounded-Cost Search(BCS)**등의 방식을 통해 탐색을 수행한다. 이때 FS 알고리즘의 목적은 **목표 상태를 빠르게 찾되, 충분히 좋은 해를 찾기 위한 유연성을 제공**하는 것이라고 볼 수 있다. 아래의 그림은 FS 알고리즘의 동작 방식을 pseudocode로 나타낸 방식이다.

![focal search pseudo](/assets/img/FS_pseudo.png)

각 줄의 내용을 간단히 설명하면 다음과 같다.

1. **초기화 단계** (Lines 3-4)
    - 알고리즘은 **시작 상태(nstart)**를 FOCAL List와 OPEN List에 추가하여 시작

2. **반복문** (Lines 5-17)
    - FOCAL의 첫 번째 상태를 pop (Line 5)
    - 해당 상태를 OPEN List에서 제거 (Line 6)
    - 해당 상태가 **목표 상태(goal state)**이면, 해결책을 반환하고 종료 (Lines 9-10)
    - 해당 상태의 자식 상태(successors)를 생성 및 OPEN과 FOCAL에 추가 (Lines 11-14)
    - FOCAL List 업데이트 (Lines 15-16)
`
<br>

## **Anytime Focal Search**
---
내용

<br>

# 후기
---
사실 오늘 읽은 논문은 Focal Search가 최초로 소개된 원조 논문은 아니다. 원래는 [Studies in Semi-Admissible Heuristics](https://ieeexplore.ieee.org/document/4767270) 이 논문을 읽고 싶었는데 유료여서 다른 논문을 찾아보게되었다. 그래서 그나마 인용수도 많고, MAPF와 관련된 내용도 있으면서 괜찮은 학회에서 Focal Search의 변형을 소개하는 논문을 읽었는데, Focal Search에 대한 간단한 소개도 있고, 내용도 괜찮아보여서 해당논문을 고르게되었다.

회사에서 자료조사가 완벽하지 않은 시점에 기존에 사용하던 EECBS 알고리즘을 사용하지 못하고, 단순하게 CBS를 우리의 도메인에 맞게 변형하여 사용할 수 밖에 없었는데, 이번 공부로 휴리스틱에 대한 이해도도 높아지고, 요즘 MAPF의 추세인, 최적성을 조금 포기하고, 계산속도를 올리는 방법도 한번 고민할 수 있게 되는 계기가 된 것 같다.