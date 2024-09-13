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

회사에서 개발을하다가 문득 이전에 잠깐 생각했던 궁금증 하나가 생각이났다. "멀티코어에서는 뮤텍스가 어떤방식으로 동작할까?"라는 질문이었는데, 구체적으로는 한 clock에 하나의 주소에 대한 두 프로세서의 요청은 어떻게 처리되는지가 궁금했는데, 찾아보니 의도치않게 여러가지 개념에 대해 다시 공부하고 정리하게 되었다.

# Cache Coherence

일단 Mutex도 메모리상에 존재하는 값이기 때문에 점유상태인지 아닌지 여부(is Locked)가 memory, 나아가 cache에 저장된다. 근데 Multi-Processor환경에서는 아래 사진과 같이 코어마다 cache가 존재한다.

![cpu 구조](/assets/img/cache.png)

그럼 두 프로세서가 cache에 각각 캐시에 저장된 값을 읽고 같은 clock에 mutex값을 바꾸면 안되는거 아니야? 란 생각이 들었었는데, cache의 일관성을 지키는 Cache Coherence Protocol이 존재했다. 즉 이 캐시는 실시간으로 하드웨어에 의해 동기화 되기 때문에 동시에 값을 수정하는게 물리적으로 불가능 한 것이다.

### **Bus snooping**

말그대로 Bus를 통해 오가는 데이터를 관찰하는 방법이다. Bus를 관찰하고 있다가 특정 주소에 대한 작업이 발생하면 캐시에 맵핑된 주소값에 대한 동작이 감지되면 해당 프로세서의 캐시에 업데이트가 아니더라도 같이 업데이트를 해주는 방식이다. 이 Bus snooping기법에는 두가지 방식이 있다.

- **Write-update** : 프로세서가 공유 캐시 블록에 write 작업을 하면 Bus snooping을 통해 다른 캐시의 모두 공유 캐시 블록의 값을 업데이트한다. 이 경우 Bus 내부에 bottleneck현상이 발생 할 수 있다.

  ![Write-update](/assets/img/BusSnooping_write.png)

- **Write-invalidate** : 프로세서가 공유 캐시 블록에 write 작업을 하면 Bus snooping을 통해 다른 캐시의 공유 캐시 블록에 invalid flag를 업데이트 한다.

  ![Write-invalidate](/assets/img/BusSnooping_tag.png)

메모리 계층구조에 의해 프로세스는 캐시에서 모든 정보를 불러와 처리를 하는데, 위에서 설명했던 것 과 같이 동일 cache block에 대해서 하드웨어적으로 일관성을 항상 유지하여 처리를 할 수 있게 해준다.


# **Transactional Memory**

그렇다면 이러한 동기화 문제는 캐시에서만 발생할까? 물론 여러 프로세서가 메모리에 동시에 접근하는것에 대한 해결책도 있다. 다만 이 방법은 메모리 구조가 계층적으로 변하면서 보편적인 개념은 아니게 된 듯 하지만 한번 정리해 보았다.

