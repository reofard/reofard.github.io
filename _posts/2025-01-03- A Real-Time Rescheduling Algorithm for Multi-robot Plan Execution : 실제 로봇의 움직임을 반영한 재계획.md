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

# **[A Real-Time Rescheduling Algorithm for Multi-robot Plan Execution](https://ojs.aaai.org/index.php/ICAPS/article/view/31477)**
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

간단하게 위 로직에 대해서 설명하자면, 이미 로봇이 수행한 경로인 satisfied 상태의 정점은 reverse할 수 없기 때문에 step1에서는 현재 수행 정보에 따라 TPG의 정점과 간선을 분류한다. 그 이후 지연된 agent에 대해 더미 정점과 새로운 간선을 생성하여, 에이전트가 지연되는 시간 동안 가상의 경로를 만들고, 이를 통해 에이전트가 경로를 계속 진행할 수 있도록 한다. 추가된 더미 정점들은 지연 시간을 처리하는 데 필요한 "가상의 이동"을 수행하며, 간선 수정은 경로를 나타낸다.

이러한 과정을 통해 얻어진 $\mathcal{G}\_\mathcal{S}$는 모든 $\mathcal{S}\_{\mathcal{E}2}$의 요소에 $fix$연산을 수87행하면 $\mathcal{G}\_0$가 나오고, $\mathcal{G}\_0$는 MAPF의 솔루션을 기반으로 만들어진 TPG이기 때문에 교착이 존재하지 않는다. 즉 $\mathcal{G}\_\mathcal{S}$에서 파생 가능한 $\mathcal{G}^\mathcal{S} - producible\ TPG$중에서는 항상 교착이 없는 해가 존재함을 보장 할 수 있다.


## **Switchable Edge Search (SES) Framework**

SES(Switchable Edge Search)는 Heuristic Search 기반의 알고리즘으로, STPG 공간에서 A* 탐색을 통해 최적의 $\mathcal{G}^\mathcal{S} - producible\ TPG$를 찾는것이다. 이번 논문에서는 STPG 공간을 탐색하기 위해 Reduced TPG를 정의하여 $\mathcal{G}^\mathcal{S} - producible\ TPG$의 비용을 산출한다. 
$\mathcal{G}\_\mathcal{S} = (\mathcal{V}, \mathcal{E}\_1, (\mathcal{S}\_{\mathcal{E}2}, \mathcal{N}\_{\mathcal{E}2}))$가 있을 때, Reduced TPG는 아래 식과 같이 정의되고, $\mathcal{G}\_\mathcal{S}$의 partial cost는 Reduced TPG의 수행 시간이라고 정의한다.

$$
red(\mathcal{G}_\mathcal{S}) = (\mathcal{V}, \mathcal{E}_1,\mathcal{N}_{\mathcal{E}2})
$$

이때 $\mathcal{G}\_\mathcal{S}$ partial cost는 $\mathcal{G}\_\mathcal{S}$에서 파생될 수 있는 어떠한 $\mathcal{G}^\mathcal{S} - producible\ TPG$의 partial cost보다도 작거나 같다.

<br>

![SES pseudo code](/assets/img/SES_Algorithm.png)

위의 그림은 SES알고리즘의 수도코드를 나타낸 것이다. 간단하게 SES알고리즘의 동작 방식을 설명하면 다음과 같다.

1. Root STPG 생성
    - Construction 1을 사용하여 초기 STPG $\mathcal{G}^\mathcal{S}_{root}$를 생성. 이 초기 그래프는 사이클이 존재하지 않아야 하며, A* 탐색의 루트 노드가 된다.

2. 우선순위 큐 $\mathcal{Q}$를 통해 탐색할 노드 선정
    - $\mathcal{Q}$를 사용하여 STPG 노드를 $f$-value($g+h$)에 따라 정렬 및 선택
    - $h$-value는 $red(\mathcal{G}_\mathcal{S})$의 partial cost

3. 노드 확장
    - 선택된 STPG에서 하나의 switchable edge를 선택하여 fix(고정) 또는 reverse(반전) 수행하여 2가지 노드를 생성
    - 확장한 STPG에서 Cycle이 검출되면 해당 노드를 폐기한다.

사실 경로 정해놓고 누가먼저 갈지 정하는 과정을 Best-fit Search알고리즘으로 구하는 로직이라 그렇게 어렵지는 않다. 하지만 여기서 중요한 점은 pseudo code의 5번 라인인 것 같다. 분기할 노드가 정해졌을 때, 해당 $\mathcal{G}\_\mathcal{S}$의 switchable edge들 중 어떤 edge를 선택하여 $fix$, $reverse$연산을 수행하여 분기할 것인지에 대한 결정을 하는 부분이다. 이 과정에서는 $branch$라는 함수가 분기할 STPG $\mathcal{G}\_\mathcal{S}$와 보조정보 $\mathcal{X}$를 통해 정하게 된다. 또한 $Hueristic$을 결정하는 부분도 $h$-value만 정의되어있고, $g$-value는 정의되어있지 않은데, 이를 어떻게 구현하냐에 따라 달라질 수 있다.

<br>

## **Execution-based Switchable Edge Search(ESES)**

ESES는 실행 정보에 기반하여 구현된 SES알고리즘이다. 해당 알고리즘은 $BRANCH$, $HUERISTIC$ 함수를 실제 agent들의 경로 실행 정보에 기반하여 계산하게 된다. 아래는 ESES의 pseudo code이다.

![ESES algorithm](/assets/img/ESES_algorithm.png)

ESES는 앞서 설명한 것 처럼 실제 agent들의 경로 실행 정보에 기반하여 $\mathcal{X}$를 생성한다. 즉 TPG의 수행에 따라 $\mathcal{X}$에 가장 최근에 도착한 인덱스 정보(=$\mathcal{X}[i]$)가 저장되고, 이를 통해 $BRANCH$함수에서는 각 에이전트가 이동할 정점이  switchable edge에 연결되어있는지 확인한다. 만약 있다면 노드를 확장하고 $\mathcal{X}$를 업데이트 한다. 없다면 Agent의 작업수행을 대기한다.

또한 이 과정을 통해 $HUERISTIC$함수 또한 업데이트한다. ESES에서 $g$-value는 모든 에이전트가 현재 상태 $\mathcal{X}$에 도달하기까지 이동한 총 비용으로 정의가 되기 때문에 Agent의 작업 수행에 따라 함께 업데이트 된다. 아래의 그림은 ESES가 수행되는 과정을 간단히 설명한 도식도이다.

![ESES_EXEC](/assets/img/eses_execution.png)

우선 ①단계에서 수행중인 STPG의 j번째 agent의 다음 노드에 switchable edge(주황 실선)이 연결되어있기 때문에 $BRANCH$로직을 통해 $(v^i_3, v^j_1)$이 선택된다. 그리고 해당 edge를 $fix$, $reverse$연산을 수행하여 ②, ③번 노드를 확장한다. 그 이후 ②, ③중 $f$-value가 ③이 높기 때문에 선택된다.

③번 STPG를 수행하는중 agent가 각각 $v^i_1, v^j_1$에 도달한 뒤, 다시 i번째 agent의 다음노드에 switchable edge가 연결되어있고, $BRANCH$로직을 통해 $(v^i_3, v^j_2)$이 선택된다. 이를 통해 ④, ⑤노드를 확장하는데, 이때 ⑤는 cycle을 이루기 때문에 무시되어 ④번노드가 선택된다. 이때 STPG ②는 부분 비용(partial cost)이 이미 TPG ④의 비용보다 크므로 확장하지 않게 된다.

<br>

## **Graph-based Switchable Edge Search(GSES)**

GSES에서는 각 정점에 대한 최대 경로 $lp(v)$ 값을 기반으로, switchable edge를 고를 때 현재 경로의 길이가 역전된 Edge를 선택하는 알고리즘이다. 좀 말이 복잡한데 간단히 설명하면, 각 agent가 랜덤하게 지연되면, 특정 노드에 대한 도착 예정시간과 도착 선후관계의 불일치가 발생할 수 있는데, 이 경우에만 분기를 하겠다는 것이다. pseudo-code는 아래와 갖다.

![ESES_EXEC](/assets/img/GSES_algorithm.png)

우선 GSES에서는 Huristic 값에서 $g$-value를 사용하지 않는다. 대신 $\mathcal{X}$만을 이용해서 partial cost를 별도로 계산한다. 우선 $HUERISTIC$함수는 아래와 같은 로직을 갖는다.

1. $\mathcal{X}'[i] = lp(v^i_k), k \in [0, zi]$ on $red(\mathcal{G}\_\mathcal{S})$
2. $f-value = \sum\_{i \in \mathcal{A}} \mathcal{X}[v^i\_{zi}]$

간단하게 설명하면 1번 로직은 현재 선택된 STPG의 reduced-TPG에서 각각의 Agent가 경로상에서 도착하는데 걸리는 시간을 업데이트 하는것이다. 이 과정에서 각 agent가 delay된 정도가 업데이트되어, 현재 지연정보가 반영된 각 Agent별 정점 도착 예측시간을 알 수 있다. 2번로직은 해당 노드의 $f$-value를 계산하는 과정이다. GSES에서는 이때까지 걸린 시간이 아닌, 지연된 정보가 반영된 상태에서 각각의 agent가 목적지에 도착하는데 걸리는 시간의 합인 SoC함수를 목적함수로 사용한다.

GSES에서 $BRANCH$함수는 각 Agent가 특정 노드에 도착하는 시간인 $\mathcal{X}$를 기반으로 switchable-edge가 있을때 불필요하게 먼저도착해서 기다리게 하는 Edge를 찾는다. 이러한 조건을 만족하는 switchable-edge를 수식으로 표현하면 아래 식과 같다.

$$
(u, v) \in \mathcal{S}_{\mathcal{E}2} : \mathcal{X}[u] \geq \mathcal{X}[v]
$$

이렇게 GSES에서는 휴리스틱적으로 도착 예정 시간만을 업데이트하여 빠르게 휴리스틱적으로 SES알고리즘을 해결할 수 있다고 한다.


<br>

# **후기**
---
이번 논문은 계산 방법도 CBS보다 훨씬 간단하고, 기존의 Conflict based Search처럼 완벽하게 전체경로를 작성하지 않고 PIBT나 RHCR같이 현재 상황을 반영하여 주어진 경로를 일부만 수정하는 형식이라, 실제 내가 진행하고있는 Brown-Feild에서의 멀티로봇 관제 개발에서도 사용하기 좋을 것 같다. 이번 포스트에서는 간단히 넘어간 Graph-based SES(GSES)에서 PIBT의 우선순위 상속개념을 살짝 빌려와서 계산하면 더 빠르고 간결하게 문제를 해결할 수 있을것 같은데, 이건 이번 회사에서 한번 개발해보려고 한다.