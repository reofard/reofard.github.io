---
title:  "Anytime Lifelong Multi-Agent Pathfinding in Topological Maps : 경로탐색 도메인을 바꿔보자"
excerpt: "Robot"

categories:
  - Path_Finding

tags:
  - [MAPF, Path Finding, Multi-Robot]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2024-12-07
last_modified_at: 2024-12-07
---

# **Anytime Lifelong Multi-Agent Pathfinding in Topological Maps**

오늘도 MAPF논문을 찾아 헤매던 중, 저자의 이름이 이상한 MAPF논문을 보게되었다. 한국에서는 MAPF 분야가 잘 활성화 되어있지 않다고 생각 했는데 한국인 저자가 작성한 논문은 처음봐서 신기한 마음에 한번 논문을 읽어보았다. 최근에 공개되어 open acccess된 논문이었는데, 내가 평소에 고민하던 부분인, "어떻게해야 더 복잡한 환경에서도 MAPF를 적용할 수 있을까" 에 관련한 내용이라 더 신기했던것 같다.

![kiva 시스템](/assets/img/KIVA.jpg)

기존의 MAPF알고리즘같은 경우 위 그림과 같이 보통 KIVA 시스템과 같이 그리드맵, 단위시간 도메인을 기반으로 연구되었었다. 하지만 이러한 방법들은 Kiva 로봇을 위한 표준화된 그리드 맵과 실제 창고에 로봇이 인식가능한 QR등의 설치등이 필요하다. 회사에 돈이 많다면 처음부터 물류센터를 이렇게 로봇을 고려하여 지으면 되지만, 이러한 시스템을 앞서 정리한 요구조건이 고려되지 않은 일반적인 창고 공간이나 AMR과 같은 다양한 유형의 이동 로봇에 적용하기는 어려운것이 현실이다.

해당 논문은 Topological Map을 기반으로 Lifelong MAPF에 대해 다룬다. 그중에서도 Topological Map 도메인으로 옮겨오면서 발생하는 가장 큰 문제중 하나인, 좁은 협소로를 어떻게 다루어 효율적으로 경로를 계산할 수 있을지에 대해 중점적으로 다룬다. 나도 AMR을 효율적으로 사용할 수 있는 MAPF알고리즘을 개발하기 위해서 전 직장인 Navifra에서 polygon 충돌, topological map등에 기반한 Conflict based Search 알고리즘을 구상하여 개발했었던 경험이 있는데, 이러한 경험에 기대어 현실적인 MAPF를 어떻게 만들 수 있을지 생각해보려고 한다.

<br>

# **lifelong MAPF in a topological map**

그렇다면 저자가 정의한 lifelong MAPF in a topological map의 문제 정의에 대해 한번 엄밀히 짚고 넘어가보자. 우선 topological map은 정점 $\mathcal{v} \in V_M$과 간선 $\mathcal{e} \in E_M$으로 표현되는 단방향 그래프 $\mathcal{G}_M = (V_M, E_M)$로 표현이 된다. 여기까지는 보통의 단방향 그래프를 나타내는 듯 하지만, 이 논문에서는 Warehouse에서의 MAPF문제를 해결하기 위해 아래 그림과 같이 협소로가 많은 그래프를 문제 도메인으로 삼는다.

![협소로 그림](/assets/img/narrow_corridor.png)

이번 논문에서 MAPF instance는 topological map $\mathcal{G}_M$와 agent의 집합 $\mathcal{A}=\{\mathcal{a}_1, ... , \mathcal{a}_n\}$으로 이루어 진다. 이때 $a_i \in \mathcal{A}$는 각각의 고유한 시작노드와 종료노드 $s_i \in V_\mathcal{M}$, $e_i \in E_\mathcal{M}$을 가진다. 이 외의 가정은 거의 대부분 기존의 DMAPF instance와 같다.

## **Corridor Conflict**

MAPF문제를 효율적으로 해결하기 위해서 이번논문에서는 Corridor Conflict를 중점적으로 다룬다. **Corridor Conflict(복도 충돌)**은 **두 에이전트가 서로 반대 방향으로 복도를 동시에 지나려고 할 때 발생하는 충돌**을 의미한다. 이를 해결하려면 한 에이전트가 다른 에이전트가 복도를 완전히 통과할 때까지 기다려야 하는데, 이는 MAPF 문제를 크게 복잡하게 만든다. 따라서, 이번 논문에서는 Corridor Conflict의 감지 및 효율적인 해결에 대해 중점적으로 다룬다고 한다.

복도 $C$는 두 개의 끝점 $(v_{q1}), (v_{q2})$와, 그 사이의 연결된 정점 집합 $\overline{C} = \{v_{c1}, \dots, v_{cL}\} \subseteq V_\mathcal{M}$으로 구성된다. 복도 길이 $\text{len}(C)$는 끝점 $v_{q1}$와 $v_{q2}$ 사이의 거리, 즉 두 정점을 이동하는 데 걸리는 시간으로 정의하고 아래 식과 같이 나타 낼 수 있고, $|\overline{C}|$는 $\overline{C}$내의 정점의 수가 된다.

$$
\text{len}(C) = |\overline{C}| + 1
$$

이러한 복도 정보는 사전에 저장되어 경로를 계획할 때 복도 충돌을 감지하는 데 직접적으로 활용하고, 이를 통해 MAPF 문제를 보다 효과적으로 해결할 수 있다고 한다.

<br>

# **Anytime-RHCR**

해당 논문에서는 Lifelong MAPF(Multi-Agent Path Finding) 문제를 해결하기 위해 RHCR(Rolling Horizon Collision Resolution) 방식을 한다. 하지만 협소로가 많은 Graph에서는 RHCR을 적용 하기에는 적합하지 않다. 시간 구간 $\omega$가 작을 경우 긴 복도에서 **교착 상태(deadlock)**가 발생할 가능성이 높아지기 때문이다. $\omega$를 늘리면 이러한 문제를 완화할 수 있지만, 계산량이 크게 증가하여 실시간 처리에 문제가 생기게 된다.

이러한 문제를 해결하기 위해 해당 논문에서는 Anytime-RHCR를 제안한다. 이 알고리즘은 초기 계획 단계에서는 ECBS를 사용하여 짧은 시간 구간 $\omega_{init}$에서 초기 경로를 생성한다. 이후 특정 에이전트 그룹을 선택하여, 최적 해법인 Corridor-CBS를 통해 확장된 시간 구간 $\omega_{extd}$에서 경로를 다시 개선한다.

## Pseudo Code

아래 그림은 Anytime-RHCR 알고리즘의 의사코드이다. 

![협소로 그림](/assets/img/AnytimeRHCR.png)

Anytime-RHCR 알고리즘의 주요 흐름은 다음과 같다.

    1. 현재 작업을 완료한 Agent에게 미완료된 작업을 할당
    2. 모든 Agent가 할당된 작업을 실행할 수 있도록 경로를 $\lambda$ 단위의 정기적인 시간 간격으로 반복적으로 계획
    3. 초기 경로는 작은 윈도우 $\omega_{init}$를 사용해 빠르게 생성
    4. 수정 집합 (modification set)의 하위 경로를 확장된 윈도우 $\omega_{extd}$​에서 개선

수정 집합은 $\omega_{extd}$기반으로 계산한 경로 집합 $P$에서 $\omega_{extd}$ 시간에서 발생하는 충돌에 연관된 에이전트들을 기반으로 결정된다. 이렇게 Anytime-RHCR은 기존의 RHCR보다 더 넓은 범위에서 충돌을 검출하고, 수정집합을 만들어서 경로를 개선하는 방식으로 협소로 문제를 해결한다.

이러한 과정을 통해 Anytime-RHCR은 초기 경로를 빠르게 생성하고, 중요한 충돌을 해결하며 전체 솔루션 품질을 지속적으로 향상시킬 수 있다고 한다.

## Selection on the Modification Set

내용

