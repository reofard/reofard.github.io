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

![MESI state machine](/assets/img/MESISTATE.PNG)

MESI protocol에서 각각의 cache block은 위 사진과 같은 state를 가지게 된다. 상태전이 이벤트에 대해 간략히 설명하면 다음과 같다.

**Processor Requests Event**

- **PrRd**: 해당 cache block이 core에 의해 read연산이 진행되는 이벤트

- **PrWr**: 해당 cache block이 core에 의해 write연산이 진행되는 이벤트

**Bus side Event**

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

Consistency는 쉽게 말해 서로 다른 데이터 사이의 순서를 지키는 것이다. 이를 알기위해 먼저 Program Order와 Execution Order가 달라질 수 있음을 알아야 한다. Program Order는 작성된 프로그램에서의 실행순서, Execution Order 실제로 프로세서가 메모리에 접근하는 순서를 가르킨다. 근데 순서대로 실행하는데 바뀌는 이유는 바로 앞에서 설명한 cache-coherency protocol 때문이다.