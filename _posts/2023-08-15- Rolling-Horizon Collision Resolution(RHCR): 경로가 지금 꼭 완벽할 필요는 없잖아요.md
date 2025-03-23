---
title:  "Rolling-Horizon Collision Resolution(RHCR): 경로가 지금 꼭 완벽할 필요는 없잖아요"
excerpt: "Robot"

categories:
  - Path_Finding

tags:
  - [MAPF, Path Finding, Multi-Robot]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2023-08-15
last_modified_at: 2023-08-15
---

# **Lifelong Multi-Agent Path Finding in Large-Scale Warehouses: 실제 물류창고에서 MAPF돌리는법**
---
이번 논문은 저자중 Amazon Robotics가 있어서 신기한 마음에 보게 되었다. 논문의 이름부터 저자까지 모두 물류창고에서 어떻게 MAPF 문제를 해결할지에 대한 의지가 보이는데, 해당 논문을 한번 읽어놓으면 단순 기술 습득 외에도 실제 현장에서 나온 인사이트나 경험도 같이 얻을 수 있을 것 같아서 한번 읽어보려고 한다.

이 논문의 배경은 [LPCBS](https://reofard.github.io/path_finding/2023/06/20/LPCBS-Online,-Lifelong-mapf%EC%97%90-%EB%8C%80%ED%95%B4-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90!.html)와 마찬가지로 "평생(Lifelong) MAPF"라는 문제에 집중한다. 간단하게 설명하면 agent들이 계속해서 새로운 목표 위치에 도달해야 하는 상황을 가정하는 문제이다. 즉 Amazon같은 대형 물류 기업에서 대형 물류창고에서의 로봇에 의한 자동화를 어떻게 수행했냐가 이 논문의 주제라고 보여진다.

이 논문에서 제안하는 새로운 해결 방법은 **Rolling-Horizon Collision Resolution (RHCR)**이다. RHCR에서는 MAPF 문제를 Windowed MAPF라는 일련의 문제로 분해하여 처리한다. "Windowed MAPF"는 제한된 시간 범위 내에서만 에이전트들의 경로 간 충돌을 고려하고, 그 범위를 넘어선 충돌은 무시하는 방법이다. 여기까지만 봐서는 단순하게 CBS에서 우선순위가 높은 근시일내의 충돌만을 고려하는 방법같은데, 어떤 아이디어가 들어갔고, 실제로 MAPF를 현장에서 쓰기위해 어떤 고려가 있었는지를 한번 논문을 읽어보면서 알아보려한다.

# **Problem Definition**
---
언제나처럼 해당 논문에서 다루는 문제부터 정의해보자. 사실 graph $G = (V, E)$, Agent 집합 $\{a_1, a_2, ... , a_n\}$등의 입력은 전에 다루었던 [포스트](https://reofard.github.io/path_finding/2023/04/22/MAPF%EB%9E%80-%EB%AC%B4%EC%97%87%EC%9D%BC%EA%B9%8C.html)와 거의 비슷하다. 하지만 이번 논문에서는 각 agent별 도착 목표 위치를 사전에 알지 못하는 온라인 환경을 배경으로 한다. 그렇기 때문에 저자는 경로탐색 모듈 바깥에 목표 위치를 요청 할 수 있는 Task assigner가 존재한다고 가정하고, 목표 지점은 Task Assigner로부터 실시간으로 받는다.

또한 태스크 할당 로직뿐 아니라 로봇이 주어진 경로를 완벽하게 수행한다는 가정을 통해 경로 계획과 주어진 경로의 실행이 분리된 프레임워크를 고려한다.

# **Rolling-Horizion Collision Resolution**
---
Rolling-Horizion Collision Resolution(이하 RHCR)에 대해서 톺아보자. RHCR에는 유저가 지정해야하는 두가지 파라미터가 있다 바로 $w$와 $h$이다. 먼저 $w$는 RHCR 프레임워크에서 Windowed MAPF Solver가 $w$시간내의 충돌을 모두 해결해야 한다는 제약조건이다. 그리고 $h$는 h시간마다 Windowed MAPF Solver를 통해 모든 로봇의 경로를 재계획하는 주기를 의미한다. 이때 $w$는 $h$보다 커야만 $h$주기 내에 충돌이 발생하지 않는다.

시간단계 임의의 $t$에서 시작하는 모든 Windowed MAPF 계산 주기의 시작에서 RHCR은 각 agent의 시작위치 $s_i$를 Agent $a_i$의 $t-timestep$에서의 위치로 지정하고, 목표위치의 도착 순서 $g_i$를 Task Assigner로부터 받아 설정한다. 이때 $s_i$는 단순한 경로 탐색 시작점이지만, $g_i$는 i번째 agent가 순차적으로 방문해야 하는 목표노드의 리스트이다. 그 이후, RHCR은 모든 Agent가 각자의 목표위치에 도착해야 할 최소시간단계 $d$를 계산한다. $d$의 계산 공식은 아래 식과 같다.

$$
d = dist(s_i, g_i[0]) + \sum^{\mid g_i \mid - 1}_{j=1} dist(g_i[j-1], g_i[j])
$$

이때 $dist(x, y)$는 위치 $x$에서 위치 $y$까지의 거리, $\mid g_i \mid$는 i번째 agent가 방문해야 하는 목표 위치의 개수이다. 이 식에 따라서 계산된 $d$가 $h$보다 작은경우 해당 agent는 해당 주기 내에 목표를 모두 방문하여 유휴상태가 되기 때문에 Task Assigner로부터 추가적인 목표 위치를 받아야 한다.

목표위치 업데이트가 완료되면 RHCR은 본격적으로 Windowed MAPF Solver를 호출하여 $[t, t + w]$ 시간사이에 충돌이 없는 경로를 계산한다. 그 이후 각 agent는 $h$시간동안 주어진 경로에 따라 agent를 이동시키고 방문한 노드를 목표 위치를 제거한다.

RHCR의 Windowed MAPF Solver는 [Online Multi-Agent Pathfinding](https://ojs.aaai.org/index.php/AAAI/article/view/4769) 논문에서 나온 flowtime이라고 불리는 Lifelong MAPF를 위한 목적함수를 사용한다.

## **$A^*$ for a Goal Location Sequence**

RHCR에서 사용하는 Windowed MAPF Solver는 기존의 CBS의 Low-level Searcher와 다르게 **여러 개의 목표 위치(Goal Location Sequence)를 순서대로 방문하는 경로**를 찾아야 한다. 그렇기 때문에 해당 논문에서는 [Multi-Label A\*](https://ojs.aaai.org/index.php/ICAPS/article/view/3474)를 사용한다. Multi-Label $A^*$의 pseudo code는 아래 그림과 같다.

![multi_label_a\*](/assets/img/multi_label_a*.png)

Multi-Label $A^\*$에서는 기존의 $A^\*$와 달리 확장되는 노드 $N$에 추가적인 요소인 $N.label$이 붙는다. $N.label$은 루트 노드에서 현재 노드까지의 경로중에 이미 방문한 목표 위치의 개수를 의미한다. 또한 휴리스틱 함수또한 다른데, $h-value = dist(N, g[N.label]) + \sum^{\mid g \mid - 1}_{j = N.label} dist(g[j], g[j+1])$로 나타낼수 있다. 이 값은  **다음 목표 위치까지의 거리 + 미래 목표 위치들 간의 거리 총합**을 나타낸다.

## **Bounded-Horizon MAPF Solvers**

그렇다면 정확히 Windowed MAPF Solver란 뭐고, 어떻게 구현되는지 보자. 해당 논문에서는 충돌 회피를 첫 $w$ timestep 동안만 고려하는 방식을 채용한 MAPF 인스턴스를 Windowed MAPF Solver, Bounded-Horizon MAPF Solvers등으로 부른다. Bounded-Horizon MAPF Solvers는 아래와 같은 특징이 있다.

1. 처음 $w$ 스텝까지만 충돌을 고려하고 그 이후는 무시

2. 이후에는 모든 agent가 최단 경로를 따라 이동한다고 가정

3. 충돌 탐지를 적게하여 계산 속도의 고속화

해당논문에서는 Conflict Based Search, Conflict Avoidance $A*$, Priority Based Search등의 모든 MAPF 알고리즘들을 어떻게 Bounded-Horizon MAPF Solvers로 개조하는지 설계하는데, 이번 포스트에서는 CBS부분만 정리해보려고 한다. 궁금한 사람은 논문에서 직접 찾아보자(그렇게 내용이 많지는 않고, 간단하다).

기존의 CBS는 전체 경로에서 충돌을 탐지하고 해결한다. Bounded-Horizon CBS에서는 하지만 Bounded-Horizon CBS는 첫 $w$ timestep에서의 충돌만 탐지하고, 해결한다.

## **Behavior of RHCR**

해당 논문에서는 단순히 미래의 충돌을 무시하는 방식인 Bounded-Horizon CBS이 어떤 방식으로 유의미한 결과를 계산하는지에 대해서 간단하게 예시를 들어 설명한다. 우선 아래의 예시 그림을 한번 봐보자.

![시간 지평선 w에 따른 효율성](/assets/img/relation_w_and_h.png)

그림 (a)는 $w=4,\ h=2$, 그림 (b)는 $w=8,\ h=2$일때의 RHCR이 계획한 경로를 나타낸다. 논문에서는 timestep 1이 존재하지 않는데, 이는 RHCR에서 재탐색 주기가 2이기 때문에 timestep이 0, 2, 4일때 Bounded-Horizon CBS를 계산하기 때문에 이렇게 나타낸듯 하다.

먼저 그림 (a)에서의 timestep 0에서 각 Agent의 경로를 탐색했을 때 $w$시간 내에 검출되는 충돌 없이, 모두 본인의 목적지까지의 최단경로가 각 agent에 주어진다. 하지만 timestpe 2가 되고 난 후, 3번 Agent가 새로운 목표 위치를 받고 난 후 다시 경로를 계산하면 3번의 새로운 목표위치까지의 경로 때문에 1번 Agent가 1 timestep만큼 대기를 하게 된다. 반면에 그림 (b)의 경우는 $w$가 더 크기 때문에 오히려 lifelong MAPF문제에서 비효율이 발생한다고 한다. 먼저 timestep 0에서 1, 2번 Agent가 노드 A에서 충돌하는걸 예측하고 2번이 1번 기다렸다가 가는 경로가 계산된다. 그 이후 timestep 2에서 1번과 3번이 Agent가 B노드에서 충돌을 회피하기 위해 3번이 1 timestep 대기하는 경로가 계산된다.

결국 (b)가 더 먼 미래의 충돌까지 고려하지만 (a)는 1회 대기, (b)는 2회 대기하는 경로가 계산되었다. 즉 Task Assigner의 존재 때문에 3번 Agent가 2번째로 받은 Goal을 timestep 0시점에서는 알 수 없었고, 그 결과 $w$가 더 크더라도, 3번 Agent의 충돌을 고려되지 않아 비효율이 발생하게 된 것이다. 이는 저자들이 진행한 실험에서도 $w$가 클때보다 작을때 throughput(처리량)이 더 향상되는 경우가 있었다고 한다.

## **Avoiding Deadlocks**

앞선 내용에서는 무조건 $w$가 높다고 효율이 나오는것은 아니다라는 결론이 나왔다. 하지만 반대로 $w$가 낮으면 RHCR은 교착상태에 빠질 수  도 있는데, 아래와 같은 사진을 예시로 들었다.

![RHCR Deadlock](/assets/img/RHCR_deadlock.png)

위 그림에서 $w=2,\ h=2$인 경우를 상정해보자.

timestep 0에서 Windowed MAPF 솔버는 아래와 같이 $w$ timestep동안 충돌이 없는 경로를 반환한다.

- $a_1$​: [B, B, B, C, D, E] (길이 5)
- $a_1$​: [C, C, C, B, A, L] (길이 5)

하지만 실제로 충돌을 회피하고 두 Agent 모두 목적지에 도달하는 방법은 아래와 같은 경로는 다음과 같다.

- 한 Agent가 위쪽 통로를 사용하는 경로

- 한 Agent가 먼저 아래쪽 통로에서 나가 다른 Agent가 목표 위치에 도착하도록 하고, 이후 다시 통로로 들어오는 경로

하지만 이러한 경로들은 flowtime이 높아 선택되지 않고, 이후 timestep 2에서 두 Agent가 여전히 셀 B와 C에서 대기하는 상태로 돌아오게 된다. 결과적으로 두 Agent는 셀 B와 C에서 영원히 대기하게 되어, 목표 위치에 도달하지 못하는 교착 상태(deadlock)가 발생한다.

이러한 상황을 방지하기 위해 RHCR에서는 potential function을 제안한다. 이를 통해 RHCR은 현재 모든 Agent의 진행상황을 평가하고, Agent가 충분한 진행을 이루지 못하는 경우 $w$를 증가시키는 방법으로 데드락을 회피한다. Potential function $P$는 Windowed MAPF Solver로부터 모든 Agent의 경로를 받아 다음과 같은 식을 통해 진행상황을 평가한다. 식이 너무 길어져서 아래 두 식에서는 $COMPUTEHVALUE$를 $Compute$로 나타내겠다.

$$
P(w) = \mid \{ a_i | Compute(x_i, l_i) \le Compute(s_i, 0), 1 \le i \le m \} \mid
$$

$$
Compute(Location\ x, Lavel\ l) = dist(x, g_i[l]) + \sum^{\mid g_i \mid - 1}_{j=l+1} dist(g_i[j-1], g_i[j])
$$

위 식에서 $x_i$는 i번째 Agent의 timestep $w$에서의 위치이고, $l_i$는 timestep $w$까지 Agent가 지난 목표 위치의 개수이다. 이때 $COMPUTEHVALUE$ 함수는 Agent가 현재 위치에서 마지막 목표위치까지 도착할때 걸리는 최단경로를 나타내게 된다. 이때 $P(w)$은 모든 Agent중 **현재 위치에서 목표위치를 모두 순회하는 최소거리**보다 $w$**시간의 위치에서** $w$**남은 모든 목표 위치를 순회하는 최소 거리**가 긴 Agent의 개수를 나타내게 되는것이다.

이를 위에서 설명했던 Deadlock이 발생했을때의 경로 양상을 보면 간단하게 이해할 수 있다. 우선 Deadlock이 발생하면 서로 교착상태에 빠진 Agent들은 반대로 움직인다. 그렇게 목표지점방향으로 $w$시간만큼 움직어도 충돌이 검출되지 않을 정도까지 다시 멀어지면 다시 로봇들은 교착이 발생한곳으로 이동한다. 이러한 상황에서 $P(w)$는 **현재 w시간 뒤에 로봇이 목표위치로부터 멀어져있는 상태**를 감지하는것이다.

이렇게 $P(w)$를 계산하고 난 뒤, 사용자가 결정한 하이퍼파라미터 $p$와 비교를 하여 $P(w) \ge p$ 일때까지 w를 증가시켜 교착을 충분히 막을수 있을때까지 $w$를 증가시키는 방식으로 Deadlock을 예방한다.

하지만 결정적으로 이러한 알고리즘은 교착이 발생한 다음, 경로가 반복적으로 교착을 해결하지 못하는 상황을 감지하여 이를 해결하는 방식이기 때문에 실질적으로 eecbs에서 다룬 Corridor같은 문제를 효율적으로 해결하지 못하는 단점이 있을 것 같다.

# **Throughput vs Completeness**
---
하지만 RHCR은 근본적으로 처리량과 도착 보장을 동시에 완벽하게 제공할 수 는 없다고 한다. 많은 Agent가 수평으로 연이어서 이동하고 있지만 한 Agent만 수직으로 이동하려고 하는 교차점이 있을 때, 항상 수평 Agent가 움직이고 수직 Agent가 기다리면 Throughput은 최대화되지만 수직 Agent가 목표 위치에 도달할 수 없어 Completeness를 잃게 된다. 반대로 수직 Agent가 움직이고 수평 Agent가 기다리면 Completeness을 보장할 수는 있지만 Throughput은 더 낮아진다. 이 문제는 $w = ∞$로 사용하더라도 발생한다. 왜냐하면 결국 **모든 Agent의 Goal을 현재 timestep에서 알수 없기 때문**이다. 그렇기 때문에 해당 논문에서는 Completeness 대신 Throughput에 초점을 맞춘다고 한다.

# 후기
---
회사에서 실제로 산업현장에서 사용할 제품을 개발하다보니, 내가 학부연구생으로 있을때와 논문을 찾을때와 읽을 때의 마음가짐이 조금씩 달라지는거 같다. 이전에는 성능 좋고, 재미있어 보이는것을 공부했었다면, 이제는 **"그래서 지금 내가 갖고있는 문제 도메인에 맞을까?"**, **"적용하는데 드는 소요에 비해 적용되었을때 얻을수 있는 보상이 높은가?"** 와 같이 더 현실적이고, 내 재미보다는 회사의 문제를 해결하는데 더 큰 포커스가 맞추게 되는 것 같다. 이번 논문은 **CBS의 탐색시간은 충돌의 갯수의 지수승으로 증가한다.** 라는 CBS의 근본적인 문제를, 그럼 **당장 중요도가 높은 근미래의 충돌만 고려하고 나머지는 무시하자!** 라는 아이디어로 매우 심플하게 해결했다. 물론 이렇게 접근하면 당연히 **최적의 해가 아니다.** 하지만 **resonable한 output을 뱉느냐?** 라고 물어보면 그건 또 맞다. 이 논문의 공동 저자에는 Amazon Robotics가 있고, 논문이 출판된 시기를 보면 아마 KIVA의 솔루션이 이 알고리즘을 통해 돌아가는게 아닐까 싶긴한데, 역시 산업에서 실제로 기술을 사용하려면, 무조건 기술이 고도화되고, 차세대 솔루션이 필요한건 아니고, 이러한 간단하지만, 정말 강력한 아이디어 하나로 문제가 이렇게 심플하게 풀릴수 있구나 라는 생각이 들었다.