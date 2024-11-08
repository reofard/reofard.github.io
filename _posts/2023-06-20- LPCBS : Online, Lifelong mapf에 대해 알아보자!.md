---
title:  "LPCBS : Online, Lifelong mapf에 대해 알아보자!"
excerpt: "Robot"

categories:
  - Path_Finding

tags:
  - [MAPF, Path Finding, Multi-Robot]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2023-06-20
last_modified_at: 2023-06-20
---

# **Lifelong Multi-Agent Path Finding in A Dynamic Environment**

사실 [이 논문](https://ieeexplore.ieee.org/document/8581181/)을 읽게 된 이유는 Online MAPF에 대해 공부하기 위한 목적은 아니었다. 지금 직장에서는 MAPF를 현실에 적용하기 위해 CBS를 반복적으로 계산하여 실시간 경로를 계산하는 방식으로 Online MAPF 문제를 다루고 있는데, 이때 **이전 경로탐색 결과의 일부를 후속 CBS 연산에서 사용**하여 **연산시간을 확보**(이는 내용은 나중에 다시 다뤄보려고 한다)를 하는 방식으로 기존 학계와 좀 다른 방향으로 Online MAPF를 다루고 있었다. 하지만 연산은 빠르면 빠를수록 좋기 때문에 "이전 연산 결과 뿐 아니라 이전 계산에서 나온 constraint들을 다시 재활용 하여 중복 계산을 제거할 수 없을까?" 라는 의문과 생각한 아이디어에서 찾아본 논문이었다.

이전에도 [Searching with consistent prioritization for multi-agent path finding](https://arxiv.org/abs/1812.06356)같은 논문에서 정보를 재활용 해서 하는 방식이 소개 되었지만, 직접적으로 constraint를 재활용 하는 방식은 없었던 것 같다. 아직 해당 논문을 읽어보진 못해서 간단하게 constraint를 이용한 결과가 없었던 이유는 [밤늦게 여는 카페](https://goodahn.tistory.com/)님의 블로그의 [포스트](https://goodahn.tistory.com/214)에서 간접적으로 찾을 수 있었다. 자꾸 공부하면 공부하면서 더 찾고 읽어볼것이 많아지는것 같다 ㅠㅜ

그렇기 때문에 이번 포스트에서는 비슷했던 내 아이디어도 정리할 겸, 해당 논문의 자세한 내용 정리에 더불어 내가 생각했던 아이디어까지 같이 비교, 정리해보려고 한다.

<br>

## **Online, Lifelong MAPF란?**

일단 Online, Lifelong MAPF(이하 Online MAPF)에 대해서 간단하게 짚고 넘어가자. 이때까지의 기존 논문들은 보통 one shot task, 즉 주어진 입력에 대해 솔루션을 구하는 문제에 집중하는 경우가 많았다. 하지만 아래 그림과 같이 LSC(logistics sorting center)같은 경우, 로봇이 한번 목적지에 도착한다고 끝나지 않고, 반복적으로 다시 새로운 task를 수행하기 위해 새로운 goal을 받게 된다.

![실제 물류 자동화 창고](/assets/img/LSC.jpg)
![물류 자동화 창고 추상도](/assets/img/LSC_abstracted.png)

이렇게 실제 운용환경에서는 경로탐색시, 새로운 태스크에 의한 새로운 에이전트의 추가가 지속적으로 발생하고, 이러한 상황에서 모든 에이전트들의 경로를 최적으로 유지할 필요가 있다.

<br>

# **Dynamic MAPF**

우선 문제 정의부터 해보자. 일단 일반적인 MAPF에서의 문제는 [이전 포스트](https://reofard.github.io/path_finding/2023/04/22/MAPF%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%BC%EA%B9%8C.html)에서 다뤘기 때문에 바로 Dynamic MAPF(이하 DMAPF)의 문제를 확인해보자. 동적인 환경에서는 언제든지 새로운 에이전트가 추가될 수 있다. DMAPF문제는 **기존의 에이전트 집합에 추가된 에이전트 집합을 포함하여 최적의 경로를 계산** 할 수 있어야 한다. 먼저 에이전트 집합 $A$에 대한 최적 경로 솔루션을 $Sol_A$라고 하자. $A$가 $Sol_A$를 바탕으로 작업을 수행하는 도중에 새로운 에이전트 집합 $A'$이 추가되면, DMAPF에서는 $A$와 $A'$의 현재상태를 기반으로 다시 최적의 경로 $Sol_{A \cup A'}$를 계산 할 수 있어야 한다.

![DMAPF](/assets/img/DMAPF.png)

이 내용은 위 내용을 통해 예시를 들어볼 수 있다. 위 그림에서 1,2번 에이전트는 $A$에 속한다. 반대로 3,4번 에이전트는 새롭게 추가된 에이전트로 $A'$에 속한다.1번과 2번 에이전트는 3, 4번 에이전트가 추가된 시점에 이미 검은 실선 경로를 진행한 뒤이고, 점선은 현재 시점에서 1,2,3,4번 에이전트를 모두 고려하여 계산한 최적의 경로가 된다.

<br>

# **Lifelong Planning Conflict-Based Search**

해당 논문에서는 DMAPF라는 문제로 수식화 하고, 문제를 해결하기 위해 LPCBS(Lifelong Planning Conflict-Based Search)를 제안한다. 제안된 LPCBS는 반복적으로 CBS연산을 진행하고, 이때 이전 사이틀에서 CBS 연산정보인 CT(Constraint Tree)를 기반으로 다음 CBS연산을 수행하여 연산시간, 메모리를 절약할 수 있다고 한다. MAPF 연산은 기본적으로 NP-Hard문제로 여겨질만큼 연산 코스트가 높기 때문에 이러한 방법으로 효과적으로 연산 코스트를 낮출 수 있다면 실용성이 많이 높아질 것 같다.

## **추가 정의**

LPCBS에서는 기존의 [CBS](https://reofard.github.io/path_finding/2023/05/20/CBS-MAPF%EA%B3%84%EC%9D%98-%EC%84%B1%EA%B2%BD.html)에서 최적성을 증명하기 위한 전제조건에서 더 확장해서 선행 전제조건을 정의하여 DMAPF의 문제에서 최적성을 유지한다. 한번 자세히 알아보자.

### **Definition 1**

> 이전 DMAPF의 계산결과중 i번째 agent의 경로는 $l_i$와 OPEN리스트의 특정 노드 $N$이 있을 때 다음과 같은 조건을 만족하면 $time-consistent$라고 한다.
> $$
> N.solution^i(t_{curr}) = l_i(t_{curr}) 
> $$
> 이때 시간 $t_{curr}$을 기준으로 low-level search를 진행하면 경로의 시작 위치는 항상 $l_i(t_{curr})$으로 고정되기 때문에 $time-consistent$를 전개해도, $time-consistent$한 노드가 생성이 된다.

### **Definition 2**

> 모든 허용가능한 솔루션 $p$에 대해 OPEN list에 $p$를 허용하는 노드가 존재하면, OPEN list를 Constraints-Completed하다고 한다. 해당 명제는 CBS를 진행할 때 항상 만족하게 된다.

이 전제조건을 기반으로 LPCBS가 어떻게 추가된 agent를 이용하여 최적의 해를 계산하는지 알아보자.

<br>

## **문제점과 증명**

LPCBS의 기본 아이디어는 현재 timestep에서 새로 추가된 에이전트의 경로를 계획하는 동시에 이전에 수행한 CBS의 정보를 기반으로 실행 중인 에이전트의 경로를 조정하는 것이다. 이전 사이클의 정보를 상속받기위해서는 두가지 문제점이 있다.

1. Outdated Node: 노드 $N$이 $time-consistent$한 경우, 이를 Outdated 노드라고 한다. 이 노드는 전개되더라도 현재 상황에 맞지 않는 경로만 생성되기 때문에 사용할 수 없다.
2. Outdated Constraint: 특정 노드 $N$에 포함된 Constraint 중 일부가 Timestep이 $t_{curr}$보다 앞선 경우가 있다. 이러한 제약조건들은 이후 경로탐색에 전혀 필요가 없다.

위의 문제가 해결되더라도, 과연 LPCBS가 실제로 최적성을 보장하는가에 대한 논의가 필요하다. 기존에 CBS는 하나의 루트 노드에서 시작되면 최적의 솔루션을 반환할 수 있다는 것을 증명되어있다. 하지만 LPCBS는 이전사이클의 OPEN list를 상속받아 검색을 시작하기 때문에 다시 증명이 필요하다. 상속받은 Node들과 주어진 timestep부터 최적의 경로를 뽑을 수 있다는 것을 증명해보자.

### **Lemma 1**: time-consistent한 노드만 갖고, constraints-completed한 OPEN-list를 기반으로 항상 CBS는 최적의 해를 반환한다.

> Best-First Search를 사용하므로, 가장 낮은 비용을 가진 노드가 항상 CBS 로직에서 반환된다. 예를 들어, 반환된 솔루션이 노드 $N_s$ 에서 나왔다면 항상 아래와 같은 식을 만족한다.
>$$
\forall N \in OPEN, N_s.cost \le N.cost
>$$
>여기서 Constraints-Completed 속성 덕분에 모든 유효한 솔루션 $p$에 대해 허용하는 노드 $N$이 존재한다. 이는 	$\forall N \in OPEN$일때 $\forall p \in CV(N)$을 항상 만족하여 모든 솔루션 $p$가 OPEN list에 의해 허용된다. 이때 **Definition 1**에 의해 모든 솔루션 $p$는 $time−consistent$한 노드에 의해 허용되기 때문에 아래와 같은 식을 만족한다.
>$$
\forall p \in N_s.solution, \forall a_i \in A, p^i(t_{curr}) = l_i(t_{curr})
>$$
>그렇기 때문에 반환된 최적의 해는 $t_{curr}$에서 항상 모든 에이전트들이 따를 수 있는 유효하고, 최적의 해를 반환한다.

<br>

### **Eliminating Time-Inconsistency of Outdated Nodes**

**Lemma 1**은 계산의 기반이 되는 OPEN list가 constraints-completed하고, 모든 구성요소가 time-consistent하다는 조건이 있어야 하는데, 이전 CBS의 OPEN list는 $time-consistent$하지 않은 노드 $\bar{N}$을 갖는다. 모든 $\bar{N}$에 대해 어떻게 처리하여 OPEN list의 constraints-completed함을 유지하는지 한번 확인해보자.

![Time-Inconsistent Node](/assets/img/Time-Inconsistent.png)

위의 그림에서 빨간 선은 이전 MAPF의 솔루션노드 $N_s.solution$이라고 하자. 그리고 OPEN list의 또다른 노드를 $\bar{N}$이라고 하고, $\bar{N}.solution$을 파란선이라고 해보자. $N_s.solution$을 진행하던중 timestep이 2일때 새로운 agent가 추가되어 다시 LPCBS를 진행한다고 하자. 이때 $N_s.solutionl_1(t_{curr}) \ne \bar{N}.solution_1(t_{curr})$<sup>각각 (1,3),  (2,2)이다</sup> 즉, $time-consistent$가 OPEN에 존재할 수 있다.

$time-inconsistent$한 노드 $\bar{N}$를 해결하기 위해 LPCBS에서는 경로를 조정한다. 예를 들어 위 그림의 경우 2번 에이전트의 경로는 $time-consistent$하다. 그렇기 때문에 그냥 경로에서 현재위치를 기준으로 앞부분을 자르기만 하면 된다. 하지만 1번 에이전트의 경로는 $time-inconsistent$하기 때문에 현재 위치부터 목적지까지 경로를 Low-level search를 통해 재계획하여 해당 노드를 $time-consistent$하게 만든다. 특정 노드 $N$에 대해 $time-consistent$하게 재가공하는 방법은 아래 그림과 같다.

![time-consistent logic](/assets/img/Time-consistent%20algorithm.png)

<br>

### **Eliminating Outdated Constraints**

Outdated Constraint에 대해 다루기 전에 해당 논문에서 제안한 Constraint tree에 대해 잠깐 언급하고 가자. 사실 회사에서 CBS를 자체개발 할 때 사이클이 없는 트리의 특성을 이용해서 메모리를 아꼈던 방식인데, 여기서 보니까 반가웠다. LPCBS에서는 CT Node가 각 노드가 필요한 constraint를 모두 갖고있지 않는다. 대신 부모노드에서 발생한 충돌에서 나온 constraint만 갖는데, 실제 필요한 constraint 집합을 계산할때는 상위 노드로 이동하면서 조상노드의 제약조건을 갖고와 사용한다.

![lifelong Constraint Tree](/assets/img/Lifelong%20ConstraintTree.png)

예를들어 위 그림에서 CT Node인 $\{c0, c1, ...\}$는 각각 부모노드에서 발생한 충돌을 해결하기위한 제약조건만을 갖고, $N_{c3}$등의 리프노드에서 제약조건을 갖고오기위해선 $\{c3, c1, c0\}$로 트리를 거슬러 올라가며 제약조건 집합을 완성한다.

그렇다면 이러한 CT에서 어떻게 $constraints-completed$속성을 유지하며 $t_{curr}$시점 이전에 걸린 제약조건을 제거하는지 확인해보자.
<br>

## **후기**

내용
