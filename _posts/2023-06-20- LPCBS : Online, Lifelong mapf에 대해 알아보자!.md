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

> ???

### **Proposition 2**

> ???

그럼 위의 전제를 기반으로 DMAPF의 문제에서 CBS가 최적임을 증명해보자

### **Proof**

> ???

<br>

## **후기**

내용