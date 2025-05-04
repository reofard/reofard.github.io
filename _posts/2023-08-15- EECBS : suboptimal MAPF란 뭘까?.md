---
title:  "EECBS : suboptimal MAPF란 뭘까?"
excerpt: "Robot"

categories:
  - Path_Finding

tags:
  - [MAPF, Path Finding, Multi-Robot]
`
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
EECBS는 기존의 CBS의 속도 개선 버전이다. 우선 CBS는 앞서서 설명한 [포스트](https://reofard.github.io/path_finding/2023/05/20/CBS-MAPF계의-성경.html)가 있기 때문에 간단하게 CBS는 무엇이고 뭐를 개선했는지만 언급하고 넘어가려 한다. **CBS는 MAPF를 최적으로 푸는 대표적인 알고리즘**이다. 이 알고리즘은 다음과 같은 두 단계의 과정으로 이루어진다.

Low-level:  각 에이전트의 최적 경로 계산
High-level: 경로 간 충돌 발견 → 충돌 해결을 위해 분기(branching)

즉 각 Agent별 경로를 포함하는 상태공간을 탐색하는 일종의 Best-fit Search인 것이다. 이러한 방식은 최적의 해를 보장한다는 장점이 있지만, 시간복잡도가 충돌 횟수의 지수승으로 늘어나고, 충돌은 agent의 갯수, 맵상에서의 밀도에 비례하기 때문에 **CBS는 연산속도가 매우 느리다는 단점이 존재**한다.

EECBS는 앞서 설명한 CBS의 속도를 보완하기 위해서 나온 여러 Suboptimal CBS중 하나이다. 이번 논문에서는 여러 Suboptimal CBS들중 대표적인 ECBS를 간단하게 설명하고, EECBS에 대해서 소개를 한다.


<br>

# 1. **Enhanced CBS**
---
우선 ECBS에 대해서도 자세히 정리하고 넘어가려 한다. ECBS도 꽤 유명한 MAPF논문이라서 간단하게 회사에서 리뷰는 한적 있지만, 블로그에선 이번 EECBS의 포스트에서 같이 정의하고 넘어가려 한다.

## 1.1 **Focal Search**

우선 CBS의 핵심 요소인 Focal Search에 대해 알아보도록 하자. 이번 포스트에서는 focal search에 대해 간단하게 정리만하고, 다음에 제대로 정리하려고 한다. Focal Search는 특정 문제의 최적의 해가 $best-n$이고, 해의 비용이 $f(best-n)$일 때 $wf(best-n) \ge f(n)$를 만족하는 해 $n$을 구하는 알고리즘이다. Focal Search는 일반적인 $A^{\*}$ 알고리즘과 다르게 상태공간을 탐색할때 사용하는 우선순위 큐가 아래와 같이 2개이다.

- ```OPEN```: $A^{\*}$의 일반적인 우선순위 큐 ($f = g + h$로 정렬)

- ```FOCAL```: $wf(best-n) \ge f(n)$를 만족하는 노드들

여기서 ```FOCAL```은 **"좋은 상태들 중에서 골라보자"**라는 후보 해 리스트이다. 즉 어느정도 최적 해를 기준으로 $w$배수 내에 있는 상태들을 먼저 정렬하고, 그중 다시 확장할 상태를 정하는 알고리즘인 것이다.

## 1.2 **Low-Level Search**

먼저 ECBS의 Low-Level Search에 대해서 알아보자. **Low-Level Search는 제약 조건(CT 노드 N의 constraints)을 만족하는 경로를 탐색**한다. 이때 ECBS는 CBS와 달리 Focal Search를 통해 다른 Agent의 경로와 충돌이 가장 적은 경로를 탐색한다. 이때 Focal Search에서 ```OPEN```과 ```FOCAL``` 각 우선순위 큐의 정렬 조건은 아래와 같은 식을 따른다.

- ```OPEN```: $f(n) = g(n) + h(n)$

- ```FOCAL```: $d(n) = $ 다른 에이전트들과의 충돌 횟수

즉 ECBS의 Low-Level Search에서는 비용이 $wf(best-n)$ 이하인 솔루션 중 다른 로봇과 충돌이 가장 적은 경로를 반환하게 되는것이다. 이 과정에서 **단순히 빠른 길을 찾는 게 아니라, High-Level Search에서 처리할 충돌 수를 줄이기 위해 경로가 선택**되는것이다.

## 1.3 **High-Level Search**

우선 CBS에서 **High-Level Search는 선택된 노드에서 발견된 충돌을 해결하는 자식 상태 분기** 및 탐색하는 알고리즘이다. 이때 ECBS는 High-Level Search에서도 Focal Search 알고리즘을 확용하여 빠르게 솔루션을 계산한다. Low-Level Search와 같이 High-Level Search에서도 아래와 같이 High-Level 상태 $N$에 대해 ```OPEN```과 ```FOCAL``` 각 우선순위 큐의 정렬 조건을 아래와 같은 식을 통해 나타낼 수 있다.

- ```OPEN```: $cost(N) \le w × lb(bestlb),\ lb(N)=\sum^m_{i=1} f^i_{min}(N)$

- ```FOCAL```: $h_c(N) =$ 총 충돌 횟수

그 결과 ECBS의 High-Level Search에서는 충돌 횟수가 적은 상태부터 분기하게 되어 기존의 CBS보다 더 빠르게 충돌을 없애가면서 정확한 최적해는 아니지만, 항상 $w ×$ 최적해 비용 이하의 솔루션을 구할수 있게 된다고 한다.

<br>

# 2. **ECBS의 한계**
---
ECBS는 CBS보다 빠른 속도로 suboptimal 해를 찾을 수 있는 장점이 있지만, Focal Search를 사용하는 구조에서 오는 몇 가지 근본적인 한계가 존재한다. 이 한계들은 특히 복잡한 환경에서 ECBS가 지역적으로 탐색에 갇히는 원인이 될 수 있다고 한다. 한번 어떤 문제가 정확히 발생하는지 자세하게 알아보자.

## **문제 1: FOCAL 노드 선택 기준의 한계**

ECBS는 High-Level Search에서 노드를 선택할 때 다음과 같은 기준을 따른다:

- **조건**: $cost(N) ≤ w × lb(bestlb)$
- **Open 우선순위**: $cost(N) = h(N) + g(N)$이 작은 노드 우선
- **Focal 우선순위**: 충돌 수 $h_c(N)$가 작은 노드 우선

표면적으로는 효율적인 전략이지만, $h_c(N)$이 작은 노드의 $cost(N)$은 클 수 있다는 상황이 발생할 수 도 있게 된다. 이 말은 ECBS가 확장을 $h_c(N)$를 기준으로 하지만, $h_c(N)$ 낮다는것은 충돌지점을 회피하는 경로가 선택되는 것이고, 이는 돌아가는 경로가 생성되어 $cost(N)$가 높아지는 결과를 초래하는 경우가 많아진다. **즉 Focal List의 우선순위는 높아지지만, Open List의 우선순위는 낮아지는 방향으로 탐색을 진행**하게 된다. 결과적으로 ECBS는 충돌 수 $h_c(N)$만을 기준으로 선택했지만, 실제로는 suboptimality bound를 넘는 해($cost(N)$이 높은)만 반복적으로 선택하게 되어 탐색이 제한된 영역에 갇힐 수 있다. 자세한 예시는 논문에 나온 그림을 보면서 간단하게 설명해보겠다.

![ecbs limitation](/assets/img/ecbs_limitation.png)

위 그림에서 ECBS는 특정 브랜치 (예: CT 노드 1 → 2 → … → 487)를 따라 계속 확장된다. 이는 Conflict Tree의 깊이가 깊어질수록 $h_c(N)$가 낮아지기 때문이다. 결과적으로 탐색 시간에 따라  $cost(N)$은 증가하고 $h_c(N)$은 점점 감소한다. 결국 어느 시점에서 자식 노드들은 $cost(N) ≤ w × lb(bestlb)$ 조건을 만족하지 못하게 되어 FOCAL에 들어가지 못하고 확장이 멈춘다. 결과적으로
선택된 모든 이웃 노드들에서 비슷한 양상을 반복하면서, 알고리즘이 특정 지역에서 맴도는 **thrashing** 현상이 발생하게 된다. 이런 문제는 $cost(N)$과 $h_c(N)$의 사이에 음의 상관관계가 있을 때 자주 발생하게 된다.

<br>

## **문제 2: Lower Bound가 잘 올라가지 않음**

문제 1에서의 **thrashing**현상을 피하여 ECBS에서 탐색의 진행 여부를 결정하는 핵심은 **하한선** $lb(bestlb)$**의 증가**다.

$lb(bestlb)$가 커질수록 $w × lb(bestlb)$도 함께 커져, FOCAL의 범위가 넓어지고 탐색이 더 유연해지기 때문이다. 그러나 실제 ECBS에서는 앞서 언급한 점 때문에 $cost(N)$과 $h_c(N)$ 사이에 음의 상관관계가 존재하여 $lb(bestlb)$가 잘 커지지 않고, 탐색에 지속적인 지연이 발생한다. 이러한 문제가 발생시키는 악순환은 아래와 같다.

1. CT 노드를 확장해 자식 노드를 생성하면, 제약 조건이 추가되기 때문에 `lb`는 보통 같거나 커진다.
2. 반면, 충돌 수 $h_c(N)$는 줄어들 가능성이 높다.
3. 이로 인해 `bestlb`는 여전히 충돌이 많은 원래 노드로 유지되고, 이 노드는 여전히 FOCAL에 남아 확장이 지연된다.
3. 특히 많은 노드들이 동일한 $cost(N)$를 가지게 되면, FOCAL이 비워지지 않고 $lb(bestlb)$는 오랫동안 정체된다.
4. 이로 인해 탐색이 느려지고, 더 좋은 해를 찾기 위한 진전이 매우 더뎌진다.

이러한 문제로 인해 ECBS의 탐색은 매우 느리고 비효율적으로 진행되는 경우가 많다고 한다.

<br>

# **Explicit Estimation Conflict Based Search**
---
이번 논문에서 저자는 기존의 ECBS의 한계점과 여러 문제상황에 대해 해결하여 suboptimality를 만족하면서 빠른속도로 탐색 할 수 있는 알고리즘인 EECBS(Explicit Estimation Conflict Based Search)를 제안한다. 어떤 개선사항이 있었는지 한번 정리해 보려고 한다.

## **Explicit Estimation  Search**
앞서 설명한 탐색 효율성 문제를 해결하기 위해 해당 논문에서는 CBS에 EES(Explicit Estimation Search)를 도입하였다. EES는 위에서 언급한 Focal Search의 한계점을 극복하기 위해 디자인된 알고리즘으로, 기존의 ECBS를 해결하기 위해 해당 논문의 저자가 도입한 알고리즘이다. 무엇인지 간단하게 알아보자.

우선 EES에서는 **"지금 이 노드에서 출발해서 `bestlb`까지 가는 데 드는 총비용"을 대충 추정"하는 함수** $\hat{f}$를 도입하여 사용한다. EES는 기존에 Focal Search에서 확장할 노드를 관리하기 위해 `OPEN`, `FOCAL` 두가지의 리스트를 사용했던것과 달리 아래와 같이 3개의 리스트를 통해 확장 할 노드를 관리한다.

- ```CLEANUP```: 평범한 $A^*$ open 리스트. f = g + h로 정렬.
- ```OPEN```: inadmissible한 cost function $\hat{f}$의 값으로 정렬한 리스트. 또 다른 종류의 평범한 $A^*$ open 리스트이다.
- ```FOCAL```: OPEN에서 $\hat{f}$값이 $w × lb(bestlb)$ 이하인 것만 모은 리스트. 내부적으로 `distance-to-go`함수인 $d$에 의해 정렬된다.

사실 $\hat{f}$는 기존의 FOCAL Search에서 사용한 함수와 동일하게 하한선 내에서 더 공격적으로 탐색을 할 수 있도록 inadmissible한 정렬 조건을 나타내는 함수이다. 하지만 EES는 여기서 `distance-to-go` 함수를 사용하여 focal search의 한계를 돌파하고자 했다. `distance-to-go` 함수는 **"목표까지의 거리 또는 남은 단계 수의 추정"**하는 함수로, 최적일 것 같은 상태를 추정하는 함수가 아니라, **더 적은 탐색 단계로도 일단 해답은 구할 수 있는 상태**를 찾고자 하는 함수이다. 즉, 현재 상태가 얼마나 효율적인지가 아닌, 현재 상태에서 얼마나 목표 위치와 가까운지에 대한 기준인 것이다.

그렇다면 이러한 리스트들을 가지고 어떻게 

관심 있는 노드가 현재의 서브옵티멀 경계 내에 있지 않으면, 최소 f 값으로 노드를 확장하여 현재 서브옵티멀 경계를 개선합니다.
<br>

# **후기**
---
내용