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

```smp_mp()```의 적용으로 인해 아래와 같이 실행되게 되고, execution order와 perceived order를 일치 시킬 수 있게 된다.

<br>

## **Invalidate Queues**

하지만 store buffer는 CPU가 데이터를 메모리에 기록하기 전에 잠시 저장해 두는 작은 임시 저장소이다. 그렇기 때문에 캐시 미스나 캐시 무효화(invalidation) 작업에 의해 메모리에 기록하는 과정이 지연되며 buffer가 가득 차버리는 문제가 발생할 수 있다. 이러한 문제를 해결하기 위해 **Invalidate queue**라는것을 도입해서 문제를 해결했는데, 어떻게 수행되는건지 정리해보려 한다.


# **돌고돌아 Mutex**

다시 처음으로 돌아와서 두 프로세서가 동시에 뮤텍스를 걸 때는 어떻게 처리될까에 대해 답을 해보면 아래 그림과 같다.

![뮤텍스 문제]()

mutex는 read(플래그 확인) write(플래스 설정)가 한번에 이루어지는 원자명령이기 때문에 위 그림과 같이 뮤텍스가 걸릴 때 bus snooping을 통해 한쪽 명령 수행이 무력화 된다.