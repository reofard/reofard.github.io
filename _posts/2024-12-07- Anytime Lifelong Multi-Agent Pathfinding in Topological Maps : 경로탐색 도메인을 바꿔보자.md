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

MAPF문제를 효율적으로 해결하기 위해서 이번논문에서는 Corridor Conflict를 중점적으로 다룬다. **Corridor Conflict**은 **두 에이전트가 서로 반대 방향으로 복도를 동시에 지나려고 할 때 발생하는 충돌**을 의미한다. 이를 해결하려면 한 에이전트가 다른 에이전트가 복도를 완전히 통과할 때까지 기다려야 하는데, 이는 MAPF 문제를 크게 복잡하게 만든다. 따라서, 이번 논문에서는 Corridor Conflict의 감지 및 효율적인 해결에 대해 중점적으로 다룬다고 한다.

복도 $C$는 두 개의 끝점 ${(v_{q1}), (v_{q2})}$와, 그 사이의 연결된 정점 집합 $\overline{C} = \{v_{c1}, \dots, v_{cL}\} \subseteq V_\mathcal{M}$으로 구성된다. 복도 길이 ${\text{len}(C)}$는 끝점 $v_{q1}$와 $v_{q2}$ 사이의 거리, 즉 두 정점을 이동하는 데 걸리는 시간으로 정의하고 아래 식과 같이 나타 낼 수 있고, $|\overline{C}|$는 $\overline{C}$내의 정점의 수가 된다.

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

![AnytimeRHCR](/assets/img/AnytimeRHCR.png)

Anytime-RHCR 알고리즘의 주요 흐름은 다음과 같다.

  1. 현재 작업을 완료한 Agent에게 미완료된 작업을 할당
  2. 모든 Agent가 할당된 작업을 실행할 수 있도록 경로를 $\lambda$ 단위의 정기적인 시간 간격으로 반복적으로 계획
  3. 초기 경로는 작은 윈도우 $\omega_{init}$를 사용해 빠르게 생성
  4. 수정 집합 (modification set)의 하위 경로를 확장된 윈도우 $\omega_{extd}$​에서 개선

수정 집합은 $\omega_{extd}$기반으로 계산한 경로 집합 $P$에서 $\omega_{extd}$ 시간에서 발생하는 충돌에 연관된 에이전트들을 기반으로 결정된다. 이렇게 Anytime-RHCR은 기존의 RHCR보다 더 넓은 범위에서 충돌을 검출하고, 수정집합을 만들어서 경로를 개선하는 방식으로 협소로 문제를 해결한다.

이러한 과정을 통해 Anytime-RHCR은 초기 경로를 빠르게 생성하고, 중요한 충돌을 해결하며 전체 솔루션 품질을 지속적으로 향상시킬 수 있다고 한다.

<br>

## Selection on the Modification Set

앞서 설명한 알고리즘 중 수정 집합 $\mathcal{A}_M$을 결정하는 것 솔루션의 퀄리티나 계산 속도를 구하는데, 큰 영향을 미친다. 이는 앞서서 [ICTS에 다룬 포스트](https://reofard.github.io/path_finding/2023/05/07/ICTS-%EC%B5%9C%EC%A0%81%EC%9D%98-%EA%B2%BD%EB%A1%9C-%EC%A1%B0%ED%95%A9%EC%9D%84-%EC%B0%BE%EC%95%84%EB%B3%B4%EC%9E%90!.html)에어 말했다시피 ID<sup>Independence Detection</sup>알고리즘에서 아이디어를 따온 듯 한데, 간단히 말해서 MAPF problem을 계산하는데 걸리는 시간은 보통 Agent의 갯수에 따라 지수적으로 증가하기 때문인 듯 하다.

그렇다면 Anytime-RHCR에서는 어떤 방식을 통해서 $\mathcal{A}_M$을 결정하는지 간단하게 살펴보자. 가장 첫번째로 제안되는 방법은 충돌로 연관된 Agent를 찾아 $CF$를 해결하는 것이다. 이 방법을 통해 전체 솔루션에서 충돌을 모두 해결하고난 뒤 이후에 솔루션의 최적성을 높인다.


### **If conflicts exist**

충돌을 통해 연관된 Agent를 찾아내는 과정은 충돌을 이용하여 Agent의 연관 관계를 나타내는 그래프인 $\mathcal{G}_{CF} = {(V_{CF}, E_{CF})}$ 정의하여 나타낸다. 여기서 각 정점인 $\mathcal{v} \in V_{CF}$는 각 agent를 나타내고, 간선인 $\mathcal{e} \in E_{CF}$는 $\omega_{init} < t \le \omega_{extd}$인 두 agent간의 conflict이다. Anytime-RHCR은 RHCR알고리즘에 기반하기 때문에 $\omega_{init}$이전에는 충돌이 존재하지 않고, Windowed MAPF Solver를 사용하기 때문에 $\omega_{extd}$이후의 충돌은 무시한다.

해당 논문에서 이러한 $\mathcal{G}_{CF}$를 이용해서 $\mathcal{A}_M$를 구하는 방법은 꽤나 단순하다. 서로 충돌로 연결되어 있는 $\mathcal{v}' \in V_{CF}$들을 원소로 갖는 $V_{CF}$ 부분집합 중 가장 큰 부분집합 ${V'}_{CF}$를 고르는 것이다. 그리고 난 뒤, $|{V'}_{CF}|$와 $N_M$의 크기 관계에 따라 조금씩 로직이 바뀐다.

우선 $|{V'}_{CF}| > N_M$인 경우에는 ${V'}_{CF}$중 랜덤하게 $N_M$개를 골라 $\mathcal{A}_M$를 구성한다. 반대로 $|{V'}_{CF}| < N_M$인 경우에는 ${V'}_{CF}$ 중 랜덤한 agent를 고르고, 해당 agent의 경로를 막는 agent까지 포함하여 $\mathcal{A}_M$를 구성한다.

### **If there are no conflicts**

만약 지정한 시간 범위 내에 충돌이 없는경우, RHCR의 방법론에 의해 $\mathcal{A}_M$를 구성한다. 간단하게 설명하면 다음과 같다.

1. 모든 agent 중 optimal path와 mapf를 통해 계산된 경로의 시간이 가장 많이 차이나는 $a_i$를 $\mathcal{A}_M$를 추가
2. 나머지 agent 중 $a_i$를 가로막는 agent의 집합을 $\mathcal{A}_M$에 추가한다.
3. $|{V'}_{CF}| = N_M$가 될때까지 1번과 2번을 반복한다.

이러한 과정을 통해 가장 지연된 agent를 찾고 해당 agent의 경로를 개선한다.

<br>

# **Corridor Conflict-Based Search**

이번 Section에서는 수정집합에 대해 경로를 개선할때 사용하는 Corridor-CBS에 대해 설명하려고 한다. Corridor-CBS은 기본적인 [CBS알고리즘](https://reofard.github.io/path_finding/2023/05/20/CBS-MAPF%EA%B3%84%EC%9D%98-%EC%84%B1%EA%B2%BD.html)을 확장한 알고리즘으로 Corridor Conflict를 효율적으로 해결할 수 있는 알고리즘이다.

## **Pseudo Code**

아래 그림은 Corridor-CBS 알고리즘의 의사코드이다. [기존의 CBS](https://reofard.github.io/path_finding/2023/05/20/CBS-MAPF%EA%B3%84%EC%9D%98-%EC%84%B1%EA%B2%BD.html)에서 확장된 내용은 파란색으로 표시되었다.

![C-CBS](/assets/img/Corridor-CBS.png)

Corridor-CBS 알고리즘에서 확장된 로직은 다음과 같다.

Corridor-CBS는 이러한 문제를 해결하기 위해 다음의 개선 단계를 추가합니다:

  1. corridor symmetry reasoning: 좁은 통로에서 발생하는 대칭 문제를 분석하고 이를 효율적으로 처리

  2. prioritizing corridor conflict: 충돌 목록에서 좁은 통로에서 발생한 충돌을 우선적으로 선택해 해결

  3. corridor heuristic: 좁은 통로 충돌을 해결하기 위해 추가적인 휴리스틱을 사용하여 탐색 효율성을 향상.

이러한 과정을 통해 Corridor-CBS는 효과적으로 topological map에서 협소로 문제를 효과적으로 해결한다고 한다.


## **corridor symmetry reasoning**

해당 논문에서는 [New Techniques for Pairwise Symmetry Breaking in Multi-Agent Path Finding](https://ojs.aaai.org/index.php/ICAPS/article/view/6661) 논문에서 소개된 corridor symmetry을 고려하여 Corridor-CBS에서 contraint를 생성하게끔 한다.

corridor symmetry reasoning은 충돌이 발생하는 노드 대신 복도의 입구에 제약 조건을 할당하는 contraint생성 방식이다. 아래 그림과 같은 상황에서 $t_1$과 $t_2$를 각각 $a_1$과 $a_2$가 복도 $C$의 끝점 $v_{q2}$와 $v_{q1}$에 도착하는 가장 빠른 시간 단계라고 가정해보자.

![corridor conflic](/assets/img/corridor_conflict.png)

그렇다면 아래 수식과 같이 각 agent에게 반대편의 agent가 복도를 완벽하게 지나갈 때까지의 시간동안 오지 못하도록 constraint를 추가 할 수 있다.

$$
ct_{1}=\langle a_{1}, v_{q_{2}}, [0, \min (t'_{1} - 1, t_{2} + len(C))] \rangle \\
ct_{2}=\langle a_{2}, v_{q_{1}}, [0, \min (t'_{2} - 1, t_{1} + len(C))] \rangle 
$$

이 방식으로 하면 corridor에 의해서 불필요하게 CT가 확장되는것을 효과적으로 막을 수 있다.

## **prioritizing corridor conflict**

해당 내용은 큰 내용을 담고있지는 않는다. [ICBS](https://www.ijcai.org/Proceedings/15/Papers/110.pdf)에서 카디널리티 계산보다 Corridor충돌의 우선순위를 최우선으로 해결하겠다는 내용이다.

이는 lifelong MAPF에서는 MDD생성을 위한 계산 비용이 너무 크기 때문이라고 한다. 그렇기 때문에 Corridor 충돌이 있는경우 이를 우선적으로 처리하여 계산 시간을 단축시킬 수 있다고 한다.

## **corridor heuristic**

해당 내용은 Corridor-CBS에서 High-level Node를 선택할 때 corridor conflict를 고려한 hueristic 값을 고려하여 선택하는 로직에 대한 설명이다. Corridor conflict가 존재하는 high-level Node $\mathcal{N}$이 있을 때, corridor conflict에 의해 증가하는 값은 아래 식과 같다.

$$
\Delta c_{ij} = \bar {c}_{ij} - c_{ij}
$$

이때 $c_{ij}$는 $a_i$​와 $a_j$​가 corridor conflict 충돌이 없을 때 경로의 최소 비용이고, $c_{ij}$는 충돌을 고려한 현재 경로에서의 비용이다.

해당 논문에서는 high-level node에서 휴리스틱 값을 계산하기 위해 $\mathcal{G}_{CCF} = {(V_
{CCF, W_{CCF}})}$를 만든다. $\mathcal{G}_{CCF}$는 각 agent사이에 발생하는 Corridor Conflict에 의한 비용 증가 증분을 나타낸 그래프로 각 정점인 $\mathcal{v}_i \in V_{CCF}$는 각 agent를 나타내고, 간선인 $\mathcal{e}_{ij} \in E_{CCF}$는 두 agent간의 corridor conflict 관계를 나타내며 $\mathcal{e}_{ij}$는 corridor conflict로 인해 발생한 cost의 증분값 $\Delta c_{ij}$를 갖는다.

이제 hueristic값을 해결하기 위해 해당 그래프에서 Edge-weighted Minimum Vertex Cover 문제를 해결하여 통해 모든 conflict를 해결하기 위해 constraint가 걸려야 하는 모든 agent 집합중 계산 비용의 합이 가장 낮은 집합을 계산한다. 아래 그림을 보면서 간단히 설명해보겠다.

![Edge-weighted Minimum Vertex Cover](/assets/img/corridor_hueristics.png)

Corridor conflict를 기반으로 한 휴리스틱 계산의 예는 다음과 같다. $\mathcal{N}$이 MAPF conflict를 고려하지 최단 경로를 가진 노드라고 가정해 보자. 이때 $\mathcal{N}$에서 corridor conflict를 검출하고, GCCF를 구성하면 위 그림에서 (b)와 같은 그래프가 나온다. 그런 다음 Edge-weighted Minimum Vertex Cover 문제를 해결하여 각 Agent에 걸릴 예상 Constraint를 계산한다. $a_1$에 6, $a_2$에 1, $a_4$에 7의 예상 최소 제약시간이 계산되면서 $\mathcal{N}$의 hueristic 값은 $\sum_{{v_i} \in V_{CCF}} {x_i} : h = 6 + 1 + 0 + 7 + 0 + 0 + 0 = 14$가 된다.

# **후기**

단순히 저자분이 한국인이셔서 신기한 마음에 읽기 시작한 논문이었지만 공장 환경에서 발생할 수 있는 여러가지 상황에 대해 다양한 논문에서 소개된 아이디어를 통해 해결하려고 하셨던 부분이 보여서 내가 하는 업무에서도 많은 참고가 되었던 논문이었던것 같다. 앞으로도 한국에서도 MAPF 분야가 더 활성화 되어서 아이디어와 의견을 교류할 수 있는 기반이 더 다져지면 좋을 것 같다.