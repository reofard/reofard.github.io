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

이번논문은 EECBS, RHCR의 저자이신 Jiaoyang Li님의 연구실 페이지를를 살펴보던중 발견한 논문이다. 내가 나비프라에서 로봇 관제 솔루션 업무를 진행 할때도 MAPF로 로봇을 돌릴 때, **로봇이 주어진 경로를 완벽하게 시간을 맞춰서 가는 경우가 더 드물었었다.** 그렇기 때문에 **MAPF 결과를 현실에 적용**할 수 있도록 여러가지 기법을 적용하여 해결을 했었던 기억이 있다. 그렇기 때문에 순수한 MAPF알고리즘을 실제 산업환경에 적용하기 위해서는 **경로 후처리 기법**이 더 중요한 경우도 많다고 생각해서 이번 논문에 더 흥미가 가고, 리뷰를 하게 되었다.

[기존의 연구](https://ojs.aaai.org/index.php/ICAPS/article/view/13796)에서는 Temporal Plan Graph(TPG)라는 개념을 이용하여 MAPF 인스턴스가 계획한 경로대로 로봇이 움직일 수 있도록 한다. 간단하게 설명하자면, 각 노드에 로봇이 도착하는 순서를 지정하고, 특정로봇이 해당 노드로 이동하기 전에 앞선 방문이 종료되었는지 확인하고, 이동을 수행한다. 이 방법은 MAPF에서 계산된 경로를 유지하는 방식이기 때문에 특정 로봇이 지연되는 경우, 해당 로봇이 앞으로 가야하는 노드를 더 늦은 순서로 도착하는 로봇들 모두 같이 경로가 지연되게 된다.

이 논문에서는 기존의 TPG에서 일부 방문순서를 변경 할 수 있는 [Switchable TPG](https://arxiv.org/abs/2010.05254)을 기반으로 재 스케줄링 결과를 최적화 할 수 있는 Switchable-Edge Search(SES)라는 알고리즘을 제안한다. 한번 어떤 방식으로 MAPF알고리즘의 결과를 현실에 적용할 수 있도록 후처리하는지 자세하게 알아보자.

<br>

# **Preliminaries**

가장 먼저 문제 정의부터 하고 넘어가보자. SES알고리즘은 주어진 MAPF Solution $\mathcal{P}$와 각 로봇의 지연시간을 기반으로 재스케줄링 하는 알고리즘이기 때문에, $\mathcal{P}$에 대한 정의만 다시 짚고 넘어가려고 한다. 나머지는 기존 mapf알고리즘과 conflict 정의, 이산시간과 같이 거의 비슷한거 같다. 우선 MAPF Solution은 충돌이 없는 경로의 집합 $\mathcal{P}$이다.

$$\mathcal{P} = \{ p_i : i \in \mathcal{A} \}$$

$\mathcal{A}$는 에이전트들의 집합, $p_i$는 특정 에이전트 $i$가 따라야하는 개별 경로이다. 이때 각 경로 $p\_i$는 i번째 agent의 위치와 timestep의 tuple의 리스트 $(l^i_0, l^i_0) \rightarrow (l^i_1, l^i_1) \rightarrow ... \rightarrow (l^i_{zi}, l^i_{zi})$로 이루어진다. 각 경로는 아래의 조건들을 만족해야 한다.

1. 연속적인 각 경로의 tuple에서의 시간은 항상 증가한다.
    > $0 = t^0_i < t^1_i < ... < t^{zi}_i$
2. 경로의 첫 번째 위치와 마지막 위치는 각각 에이전트$i$의 시작점과 목표점위치이다.
    > $l^0_i(start\ location), l^{zi}_i(goal\ location)$
1. 모든 연속된 두 tuple 쌍에 대해 항상 agent i는 단일 timestep에 $l^i_k$에서 $l^i_{k+1}$로 이동 가능하다.
    > $(l^i_k, t^i_k) \rightarrow (l^i_{k+1}, t^i_{k+1})\ is\ valid: all\ k > 0$ 

TPG를 선택하면서 통상적인 MAPF알고리즘과 차별화 되는 선행 개념이 하나가 존재한다. 바로 **불필요한 대기**를 생략하는 것이다. 하나의 agent가 특정경로의 일부 구간을 지난다고 가정해보면 총 두가지 케이스가 존재할 수 있다. (1) 구간의 중간에서 잠깐 멈췄다 가는경우, (2) 구간의 마지막에 도착해서 멈췄다 가능 경우. 두 케이스 해당 구간을 지나는데 걸리는 cost는 동일 할 수 있지만 TPG가 도입되면 **다음 구간을 시작하는 flag**가 MAPF Solution에서 계산된 시간이 아니라, 다음 구간을 **먼저 방문하는 로봇이 실제로 지났는지 여부**가 되기 때문에, **연속해서 지날 수 있는 특정 구간의 중간에서 멈추는것은 비효율적**이게 된다.

# **[Temporal Plan Graph(TPG)](https://ojs.aaai.org/index.php/ICAPS/article/view/13796)**

우선 간단하게 TPG에 대해 간단하게 알아보자. TPG는 뱡향성 그래프 $\mathcal{G} = (\mathcal{V}, \mathcal{E}\_1, \mathcal{E}\_2)$으로 나타내며 MAPF Solution $\mathcal{P}$에서의 우선순위 관계를 나타낸다.

<br>

# **후기**

후기