---
title:  "Multi-core에서의 Multi Processing"
excerpt: "Computer Architecture"

categories:
  - Operating_System

tags:
  - [Computer Science, Operating System]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2023-05-27
last_modified_at: 2023-05-27
---

# 갑자기 든 궁금증

회사에서 개발을하다가 문득 이전에 잠깐 생각했던 궁금증 하나가 생각이났다. "멀티코어에서는 뮤텍스가 어떤방식으로 동작할까?"라는 질문이었는데, 구체적으로는 한 clock에 하나의 주소에 대한 두 프로세서의 요청은 어떻게 처리되는지가 궁금했는데, 찾아보니 의도치않게 여러가지 개념에 대해 다시 공부하고 정리하게 되었다. 참고한 자료는 [Memory Barriers: a Hardware View for Software Hackers](http://www.puppetmastertrading.com/images/hwViewForSwHackers.pdf)를 참고하였다.

<br>

# Coherency

일단 Mutex도 메모리상에 존재하는 값이기 때문에 점유상태인지 아닌지 여부(is Locked)가 memory, 나아가 cache에 저장된다. 근데 Multi-Processor환경에서는 아래 사진과 같이 코어마다 cache가 존재한다.

![cpu 구조](/assets/img/cache.png)

그럼 두 프로세서가 cache에 각각 캐시에 저장된 값을 읽고 같은 clock에 mutex값을 바꿔버리면 원자명령이 의미 없는게 아닐까? 란 생각이 들었다. 여러 cache가 동기화되지 않으면 해당 데이터가 접근할 때 최신이라는 보장인 Coherency가 없기 때문이다. 하지만 찾아보니 데이터에 접근할 때 해당 데이터가 최신임을 보장해주는 Cache-coherency Protocol이 존재했다. 이 프로토콜은 cache를 실시간으로 하드웨어에 의해 동기화하여 동시에 값을 수정하는것을 물리적으로 불가능 하게 만든다. 그럼 어떤방식으로 하드웨어에서 Coherency를 유지해주는지 알아보자.

### **Bus snooping**

말그대로 Bus를 통해 오가는 데이터를 관찰하는 방법이다. Bus를 관찰하고 있다가 특정 주소에 대한 작업이 발생하면 캐시에 맵핑된 주소값에 대한 동작이 감지되면 해당 프로세서의 캐시에 업데이트가 아니더라도 같이 업데이트를 해주는 방식이다. 이 Bus snooping기법에는 두가지 방식이 있다.

- **Write-update** : 프로세서가 공유 캐시 블록에 write 작업을 하면 Bus snooping을 통해 다른 캐시의 모두 공유 캐시 블록의 값을 업데이트한다. 이 경우 Bus 내부에 bottleneck현상이 발생 할 수 있다.

  ![Write-update](/assets/img/BusSnooping_write.png)

- **Write-invalidate** : 프로세서가 공유 캐시 블록에 write 작업을 하면 Bus snooping을 통해 다른 캐시의 공유 캐시 블록에 invalid flag를 업데이트 한다.

  ![Write-invalidate](/assets/img/BusSnooping_tag.png)

간단하게 설명하면 위와 같은 방식으로 작동하는데 여러 프로토콜중 MESI 프로토콜에 대해 알아보자.

<br>

### **MESI protocol**

MESI protocol에서 각각의 cache block(참고 논문에선 cache line이라고 함) “modified”, “exclusive”, “shared”, 그리고 “invalid” 총 4가지 state로 구분된다. 물론 더 많은 상태가 있지만 이것만 봐도 된다고 한다.

- **modified** : 해당 core가 해당하는 cache block에 대한 write연산을 가장 최근에 한 경우이다. 이 경우 해당 core에서 가장 최근의 값을 수정했기 때문에 해당하는 cache block의 유일한 최신 사본을 갖고있는 상태가 된다.

- **exclusive** : 특정 cache block이 메모리로부터 처음 복사되어 다른 cache는 아무도 갖고있지 않은 상태이다. 즉 해당 cache block이 해당 cache에만 존재한다는 의미로 core가 값을 수정하면 책임지고 memory에 변경사항을 저장해야 하는 상태이다.

- **shared** : 해당하는 cache block이 모두 최신 상태이고, 다른 cache와 공유된 상태를 의미한다. 이 경우 cache block을 수정하기 위해서는 다른 core와 상의가 필요한 상태이다.

- **invalid** : 해당 cache block이 최신사본이 아닌경우이다. 다른 core의 cache에서 해당 cache block이 수정되는 경우 해당 상태로 진입하고, 해당 cache block이 없는 상태이다.

![MESI state machine](/assets/img/MESISTATE.png)

MESI protocol에서 각각의 cache block은 위 사진과 같은 state를 가지게 된다. 상태전이 이벤트에 대해 간략히 설명하면 다음과 같다.

### **Processor Requests Event**

- **PrRd**: 해당 cache block이 core에 의해 read연산이 진행되는 이벤트

- **PrWr**: 해당 cache block이 core에 의해 write연산이 진행되는 이벤트

### **Bus side Event**

- **BusRd**: 다른 core에서 특정 캐시블럭에 대한 읽기 요청이 발생한 경우

- **BusRdX**: 보유중인 cache block에 대해 다른 core에서 쓰기 요청이 발생한 경우.

- **BusUpgr**: 가지고 있지 않은 cache block에 대해 다른 core에서 쓰기 요청이 발생한 경우.

- **Flush**: 전체 cache block에 대해 다른 core에서 쓰기 요청이 발생한 경우. 해당 변경 내용과 같이 다른 캐시에 전달하는 메세지

- **FlushOpt**: 특정 cache block에 대해 다른 core에서 쓰기 요청이 발생한 경우. 해당 변경 내용과 같이 다른 캐시에 전달하는 메세지

<br>

메모리 계층구조에 의해 프로세스는 캐시에서 모든 정보를 불러와 처리를 한다. 그렇기 때문에 같은 메모리 주소(위에서 의문으로 제기했던 멀티코어에서 하나의 뮤텍스)는 하나의 cache에 대응되고, 위에서 설명한것 과 같이 동일 cache block에 대해서 하드웨어적으로 항상 최신의 값이라는 보장을 해주어 Coherency를 지키게 된다.

<br>

# **Consistency**

그렇다면 이러한 일관성 문제는 모두 해결 된걸까? 위에서의 Cache Coherency Protocol은 여러개의 cache 저장소의 일관성을 유지하기 위한 프로토콜이었다. 하지만 메인메모리(RAM)는 하나기 때문에 항상 메모리가 최신이라는 보장인 일관성(Coherency)에 대해 신경 쓸 필요가 없다. 여기서 또 다른 개념 하나가 더 등장하는데, 바로 Consistency이다. 이 단어도 마찬가지로 일관성이라는 뜻을 가지지만 위에서 다룬 의미와는 조금 다르다.

Consistency는 서로 다른 데이터 사이의 순서를 지키는 것이다. 쉽게 말해 Program Order와 Execution Order의 순서가 일치함을 의미한다. Program Order는 작성된 프로그램에서의 실행순서, Execution Order 실제로 프로세서가 메모리에 접근하는 순서를 가르킨다. 

하지만 Consistency는 바로 앞에서 설명한 cache-coherency protocol에 의해 깨질 수 있다(물론 pipelining을 위한 최적화에 의해서도 깨질 수 있음). 일단 이번 포스트에서는 cache-coherency protocol에 관련해서 Consistency에 문제가 생기는 원인 만 알아보려고 한다.

![cache write stall example](/assets/img/cachestall.png)

위 사진은 CPU 0(이하 0번 core)이 특정 cache블럭에 대한 write연산을 하기위해 CPU 1((이하 1번 core))의 cache block을 무효화 하는 과정을 나타낸 그림이다. 이와 같이 하나의 코어가 write연산을 하기위해서는 다른 코어에 의해 허락을 받아야 하는 불필요한 지연 시간이 생기게 된다.

<br>

### **Store buffer**

현대 컴퓨터 구조에서는 이러한 불필요한 지연을 해결하기 위해 아래 그림과 같은 **Storer buffer**를 도입하였다. Store buffer는 현재 아래 그림과 같이 core와 cache사이에 존재하는 버퍼이다. 이 버퍼는 write연산 결과를 잠깐 저장했다가 core대신 Acknowledgement 메세지를 받아 캐시에 적어주는 역할을 한다. 그럼 core는 굳이 Acknowledgement를 기다리지 않고 다른 작업을 하면 된다.

![store buffer](/assets/img/storebuffer.png)

<br>

### **self-consistency violation**

하지만 store buffer의 도입에 따라 새로운 문제가 발생한다. 첫번째 문제는 **self-consistency violation**이다. 먼저 아래 사진과 같은 상태의 시스템을 생각해보자.

![self-consistency violation](/assets/img/selfviolation.png)

이 상황에서 0번째 코어가 아래와 같은 코드를 수행한다면 정상적으로 b가 2가 되지 않을 수 있다. Cache 0에서 a=1을 업데이트하기 위해서는 1번 core의 허락을 맡아야 하는데, 허락을 맡기위해 a=1결과가 store buffer에 체류하는 동안 두번째 줄을 실행하면 a가 아직 0인 상태로 계산되기 때문이다.

```c
a = 1;
b = a + 1;
assert(b==2);
```

![self cosistency violation timeline](/assets/img/selfviolationtimeline.png)

위 사진은 self-consistency violation이 일어나는 타임라인에 대해 그린 그림이다. 앞 명령어의 stall 기간 동안 뒤의 명령어가 앞의 명령의의 결과를 사용하는 경우에 주로 self-consistency violation이 발생할 수 있다.

물론 위 문제의 해결책은 존재한다. 각각의 core가 자신의 store buffer에서 load 중인 cache block을 조회 할 수 있게 하면 된다. 이를 **Caches With Store Forwarding**이라고 한다. 아래 사진은 Caches With Store Forwarding의 간략한 구조와 적용되었을때 위 문제사항에서 실행 플로우이다.

![Timeline Caches With Store Forwarding](/assets/img/store_buffer_timeline.png)

<br>

### **violation of global memory ordering**

두번째 문제는 **violation of global memory ordering**이다. 아래 사진과 같이 캐시에 데이터가 저장되어있고, 아래의 코드를 각각의 프로세스가 실행하는 상황이라고 가정해보자.

![violation of global memory ordering](/assets/img/globalviolation.png)

```c
void foo(void) // execute by Core 0
{
    a=1;
    b=1;
}
void bar(void) // execute by Core 1
{
    while (b == 0) continue;
    assert(a == 1);
}
```

이 경우 아래 그림의 타임라인과 같이 실행 순서가 바뀌는 문제가 발생 할 수 있다. 이는 앞서 말했던 **Consistency** 문제로 서로 다른 core사이에서 특정 변수들의 업데이트 순서를 관찰 할 때, 실제 코드의 실행순서와 다르게 관찰 될 수 있게 된다. 이러한 문제를 해결하기 위해서 프로그래머는 **Memory Barriers**라는 기능을 사용할 수 있다.

![violation of global memory ordering timeline](/assets/img/.png)

<br>

### **Memory Barriers**

그럼 Memory Barriers는 뭐고, 어떻게 쓰는걸까?