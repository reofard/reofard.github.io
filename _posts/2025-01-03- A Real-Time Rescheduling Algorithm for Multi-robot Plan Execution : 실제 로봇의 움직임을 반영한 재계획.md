---
title: "A Real-Time Rescheduling Algorithm for Multi-robot Plan Execution : 실제 로봇의 움직임을 반영한 재계획"
excerpt: "Robot"

categories:
  - Path_Finding

tags:
  - [MAPF, Path Finding, Multi-Robot]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2025-01-03
last_modified_at: 2025-01-03
---

# **A Real-Time Rescheduling Algorithm for Multi-robot Plan Execution**
---
이번논문은 EECBS, RHCR의 저자이신 Jiaoyang Li님의 연구실 페이지를를 살펴보던중 발견한 논문이다. 내가 로봇 관제 솔루션 업무해오면서 MAPF로 로봇을 돌릴 때, **로봇이 주어진 경로를 완벽하게 시간을 맞춰서 가는 경우가 더 드물었었다.** 그렇기 때문에 **MAPF 결과를 현실에 적용**할 수 있도록 여러가지 기법을 적용하여 해결을 했었던 기억이 있다. 그렇기 때문에 순수한 MAPF알고리즘을 실제 산업환경에 적용하기 위해서는 **경로 후처리 기법**이 더 중요한 경우도 많다고 생각해서 이번 논문에 더 흥미가 가고, 리뷰를 하게 되었다.

[기존의 연구](https://ojs.aaai.org/index.php/ICAPS/article/view/13796)에서는 Temporal Plan Graph(TPG)라는 개념을 이용하여 MAPF 인스턴스가 계획한 경로대로 로봇이 움직일 수 있도록 한다. 간단하게 설명하자면, 각 노드에 로봇이 도착하는 순서를 지정하고, 특정로봇이 해당 노드로 이동하기 전에 앞선 방문이 종료되었는지 확인하고, 이동을 수행한다. 이 방법은 MAPF에서 계산된 경로를 유지하는 방식이기 때문에 특정 로봇이 지연되는 경우, 해당 로봇이 앞으로 가야하는 노드를 더 늦은 순서로 도착하는 로봇들 모두 같이 경로가 지연되게 된다.

이 논문에서는 기존의 TPG에서 일부 방문순서를 변경 할 수 있는 [Switchable TPG](https://arxiv.org/abs/2010.05254)을 기반으로 재 스케줄링 결과를 최적화 할 수 있는 Switchable-Edge Search(SES)라는 알고리즘을 제안한다. 한번 어떤 방식으로 MAPF알고리즘의 결과를 현실에 적용할 수 있도록 후처리하는지 자세하게 알아보자.

<br>

# **Preliminaries**
---
가장 먼저 문제 정의부터 하고 넘어가보자. SES알고리즘은 주어진 MAPF Solution $\mathcal{P}$와 각 로봇의 지연시간을 기반으로 재스케줄링 하는 알고리즘이기 때문에, $\mathcal{P}$에 대한 정의만 다시 짚고 넘어가려고 한다. 나머지는 기존 mapf알고리즘과 conflict 정의, 이산시간과 같이 거의 비슷한거 같다. 우선 MAPF Solution은 충돌이 없는 경로의 집합 $\mathcal{P}$이다.

$$\mathcal{P} = \{ p_i : i \in \mathcal{A} \}$$

$\mathcal{A}$는 에이전트들의 집합, $p_i$는 특정 에이전트 $i$가 따라야하는 개별 경로이다. 이때 각 경로 $p\_i$는 i번째 agent의 위치와 timestep의 tuple의 리스트 $(l^i_0, t^i_0) \rightarrow (l^i_1, t^i_1) \rightarrow ... \rightarrow (l^i_{zi}, t^i_{zi})$로 이루어진다. 각 경로는 아래의 조건들을 만족해야 한다.

1. 연속적인 각 경로의 tuple에서의 시간은 항상 증가한다.
    > $0 = t^0_i < t^1_i < ... < t^{zi}_i$
2. 경로의 첫 번째 위치와 마지막 위치는 각각 에이전트$i$의 시작점과 목표점위치이다.
    > $l^0_i(start\ location), l^{zi}_i(goal\ location)$
1. 모든 연속된 두 tuple 쌍에 대해 항상 agent i는 단일 timestep에 $l^i_k$에서 $l^i_{k+1}$로 이동 가능하다.
    > $(l^i_k, t^i_k) \rightarrow (l^i_{k+1}, t^i_{k+1})\ is\ valid: all\ k > 0$ 

TPG를 선택하면서 통상적인 MAPF알고리즘과 차별화 되는 선행 개념이 하나가 존재한다. 바로 **불필요한 대기**를 생략하는 것이다. 하나의 agent가 특정경로의 일부 구간을 지난다고 가정해보면 총 두가지 케이스가 존재할 수 있다. (1) 구간의 중간에서 잠깐 멈췄다 가는경우, (2) 구간의 마지막에 도착해서 멈췄다 가능 경우. 두 케이스 해당 구간을 지나는데 걸리는 cost는 동일 할 수 있지만 TPG가 도입되면 **다음 구간을 시작하는 flag**가 MAPF Solution에서 계산된 시간이 아니라, 다음 구간을 **먼저 방문하는 로봇이 실제로 지났는지 여부**가 되기 때문에, **연속해서 지날 수 있는 특정 구간의 중간에서 멈추는것은 비효율적**이게 된다.

# **[Temporal Plan Graph(TPG)](https://ojs.aaai.org/index.php/ICAPS/article/view/13796)**
---
우선 간단하게 TPG에 대해 간단하게 알아보자. TPG는 뱡향성 그래프 $\mathcal{G} = (\mathcal{V}, \mathcal{E}\_1, \mathcal{E}\_2)$으로 나타내며 MAPF Solution $\mathcal{P}$에서의 우선순위 관계를 나타낸다. 우선 정점의 집합 $\mathcal{V} = \{ v^i\_k : i \in \mathcal{A}, k \in [0, zi] \}$에서 각각의 정점 $v^i\_k$는 $i$번째 agent가 수행하는 $k$번째 이동을 나타낸다. 즉 특정 에이전트가 특정 시간에 특정 위치에서 수행하는 이동이다. 간선 집합 $\mathcal{E}\_1, \mathcal{E}\_2$에서 각각의 간선 $(u, v)$는 각 정점간의 선후관계를 나타낸다. 아래 그림은 실제 두 로봇의 움직임에 대한 MAPF결과를 TPG로 표현 한 것이다.

![TPG 예시](/assets/img/TPG.png)

우선 $\mathcal{E}\_1$이 나타내는 정보에 대해서 알아보자. $\mathcal{E}\_1$의 간선은 실선을 나타낸다. 위 그림에서 우선 빨간로봇은 순차적으로 $A \rightarrow B \rightarrow D \rightarrow F$순으로 움직어야 한다. 이때 $A \rightarrow B $, $B \rightarrow D $, $D \rightarrow F$ 움직임은 각각 $v^{red}\_1$, $v^{red}\_2$, $v^{red}\_3$으로 나타내진다. 이때 각 움직임은 반드시 순차적으로 행해져야만 하고, 이를 $(v^{red}\_1, v^{red}\_2)$, $(v^{red}\_2, v^{red}\_3)$  같이 간선으로 표현할 수 있다. 이렇게 하나의 Agent가 수행해야 하는 움직임들이 있을때 해당 움직임들에 대해 선후관계를 $\mathcal{E}\_1$으로 나타낸다.

$\mathcal{E}\_1$는 서로 다른 Agent의 움직임 사이의 선후관계를 나타낸다. $v^{red}\_3$와 $v^{blue}\_2$은 각각 빨간 agent가 $D \rightarrow F$로 움직이는 이동과 파란 agent가 $C \rightarrow D$로 움직이는 이동이다. 이때 파란색 Agent는 충돌이나 교착을 피하기 위해 항상 빨간색 Agent이 지나가고 난 뒤에 D로 이동해야한다. 즉 $v^{blue}\_2$는 항상 $v^{red}\_3$가 종료되고 난 뒤에 수행되어야 한다는 것 이다. 이러한 정보는 마찬가지로 $(v^{red}\_3, v^{blue}\_2)$와 같이 간선으로 나타낼 수 있고, 두개 Agent의 움직임 사이에 선후관계가 있을때 이러한 관계를 $\mathcal{E}\_2$으로 나타낸다.

## **Executing a TPG**

각 정점(vertex)은 두 가지 상태("satisfied" 또는 "unsatisfied") 중 하나를 가진다. 특정 정점이 "satisfied" 상태가 된다는 것은 해당 위치에 에이전트가 도착했음을 의미하며, 이 상태로 변경되려면 모든 in-neighbor(선행 정점)들이 먼저 "satisfied" 상태여야 한다. 이때 TPG를 기반으로 경로를 Agent에 수행하게 하는 것은 간단하다. 순차적으로 "satisfied"로 전환하기 전에 각 agent에 해당 움직임을 명령하고, 종료되면 해당 정점을 "satisfied"로 변환하는것을 반복하면 된다. 그렇게 하여 모든 정점이 "satisfied" 상태가 되면 실행이 종료된다. 이때 TPG내부에 Cycle이 존재하면, "unsatisfied" 상태의 되지 않은 정점이 존재하지만 새롭게 "satisfied" 상태가 될 정점이 없는 Deadlock이 발생할 수 있다.

이때 MAPF 솔루션에서 생성된 TPG는 항상 사이클이 없음을 보장하고, Deadlock이 발생하지 않는다.

<br>

# **[Switchable TPG(STPG)](https://arxiv.org/abs/2010.05254)**
---
기존의 TPG는 특정 MAPF 솔루션에 고정된 형태로 구성되므로, 경로가 고정되며 유연한 수정이 어렵다. 이를 해결하기 위해 Switchable TPG (STPG)에서는 우선순위 관계를 동적으로 조정할 수 있게 기능을 확장하였다. 주어진 TPG의 그래프 $\mathcal{G} = (\mathcal{V}, \mathcal{E}\_1, \mathcal{E}\_2)$에서,
Switchable TPG는 아래 식과 같이 $\mathcal{E}\_2$를 두개의 서로소 집합 $\mathcal{SE}\_2, \mathcal{NE}\_2$로 나눈다. 이때 $\mathcal{S}\_{\mathcal{E}2}, \mathcal{N}\_{\mathcal{E}2}$는 각각 변경 가능한 간선집합과 변경 불가능한 간선집합을 나타낸다.

$$\mathcal{G}^S=(\mathcal{V}, \mathcal{E}_1, (\mathcal{S}_{\mathcal{E}2}, \mathcal{N}_{\mathcal{E}2}))$$

처음 TPG를 기반으로 STPG를 구성할 때는 모든 $\mathcal{E}\_2$의 요소를 $\mathcal{S}\_{\mathcal{E}2}$에 넣고, $\mathcal{N}\_{\mathcal{E}2}$는 공집합으로 구성한다.

## **Operation of STPG**

STPG는 아래와 같은 두가지 연산($fix(v_{s+1}^j, v_k^i)$, $reverse(v_{s+1}^j, v_k^i)$)를 통해 각 agent사이에서 움직임 우선순위를 조정할 수 있다.

1. $fix(v_{s+1}^j, v_k^i)$
    > $(v_{s+1}^j, v_k^i)$를 $\mathcal{S}\_{\mathcal{E}2}$에서 제거하고, $\mathcal{N}\_{\mathcal{E}2}$에 추가한다.

2. $reverse(v_{s+1}^j, v_k^i)$
    > $(v_{s+1}^j, v_k^i)$를 $\mathcal{S}\_{\mathcal{E}2}$에서 제거하고, $\mathcal{N}\_{\mathcal{E}2}$에 $(v^i\_{k+1}, v^k\_s)$을 추가한다.

$reverse$ 연산이 좀 난해한데, 간단히 설명하면 아래그림과 같다. 먼저 그림의 왼쪽에서 검은 실선이 $\mathcal{S}\_{\mathcal{E}2}$의 요소라고 해보자. 해당 간선은 빨간 agent가 먼저 움직이고 난 뒤에 파란 agent가 이후에 지나가는 선후관계를 나타낸다. 이때 빨간 Agent가 어떠한 움직임 $v^{red}\_1$의 수행이 지연되어 파란색 agent를 먼저 보내는게 효율적인 상황이라고 가정해보자.

![reverse operation](/assets/img/stpg_reverse_operation.png)

이때 아래의 그림과 같이 빨간색 agent은 맵에서의 정점을 $l^{red}\_0 \rightarrow l^{red}\_1 \rightarrow l^{red}\_2 \rightarrow l^{red}\_3$의 순서로 지나고, 파란색 agent는  정점을 $l^{blue}\_0 \rightarrow l^{blue}\_1 \rightarrow l^{blue}\_2 \rightarrow l^{blue}\_3$ 순서로 지나게 되고, $l^{red}\_0$와 $l^{blue}\_2$는 동일한 위치라고 볼 수 있다. 즉 $reverse$연산의 대상인 $\mathcal{S}\_{\mathcal{E}2}$의 요소 $(v_{1}^{red}, v_2^{blue})$는 빨간색 agent가 $l^{red}\_0 = l^{blue}\_2$를 파란 agent보다 먼저 지나가게 하는 제약 조건이라고 볼 수 있다. 이를 파란 agent가 먼저 지나가게 하려면, 파란 agent가 $l^{blue}\_2 \rightarrow l^{blue}\_3$움직임을 먼저 수행하고, 빨간 agent가 $l^{red}\_? \rightarrow l^{red}\_0$을 이후에 수행해야 하기 때문에 $(v\_{1}^{red}, v\_2^{blue})$가 $(v\_{3}^{blue}, v\_0^{red})$로 변경되는 것이다.

![reverse operation example](/assets/img/stpg_reverse_op_ex.png)

결과적으로 각각의 연산은 실제 수행에 앞서 우선순위 관계를 변경 불가능한 상태, 즉 실제 경로를 수행함에 앞서 STPG의 그래프에서 TPG의 그래프로 다시 변형하는 작업이라고 할 수 있다. 이때 경로 수행 delay에 따라 각 노드에 도착하는 우선순위를 변경하여 고정된 TPG보다 더 효율적으로 현실의 정보를 반영하여 경로를 변경할 수 있게 된다.

이후 설명할 Switchable Edge Search알고리즘에서는 주어진 TPG를 통해 STPG를 만들고, 이를 $fix$, $reverse$연산을 통해 만들수 있는 TPG중 가장 코스트가 낮은 TPG를 탐색함으로써 경로를 재계획한다.

# **Switchable Edge Search(SES)**
---
먼저 Switchable Edge Search(SES)를 설명하기 전에 정의해야하는게 있다. 논문에서는 STPG 부분에서 설명이되어있던데, 내용상 여기가 맞는거같아서 내용의 순서를 좀 임의로 변경했다. STPG $\mathcal{G}\_\mathcal{S}$가 주어졌을때 $fix$, $reverse$연산들을 순차적으로 수행하여 얻을 수 있는 TPG를 $\mathcal{G}^\mathcal{S} - producible\ TPG$라고 한다.

<br>

이제 이번 논문의 핵심 Switchable Edge Search(SES)에 대해 설명할 차례가 왔다. SES는 주어진 MAPF의 solution $\mathcal{P}$에 대해, $\mathcal{G}\_0$라는 TPG를 구성한다. 이후 TPG를 수행하는 중 delay가 발생하면 TPG를기반으로 STPG $\mathcal{G}\_\mathcal{S}$를 구성하고, 이를 기반으로 생성할 수 있는 모든 $\mathcal{G}^\mathcal{S} - producible\ TPG$중 가장 코스트가 낮은 TPG를 찾아 반환하는 알고리즘이다.

## **Initialize of SES**

먼저 수행중인 $\mathcal{G}\_0 = (\mathcal{V}, \mathcal{E}\_1, \mathcal{E}\_{2})$로 부터 STPG $\mathcal{G}\_\mathcal{S}$를 구축하는 방법부터 정리해보자. STPG $\mathcal{G}\_\mathcal{S}$는 아래와 같은 두가지 step을 통해 생성된다.

1. $\mathcal{G}\_\mathcal{S} = (\mathcal{V}, \mathcal{E}\_1, (\mathcal{S}\_{\mathcal{E}2}, \mathcal{N}\_{\mathcal{E}2}))$로 설정하고, 여기서
    - $\mathcal{S}\_{\mathcal{E}2} = \{ (v^{i}\_{ki}, v^{j}\_{kj}) \in \mathcal{E}\_2 : v^{i}\_{ki}\ is\ not\ satisfied \}$
    - $\mathcal{N}\_{\mathcal{E}2} = \{ (v^{i}\_{ki}, v^{j}\_{kj}) \in \mathcal{E}\_2 : v^{i}\_{ki}\ is\ satisfied \}$로 정의
2. 특정 $c$번째 agent가 현재 위치 $l^c_{d}$에서 $l^c_{d+1}$로 갈때 $\Delta$시간 만큼 지연이 예상될 때
    - 새로운 더미 정점 $\mathcal{V}\_{new} = \{v\_1, v\_1, ..., v\_\Delta \}$ 생성
    - 새로운 더미 간선 $\mathcal{E}\_{new} = \{(v^c_d, v_1), (v\_1, v\_2), ..., (v\_\Delta, v^c_{d+1}) \}$ 생성
    - $\mathcal{V} \leftarrow \mathcal{V} \cup \mathcal{V}\_{new}$,  $\mathcal{E}\_1 \leftarrow (\mathcal{E}\_1 \cup \mathcal{E}\_{new}) - \{(v^c\_d, v^c\_{d+1})\}$ 

간단하게 위 로직에 대해서 설명하자면, 이미 satisfied 상태의 정점은 이미 로봇이 수행한 경로기 때문에, reverse할 수 없기 때문에 step1에서는 현재 수행 정보에 따라 TPG의 정점과 간선을 분류한다. 그 이후 지연된 agent에 대해 더미 정점과 새로운 간선을 생성하여, 에이전트가 지연되는 시간 동안 가상의 경로를 만들고, 이를 통해 에이전트가 경로를 계속 진행할 수 있도록 한다. 추가된 더미 정점들은 지연 시간을 처리하는 데 필요한 "가상의 이동"을 수행하며, 간선 수정은 경로를 나타낸다.

이러한 과정을 통해 얻어진 $\mathcal{G}\_\mathcal{S}$는 모든 $\mathcal{S}\_{\mathcal{E}2}$의 요소에 $fix$연산을 수행하면 $\mathcal{G}\_0$가 나오고, $\mathcal{G}\_0$는 MAPF의 솔루션을 기반으로 만들어진 TPG이기 때문에 교착이 존재하지 않는다. 즉 $\mathcal{G}\_\mathcal{S}$에서 파생 가능한 $\mathcal{G}^\mathcal{S} - producible\ TPG$중에서는 항상 교착이 없는 해가 존재함을 보장 할 수 있다.


## **Initialize of SES**

<br>

# **후기**
---
후기