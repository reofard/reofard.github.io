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
이번에는 경로탐색 알고리즘에서 자주쓰이는 $A^*$알고리즘의 변형인 Focal Search에 대해 자세히 알아보려고 한다. 회사에서는 로봇 관제를 하기위해 [EECBS](https://arxiv.org/abs/2010.01367)알고리즘을 사용하고 있었는데, 해당 코드가 내가 해결해야하는 다이나믹하고, 비정형 환경에 맞지 않는 문제점이 있었다. 그래서 해당 문제를 해결하고자 [CBS]알고리즘을 직접 변형 및 구현하여, 그래프맵에서 비정형 충돌과 로봇의 감가속을 고려하여 Continuous하게 알고리즘을 새로 개발했었는데, 그 과정에서 EECBS에서 적용되었던 bounded-suboptimal search를 사용하지 못했었다. 단순하게 내가 해당 알고리즘에 대한 이해도가 부족해서였는데, 이번 논문에서 과연 bounded-suboptimal search란 무엇이고, 어떤 이점이 있어서 사용하는지를 알아보기 위해서 해당 논문을 리뷰해보고 공부하려고 한다.

<br>

# $A^*$ **알고리즘이란?**
---
우선 간단하게 $A^*$ 알고리즘에 대해 알아보도록하자. $A^*$ 알고리즘은 최선 우선 탐색(best-first search) 알고리즘의 일종으로 $OPEN-List$에서 평가값이 가장 낮은 상태롤 확장하여 새롭게 $OPEN-List$에 넣어 상태를 확장하고 탐색하는 알고리즘이다. 이때 특정 상태 $n$의 평가값은 $f(n)=g(n)+h(n)$이고, $g(n)$과 $h(n)$의 정의는 아래와 같다.

- $g(n)$: 시작 상태에서 $n$이 되는데 걸린 시간 혹은 비용
- $f(n)$: $n$에서 목표 상태가 되는데 필요한 예상(heuristics)비용

하지만 완벽하지 않은 휴리스틱이 사용되지 않는경우 $A^*$는 비효율적인 경우가 있고, 이에 따라, 최적해를 반드시 찾을 필요 없이 탐색 속도를 높이는 다양한 $A^*$의 변형 알고리즘이 제안되었고, 그중 하나가 Focal Search이다. $A^*$ 탐색은 admissible 휴리스틱을 사용하지만, 최적 솔루션을 찾는 데 시간이 오래 걸릴 수 있다고 한다. 이 말은, $A^*$는 $f(n)=f_{min}$​인 상태만 확장하는데, 이는 **좋은 상태의 판정 기준이 1개뿐이라는 뜻**이 된다. 그렇기 때문에 $A^*$는 유연성이 떨어진다고 한다.

<br>

# BSS(Bounded-Suboptimal Search)와 BCS(Bounded-Cost Search)
---
FS(Focal Search)는 "완벽하게 최적은 아니지만, 충분히 좋은 해(solution)를 빠르게 찾는 탐색 방법"이다. 이를 이해하기 위해 필요한 BSS(Bounded-Suboptimal Search)와 BCS(Bounded-Cost Search)의 개념을 먼저 정리해보려고 한다.

먼저 BSS (Bounded-Suboptimal Search)란 **사용자가 "해결책이 최적보다 최대 w배 비싸도 괜찮다"**라고 지정하는 탐색 방식이다. 즉 최적 경로의 비용이 $P_{opt}$​라면, BSS는 최대 $wP_{opt}$ 이내의 경로를 보장한다.

두번째로 BCS(Bounded-Cost Search)란 **사용자가 "비용이 C 이하인 해결책만 찾아줘"** 라고 지정하는 방식이다. 즉 경로의 비용이 C를 초과하지 않는 해만 허용한다.

    예시
    BSS (Bounded-Suboptimal Search): 최적 경로가 10이라면, w=1.5w=1.5이면 비용이 최대 15까지 허용
    BCS (Bounded-Cost Search): 비용이 100을 초과하면 안 된다면, BCS는 비용 100 이하의 해결책을 찾는 것만 허용

이러한 기준들은 Focal Search에서 Focal List를 구축하는 과정에서 유용하게 사용된다고 한다.

<br>

# **Focal Search(FS) 알고리즘**
---

### **Definition of FOCAL List**
이제 Focal Search(FS)에 대해서 본격적으로 알아보자. FS의 핵심은 Focal List이다. Focal List는 **"현재 확장할 수 있는 상태($n \in OPEN$)들 중, 충분히 좋은 것들만 모아놓은 리스트"** 라고 생각하면 된다. 이때 **충분히 좋은것**의 기준은 OPEN List의 데이터중 가장 평가값이 낮은 값($f_{min}$)이 된다. 그렇기 때문에 Focal List는 아래와 같은 식을 통해 정의할 수 있다.

1. **BSS(Bounded-Suboptimal Search) 기반**

    $FOCAL=\{n \in OPEN \mid f(n) \le wf_{min}\}$


2. **BCS(Bounded-Cost Search) 기반**

    $FOCAL=\{n \in OPEN \mid f(n) \le C\}$

즉, BSS는 "적당히 좋은 해를 빠르게 찾자", BCS는 "비용을 넘지 않는 해만 찾자" 정도로 해석할 수 있다.

<br>

### **Priority Function**

FS를 BSS관점에서 볼 때 $A^*$의 한계를 극복하기 위한 방법으로 FOCAL List가 사용된다. FOCAL List는 OPEN List와 별개의 우선순위를 가질 수 있는데, 이는 FOCAL List를 이용하여 **"충분히 좋은" 솔루션을 찾기 위해 FOCAL 리스트에 있는 어떤 상태라도 확장**할 수 있음을 의미한다. 이 유연성 덕분에 **FS는 A보다 더 빠르면서, 준최적성 경계를 보장**할 수 있다.


<br>

# 후기
---
사실 오늘 읽은 논문은 Focal Search가 최초로 소개된 원조 논문은 아니다. 원래는 [Studies in Semi-Admissible Heuristics](https://ieeexplore.ieee.org/document/4767270) 이 논문을 읽고 싶었는데 유료여서 다른 논문을 찾아보게되었다. 그래서 그나마 인용수도 많고, 괜찮은 학회에서 Focal Search의 변형을 소개하는 논문을 읽었는데, Focal Search에 대한 간단한 소개도 있고, 내용도 괜찮아보여서 해당논문을 고르게되었다.
회사에서 자료조사가 완벽하지 않은 시점에 기존에 사용하던 EECBS 알고리즘을 사용하지 못하고, 단순하게 CBS를 우리의 도메인에 맞게 변형하여 사용할 수 밖에 없었는데, 이번 공부로 휴리스틱에 대한 이해도도 높아지고, 요즘 MAPF의 추세인, 최적성을 조금 포기하고, 계산속도를 올리는 방법도 한번 고민할 수 있게 되는 계기가 된 것 같다.