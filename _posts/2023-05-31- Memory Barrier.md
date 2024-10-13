---
title:  "Memory Barrier"
excerpt: "Computer Architecture"

categories:
  - Operating_System

tags:
  - [Computer Science, Operating System]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2023-05-31
last_modified_at: 2023-05-31
---

# 앞선 내용 복기

며칠전 정리한 [포스트](https://reofard.github.io/operating_system/2023/05/27/Multi-core에서의-Multi-Processing.html)에서는 캐시일관성의 두가지 룰인 Coherency와 Consistency에 대해 알아보고 왜 이런문제가 발생하고, 어떻게 해결할 수 있는지 알아봤다.

지난번에 Coherency에 대해 정리할 때, 정보가 가장 최신이라는 보장을 해주는것 이라고 했었다. 하지만 **violation of global memory ordering**의 예제를 보면 store buffer에서 처리되는 시간에 따라 서로다른 변수의 값 변경이 프로그래머가 의도한대로 처리되지 않을 수 있다는것을 볼 수 있다.

즉 cache Coherency protocol이 항상 최신의 메모리 값을 보장해주는것은 아니다. 이 부분에 대해서는 각 cpu 아키텍쳐마다 조금씩 다르고, 관련 내용을 조금 찾아보니까 조금 복잡해서 내용을 따로 만들어 다뤄보려고 한다.

<br>

# **Memory Barrier**

Memory Barrier는 **명령어들이 실행되는 순서인 execution order**와 **메모리 연산의 결과가 CPU들에게 보이는 순서인 perceived order**를 일치시켜주는 역할을 진행한다. 먼저 간단한 Memory Barrier의 예제에 대해 알아보자.

## **Simple Memory Barrier Example**

가장 먼저 **violation of global memory ordering**의 예제에서 간단한 Memory Barrier가 적용된 아래 코드의 실행 순서에 대해 정리해보자. ```smp_mp()```는 메모리 베리어로 store buffer로 하여금 이전에 적용된 값이 cache에 적용되기 전까지 이후의 값을 cache에 적용시키는것을 보류시키는 역할을 수행한다.

```c
void foo(void) // execute by Core 0
{
    a=1;
    smp_mp()
    b=1;
}
void bar(void) // execute by Core 1
{
    while (b == 0) continue;
    assert(a == 1);
}
```

```smp_mp()```의 적용으로 인해 아래와 같이 실행하여, execution order와 perceived order를 일치 시킬 수 있게 된다.

![뮤텍스 문제](/assets/img/memory_barrier_ex.png)

이는 Memory Barrier를 통해 store buffer가 cache에 실제로 적용된 값을 수정하는 순서를 강제하는 역할을 해주기 때문이다. 즉 Memory Barrier는 **execution order를 통해 store buffer가 cache에 값을 업데이트 하는 순서를 강제**하여 **perceived order를 동기화** 시켜주는 역할을 해준다.

<br>

# **Invalidate Queues**

하지만 store buffer는 CPU가 데이터를 메모리에 기록하기 전에 잠시 저장해 두는 작은 임시 저장소이다. 그렇기 때문에 캐시 미스나 캐시 무효화(invalidation) 작업에 의해 메모리에 기록하는 과정이 지연되며 buffer가 가득 차버리는 문제가 발생할 수 있다. 이는 Memory Barrier가 store buffer의 일부 값을 비우지 못하게 blocking하면서 앞서 말한 문제 사항이 더 자주 발생 할 수 있게 된다. 이러한 문제를 해결하기 위해 **Invalidate queue**라는것을 도입해서 문제를 해결했는데, 어떻게 수행되는건지 정리해보려 한다.

일단 store buffer가 값을 cache에 업데이트하지 못하고, 기다리게 되는 이유는 크게 두가지가 있다. 바로 **Memory barrier로 인한 blocking**과 다른 코어로 부터 **invalidate acknowledge messages 수신을 대기**하는 경우이다. Invalidate Queue는 invalidate acknowledge messages delay를 완화시키기 위한 구조이다. 왜 delay가 생기고, 이를 어떻게 완화하는지 알아보자.

## Delayed by invalidate acknowledge messages

invalidate acknowledge messages가 지연되는 주된 이유중 하나는 invalidate acknowledge messages를 보내기 위해서는 실제 cache에 있는 cache block을 무력화 시켜야 하기 때문이다. cpu가 바쁘지 않을때는 상관이 없을수도 있지만, 만약 cpu가 cache의 정보를 지속해서 읽고 쓰는 상황에서는 이러한 무력화 작업이 지연될 수 있다.

<!-- 예시 그림 넣고싶은데 -->

## Invalidate Queues and Invalidate Acknowledge

우선 아래의 그림은 core 0의 캐시가 변수 b의 cache block을 소유하고, core 1의 캐시가 변수 a의 cache block을 소유한 상태에서 Invalidate queue가 적용된 시스템의 간단한 구조도이다. Invalidate Queue는 어떤식으로 작동하는지 간단하게 정리해보자.

![Cache system with Invalidate Queue](/assets/img/Invalidate_Queue.png)

Invalidate queue가 적용된 시스템에서는 특정 cache block에 대한 무효화 요청사항에 대해 실제로 무효화 하지 않고, Invalidate queue에 메세지를 넣기만 하고 무효화 되었다는 응답을 한다. 즉 해당하는 cache block은 무효화되었다는 응답이 나간 이후에 순차적으로 캐시에서 삭제된다.

## memory-misordering by Invalidate Queue

위의 Memory Barrier예제와 마찬가지로 아래 코드를 각각의 코드가 실행시키는 상황이라고 가정해보자.

```c
void foo(void) // execute by Core 0
{
    a=1;
    smp_mp()
    b=1;
}
void bar(void) // execute by Core 1
{
    while (b == 0) continue;
    assert(a == 1);
}
```

core 0의 캐시에 변수 b에 대한 cache block이 있고, core 1에는 a의 cache block이 있다고 가정하면 아래와 같은 순서로 실행되어 invalidate queue에 의해 데이터가 무효화되었다는 응답을 주었지만, 실제 무효화 작업은 이후에 진행되어 **execution order**와 **perceived order** 사이의 불일치가 발생할 수 있게 된다.

![memory-misordering by Invalidate Queue](/assets/img/missordering_by_invalidatequeue.png)

## **Invalidate Queues with Memory Barrier**

하지만 이러한 문제도 우리의 Memory Barrier와 함께라면 극복이 가능하다.이는 Memory Barrier가 동일하게 Invalidate Queue내부에서 데이터 수정 순서를 강제해 줄 수 있기 때문이다. 우선 각 core의 cache가 가진 초기 값은 동일하지만 아래와 같은 코드를 실행한다고 가정해보자.

```c
void foo(void) // execute by Core 0
{
    a=1;
    smp_mp()
    b=1;
}
void bar(void) // execute by Core 1
{
    while (b == 0) continue;
    smp_mp()
    assert(a == 1);
}
```

위 코드를 실행할때의 수행 순서는 아래의 TimeLine과 같이 진행된다. 즉 Memory Barrier가 invalidate queue에서 무효화가 진행되고 있는 항목에 대한 명령 수행을 blocking시키게 된다. 즉 **execution order를 통해 invalidate queue가 cache에 값을 업데이트 하는 순서를 강제**하여 **perceived order를 동기화** 할 수 있게 된다.

<br>

# **돌고돌아 Mutex**

다시 처음으로 돌아와서 두 프로세서가 동시에 뮤텍스를 걸 때는 어떻게 처리될까에 대해 답을 해보면 아래 그림과 같다.

![뮤텍스 문제]()

mutex는 read(플래그 확인) write(플래그 설정)가 한번에 이루어지는 원자명령이기 때문에 위 그림과 같이 뮤텍스가 걸릴 때 bus snooping을 통해 한쪽 명령 수행이 무력화 된다.