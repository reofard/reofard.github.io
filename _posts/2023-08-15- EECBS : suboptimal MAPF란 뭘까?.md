---
title:  "EECBS : suboptimal MAPF란 뭘까?"
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

# **Explicit Estimation Conflict-Based Search**
---
이전에 정리했던 MAPF알고리즘인 [Conflict Based Search](https://reofard.github.io/path_finding/2023/05/20/CBS-MAPF계의-성경.html)에는 앞서서 정리했던 [LPCBS](https://reofard.github.io/path_finding/2023/06/20/LPCBS-Online,-Lifelong-mapf에-대해-알아보자!.html)같이 여러 확장, 변형된 버전이 있다. 회사에서 일하다 보니, 실제 적용 가능한 사례에 대해 주로 찾아보게 되었는데, [Jiaoyang Li](https://jiaoyangli.me) 이분이 이런 MAPF의 실제 적용에 대해 많이 연구하시는거 같아서 이번 EECBS논문 리뷰 이후에도 하나씩 관련된 논문을 찾아보고 정리해보려고 한다.

추가적으로 이번 포스트는 CBS와 문제 도메인, 정의등이 완벽하게 일치하기때문에
전에 작성했던 [CBS 포스트](https://reofard.github.io/path_finding/2023/05/20/CBS-MAPF계의-성경.html)의 정의 부분을 간단하게 읽고 오면 좋을 것 같다.

<br>

# **간단한 소개**
---
우선 EECBS는 기존의 CBS의 속도 개선 버전이다. 우선 CBS는 앞서서 설명한 [포스트](https://reofard.github.io/path_finding/2023/05/20/CBS-MAPF계의-성경.html)가 있기 때문에 간단하게 CBS는 무엇이고 뭐를 개선했는지만 언급하고 넘어가려 한다. **CBS는 MAPF를 최적으로 푸는 대표적인 알고리즘**이다. 이 알고리즘은 다음과 같은 두 단계의 과정으로 이루어진다.

Low-level:  각 에이전트의 최적 경로 계산
High-level: 경로 간 충돌 발견 → 충돌 해결을 위해 분기(branching)

즉 각 Agent별 경로를 포함하는 상태공간을 탐색하는 일종의 Best-fit Search인 것이다. 이러한 방식은 최적의 해를 보장한다는 장점이 있지만, 시간복잡도가 충돌 횟수의 지수승으로 늘어나고, 충돌은 agent의 갯수, 맵상에서의 밀도에 비례하기 때문에 **CBS는 연산속도가 매우 느리다는 단점이 존재**한다.

EECBS는 앞서 설명한 CBS의 속도를 보완하기 위해서 나온 여러 Suboptimal CBS중 하나이다. 이번 논문에서는 여러 Suboptimal CBS들중 대표적인 ECBS를 간단하게 설명하고, EECBS에 대해서 소개를 한다.


<br>

# **1. Enhanced CBS **
---
우선 ECBS에 대해서도 자세히 정리하고 넘어가려 한다. ECBS도 꽤 유명한 MAPF논문이라서 간단하게 회사에서 리뷰는 한적 있지만, 블로그에선 이번 EECBS의 포스트에서 같이 정의하고 넘어가려 한다.

## 1.1 **Focal Search**

우선 CBS의 핵심 요소인 Focal Search에 대해 알아보도록 하자. 이번 포스트에서는 focal search에 대해 간단하게 정리만하고, 다음에 제대로 정리하려고 한다. Focal Search는 특정 문제의 최적의 해가 $best-n$이고, 해의 비용이 $f(best-n)$일 때 $wf(best-n) \ge f(n)$를 만족하는 해 $n$을 구하는 알고리즘이다. Focal Search는 일반적인 $A^{\*}$ 알고리즘과 다르게 상태공간을 탐색할때 사용하는 우선순위 큐가 아래와 같이 2개이다.

```OPEN```: $A^{\*}$의 일반적인 우선순위 큐 ($f = g + h$로 정렬)

```FOCAL```: $wf(best-n) \ge f(n)$를 만족하는 노드들

여기서 ```FOCAL```은 **"좋은 상태들 중에서 골라보자"**라는 후보 해 리스트이다. 즉 어느정도 최적 해를 기준으로 $w$배수 내에 있는 상태들을 먼저 정렬하고, 그중 다시 확장할 상태를 정하는 알고리즘인 것이다.

## 1.2 **Low-Level Search**

먼저 ECBS의 Low-Level Search에 대해서 알아보자. **Low-Level Search는 제약 조건(CT 노드 N의 constraints)을 만족하는 경로를 탐색**한다. 이때 ECBS는 CBS와 달리 Focal Search를 통해 다른 Agent의 경로와 충돌이 가장 적은 경로를 탐색한다. 이때 Focal Search에서 ```OPEN```과 ```FOCAL``` 각 우선순위 큐의 정렬 조건은 아래와 같은 식을 따른다.

```OPEN```: $f(n) = g(n) + h(n)$

```FOCAL```: $d(n) = $ 다른 에이전트들과의 충돌 횟수

즉 ECBS의 Low-Level Search에서는 비용이 $wf(best-n)$ 이하인 솔루션 중 다른 로봇과 충돌이 가장 적은 경로를 반환하게 되는것이다. 이 과정에서 **단순히 빠른 길을 찾는 게 아니라, High-Level Search에서 처리할 충돌 수를 줄이기 위해 경로가 선택**되는것이다.

## 1.3 **High-Level Search**

우선 CBS에서 **High-Level Search는 선택된 노드에서 발견된 충돌을 해결하는 자식 상태 분기** 및 탐색하는 알고리즘이다. 이때 ECBS는 High-Level Search에서도 Focal Search 알고리즘을 확용하여 빠르게 솔루션을 계산한다. Low-Level Search와 같이 High-Level Search에서도 아래와 같이 High-Level 상태 $N$에 대해 ```OPEN```과 ```FOCAL``` 각 우선순위 큐의 정렬 조건을 아래와 같은 식을 통해 나타낼 수 있다.

```OPEN```: $cost(N) \le w × lb(bestlb),\ lb(N)=\sum^m_{i=1} f^i_{min}(N)$

```FOCAL```: $hc(N) =$ 총 충돌 횟수

그 결과 ECBS의 High-Level Search에서는 충돌 횟수가 적은 상태부터 분기하게 되어 기존의 CBS보다 더 빠르게 충돌을 없애가면서 정확한 최적해는 아니지만, 항상 $w ×$ 최적해 비용 이하의 솔루션을 구할수 있게 된다고 한다.

<br>

# 2. **ECBS의 한계**
---
ECBS는 CBS보다 빠른 속도로 suboptimal 해를 찾을 수 있는 장점이 있지만, Focal Search를 사용하는 구조에서 오는 몇 가지 근본적인 한계가 존재한다. 이 한계들은 특히 복잡한 환경에서 ECBS가 지역적으로 탐색에 갇히는 원인이 될 수 있다고 한다. 한번 어떤 문제가 정확히 발생하는지 자세하게 알아보자.

## **문제 1: FOCAL 노드 선택 기준의 한계**

ECBS는 High-Level Search에서 노드를 선택할 때 다음과 같은 기준을 따른다:

- **조건**: `cost(N) ≤ w × lb(bestlb)`
- **우선순위**: 충돌 수 `hc(N)`가 작은 노드 우선

표면적으로는 효율적인 전략이지만, 실제로는 다음과 같은 문제가 발생할 수 있다:

- `cost(N)`이 기준 내에 있다고 해도, 해당 노드의 자식 노드들에서는 실제 해 비용이 suboptimality bound를 초과할 수 있다.
- 즉, 겉보기에는 괜찮아 보여서 선택된 노드가, 실제로는 형편없는 해로 이어질 수 있다.

#### 🌀 실제 현상 (예시 기반)

- ECBS가 특정 브랜치 (예: CT 노드 1 → 2 → … → 487)를 따라 계속 확장한다.
- 시간이 지날수록 `cost(N)`은 증가하고 `hc(N)`은 점점 감소한다.
- 결국 어느 시점에서 자식 노드들은 `w × lb(bestlb)` 조건을 만족하지 못하게 되어 FOCAL에 들어가지 못하고 확장이 멈춘다.
- 그 다음 선택된 이웃 노드도 비슷한 양상을 반복하면서, 알고리즘이 특정 지역에서 맴도는 **thrashing** 현상이 발생하게 된다.
- 이런 문제는 `cost(N)`과 `hc(N)` 사이에 음의 상관관계가 있을 때 자주 발생한다.

👉 결과적으로 ECBS는 충돌 수 `hc(N)`만을 기준으로 선택했지만, 실제로는 suboptimality bound를 넘는 해만 반복적으로 선택하게 되어 탐색이 제한된 영역에 갇힐 수 있다.

<br>

## **문제 2: Lower Bound가 잘 올라가지 않음**

ECBS에서 탐색의 진행 여부를 결정하는 핵심은 **하한선 `lb(bestlb)`**의 증가다.

- `lb(N)`: CT 노드 N 아래에서 가능한 최소 해 비용의 추정
- `bestlb`: 현재 CT 노드 중 가장 낮은 `lb`를 가진 노드

`lb(bestlb)`가 커질수록 `w × lb(bestlb)`도 함께 커져, FOCAL의 범위가 넓어지고 탐색이 더 유연해진다. 그러나 실제 ECBS에서는 다음과 같은 문제가 발생한다:

#### 📉 현상

- CT 노드를 확장해 자식 노드를 생성하면, 제약 조건이 추가되기 때문에 `lb`는 보통 같거나 커진다.
- 반면, 충돌 수 `hc(N)`는 줄어들 가능성이 높다.
- 이로 인해 `bestlb`는 여전히 충돌이 많은 원래 노드로 유지되고, 이 노드는 여전히 FOCAL에 남아 확장이 지연된다.
- 특히 많은 노드들이 동일한 `cost(N)`를 가지게 되면, FOCAL이 비워지지 않고 `lb(bestlb)`는 오랫동안 정체된다.
- 이로 인해 탐색이 느려지고, 더 좋은 해를 찾기 위한 진전이 매우 더뎌진다.

<br>

# **후기**
---
내용