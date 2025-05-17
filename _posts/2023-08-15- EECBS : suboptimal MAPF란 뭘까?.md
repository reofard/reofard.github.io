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
EECBS는 기존의 CBS의 속도 개선 버전이다. 우선 CBS는 앞서서 설명한 [포스트](https://reofard.github.io/path_finding/2023/05/20/CBS-MAPF계의-성경.html)가 있기 때문에 간단하게 CBS는 무엇이고 뭐를 개선했는지만 언급하고 넘어가려 한다. **CBS는 MAPF를 최적으로 푸는 대표적인 알고리즘**이다. 이 알고리즘은 다음과 같은 두 단계의 과정으로 이루어진다.

<br>

Low-level:  각 에이전트의 최적 경로 계산

High-level: 경로 간 충돌 발견 → 충돌 해결을 위해 분기(branching)

<br>

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

그렇다면 이러한 리스트들을 가지고 어떻게 상태공간을 확장하면서 탐색을 하는지도 알아보자. 우선 EES에서는 확장할 노드를 선택하기위해 $f(best_d) \le w \times f(best_f)$인 노드가 있는지 확인한다. 있다면, `FOCAL`에서 $best_d$를 확장한다. 하지만 해당하는 노드가 없을 경우, `OPEN`에서 $f(best_{\hat{f}}) \le w \times f(best_f)$ 를 만족하는 $best_{\hat{f}}$가 존재할 경우 해당 노드를 확장한다. 마지막으로 해당하는 노드가 없을경우, $f(best_f)$를 확장하는 방식으로 탐색을 진행하며, 앞서 나온 ECBS의 한계를 해결한다.

이러한 방식에 따라 **답을 찾을 수 있는 노드를 확장**하고, 없으면 **최적일 가능성이 높은 노드를 확장**하고, 그래도 없으면 그냥 **최소비용 노드를 확장**한다. 이러한 방식을 통해 앞서 설명한 한계를 돌파 할 수 있다고 한다.

## **Explicit Estimation CBS**

그렇다면 EECBS에서는 이러한 EES를 어떠한 방식으로 적용하여 탐색하는지도 알아보자. 우선 EECBS란 ECBS에서 High-Level Search를 Focal Search에서 Explicit Estimation Search로 대체하여 만든 알고리즘이다. 그렇기 때문에 앞의 EES설명 단원에서처럼 EECBS도 CT 노드를 3가지 리스트 `CLEANUP`, `OPEN`, `FOCAL`을 통해 관리한다. 그럼 각각의 리스트는 어떤 기준으로 정렬되고, 어떤 역할을 하는지 간단하게 알아보자.

`CLEANUP` 리스트는 기존의 ECBS에서 `OPEN`리스트와 유사하게, lowbound의 기준이 되는 함수로 정렬된 일반적인 $A^*$의 `OPEN`리스트와 같다. `OPEN`리스트는 앞선 설명과 마찬가지로 inadmissible한 cost function $\hat{f}$의 값으로 정렬된 리스트이다. 이때 $\hat{f}$은 아래와 같은 식으로 나타낼 수 있는데, 이는 해당 노드의 cost에 online learning을 이용한 가변적인 hueristic function인 $\hat{h}$ 함수의 합으로 나타내진다. $\hat{h}$ 함수는 다음 챕터에서 더 자세히 설명하도록 하겠다.

$\hat{f}(N) = Cost(N) + \hat{h}(N)$

마지막을 `FOCAL`리스트는 `OPEN`리스트의 노드들 중 $\hat{f}(N) \le w \times \hat{f}(best\_{\hat{f}})$을 만족하는 노드를 포함하고, 솔루션에 도달하는데 걸리는 비용을 나타내는 `distance-to-go` 함수인 $h\_c$로 정렬된다.

마지막으로 EECBS에서 High-Level Search에서 다음에 확장할 노드를 고르는 기준은 기존의 EES와 같이 `Focal`에서 먼저 확인하고, 만족하는게 없으면 `OPEN`에서 확인하고, 마지막으로 `CLEANUP`에서 확장하는 방식인데, 뭐 EES와 동일하기때문에 자세한 내용은 패스하고, 다음 포스트에서 자세히 정리해보려고 한다.

## **Online Learning of the Cost-To-Go**

주어진 CT 노드 N에서의 솔루션의 최소 비용 추정치는 전처리가 필요하지 않고 각 인스턴스별 학습이 가능하기 때문에 Online Learning이 가능하다고 한다. 이 과정은 노드 확장 중에 발생한 오류를 사용하여 검색 중에 ```cost-to-go```을 학습한다. 한번 어떤값을 학습하여 휴리스틱 비용으로 사용하는지 간단히 알아보자.

우선 특정 노드 $n$에 대해 admissible한 비용함수 $h$와 도달하는데 필요한 거리 비용함수(```distance-to-go```) $d$라는 휴리스틱 함수들이 있다고 하자. 여기서 거리 비용을 예측하는 휴리스틱 함수 $d$의 오차는 $\epsilon_d$으로 정의되고, 이는 아래와 같은 식으로 나타낼 수 있다.

$$
\epsilon_d(n) = d(bestchild(n)) - (d(n)-1)
$$

위 식은 "노드 $n$에서 자식노드를 한번 확장했을 때, 예상한 것처럼 $d(n)$이 1만큼 줄어들었는가?"라는 뜻으로 노드의 거리비용을 예측하기 위해 사용된 휴리스틱 함수가 얼마나 정확하게 예측했는지에 대한 값으로 볼 수 있다. 이와 마찬가지로 앞서 설명한 비용함수 $f$에 대한 에러를 $\epsilon_h$으로 나타낼 수 있으며 아래와 같은 식으로 나타낼 수 있다.

$$
\epsilon_h(n) = h(bestchild(n)) - (h(n) - c(n,bestchild(n)))
$$

여기서 $c(n_1, n_2)$는 노드 $n_1$에서 노드 $n_2$로 이동하는데 걸리는 비용을 의미한다. 앞서 설명한 두 에러를 각각 $one-step\ distance\ error$, $one-step\ cost\ error$라 하고, 각 error들은 노드가 확장이 종료되고 난 뒤 계산할 수 있다.

이러한 에러들은 검색과정에서 ```one-step error```의 분포가 균일하고, 관찰된 에러들의 평균을 통해 예측 할 수 있다고 가정한다. 그렇기 때문에 검색 과정에서 관찰된 모든 ```one-step error```의 평균을 유지하며, 이는 $\bar{\epsilon}\_d$와 $\bar{\epsilon}\_e$로 나타낸다. 이때 ```cost-to-go``` 노드 $n$에 대해 $\hat{h}(n) = h(n) + \frac{d(n)}{1-\hat{\epsilon}\_n(n)}\cdot\hat{\epsilon}\_h(n)$으로 근사 할 수 있다([출처](https://ojs.aaai.org/index.php/ICAPS/article/view/13474)).

이러한 Online learning 값을 기반으로 EECBS에서는 허용가능한 비용 휴리스틱함수 $\hat{h}$를 정의한다.그렇기 때문에 이 논문에서 CT 노드 $N$에서 앞에서 정의한 $\epsilon\_d$와 $\epsilon\_h$을 기반으로 하여 아래와 같은 식으로 나타낼 수 있다.

$$
\hat{h}(N) = \frac{h\_c(N)}{1-\bar{\epsilon_d}(N)} \cdot \bar{\epsilon}_h(N)
$$

따라서 $\hat{h}(N)$은 $h_c(N)$에서 선형이며, CT 노드의 충돌 수가 많을수록 CT 노드의 잠재적 비용 증가가 더 높아질 수 있음을 나타낼 수 있어, ```cost-to-go``` 값을 의도한 대로 정의 할 수 있다.

<br>

# **Extra Improvements Idea**
---

내용

<br>


# **후기**
---
사실 이번 회사에 들어오기 전까지 학교에서 Visual SLAM이나 Image Processing분야를 공부했었다. 하지만 스타트업에 들어오면서, 알고리즘, 암호보안 경진대회 수상실적이 있다는 이유로, 경로탐색 잘할 것 같다면서 멀티로봇 관제(경로탐색 및 작업할당) 개발 업무를 할당받았었다. 그러면서 처음에 회사에서 인수인계 해준 코드가 이 논문의 구현체를 비정형 그래프기반으로 어느정도 변형한 코드였다. 물론 기존에 있으시던 분들도 해당 분야에 대해 잘 모르시고, 일단 해야하니까 가장 유명한 논문 찾아서 ROS와 엮어서 당장 돌아가게끔만 만든 코드라 버그도 많고, 제대로 돌아가지 않던 코드였긴 했다. 하지만, 그것과도 별개로 내가 EECBS, 더 나아가 MAPF라는 분야에 대해서 잘 몰랐고, 그런 상황에서 MAPF를 입문하게된 최초의 알고리즘이 EECBS였고, 그렇기 때문에 더 자세히 알아보고, 완벽하게 이해하고 싶다는 생각에 이전 논문처럼, 간단하게 이해하고만 넘어가지 않고, 실제 구현체를 보면서 더 완벽하게 정리해보려고 한다.