---
title:  "C++ MemoryLayout"
excerpt: "Programming"

categories:
  - Programming

tags:
  - [C++, Complier]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2026-09-24
last_modified_at: 2026-09-24
---

# **서버 프로세스 크래시**

이번에 회사에서 마일스톤 마감을 앞두고, **서버 프로세스가 동작 직후 크래시가 발생**하는 문제가 생겼었다. 콜스택 자체는 서버 모니터링 ui를 초기화 하는 과정에서 free를 하다 **손상된 heap에 의해 죽고있었다.** 그 말은 이전 로직에서 heap을 오염시키는 어떠한 로직이 있었다는말인데..... **문제는 죽기 직전 로직이 내가 최근에 만든 로직**이라는 것 이었다. 굉장히 난감하고, 실제로도 해당 부분이 범인으로 지목되었어서 굉장히 난감했었는데.... 결론적으로 내가 한 부분의 문제는 아니었다.

힙메모리 오염 특성상 누가 힙을 오염시켰는지 찾기 힘들었는데, 이번에 팀장님과 팀원분들께서 디버깅하시면서 찾은원인과 함께 이번 크래시 이슈에서 공부한 내용까지 함께 정리하여 공유해보려고한다. 우선, 크래시의 원인은 **컴파일러가 객체의정보를 잘못 구조화해서 멤버함수를 호출하는과정에서 프로그램카운터가 이상한곳으로 마구 튄게 문제**였다. ~~이걸 진짜 어떻게 눈치채고, 찾으셨을까....~~

<br>
<br>

---

# **C++ object model**

우선 c++에서 객체가 어떤식으로 표현되는지 알아보자. **객체는 메모리공간(주로 스택, 힙)영역에 존재하는 연속된 바이트 덩어리**이다. 그 바이트의 시작주소로부터 1~3번째 주소들은 첫번째 멤버변수의 값을, 4~7 두번째 멤버변수의 값을 저장하는식으로 객체는 메모리상에 존재한다. 이때 **각 객체가 타입에 따라 메모리 상에서 어떤 구조로 존재하는 결정하는것 구조를 객체 레이아웃 혹은 메모리 레이아웃이라고 한다**. 아래는 메모리상에서 오브젝트가 존재하는 방식을 간략화 해서 그려놓은 그림이다.

![메모리상에서 객체](/assets/img/object_in_memory.png)

> vector와 string은 일부 케이스를 제외하고, 실제 데이터를 힙에서 할당받아서, 스택영역에선 할당받은 힙영역의 접근 포인터만 캡슐화하여 존재한다.

그리고 위 그림과 같이 객체가 실제로 메모리에 담기는 규칙인 **클래스의 레이아웃은 컴파일타임에 결정**된다. 그렇게 결정된 레이아웃을 기반으로 프로세스는 저장된 오브젝트의 멤버에 정확한 위치에 접근할 수 있게 되는것이다.

<br>
<br>

---

## **Non-Polymorphic Class**

먼저 가상함수를 갖지 않은 Non-Polymorphic Class, 즉 비다형 클래스의 레이아웃 예시를 보면서 한번 이해해보자. 아래와 같은 구조체들이 정의되어있다고 가정해보자.

```c++
// 구조체
```

>클래스와 구조체는 본질적으로 동일하게 작동한다.

위와같은 자료구조는 아래 그림과 같이 트리로 상속구조를 나타낼수있다.

![구조체 상속 트리](/assets/img/struct_tree.png)

이러한 트리는 이제 실제로 메모리상에 올라가기위해 직렬화되어야 한다. 컴파일러는 클래스 상속구조 트리를 후위순회하며 순회중인 클래스의 멤버변수의 메모리상 위치를 저장하고 오프셋 포인터를 증가시킨다. 수도 코드는 다음과 같다.

```python
memlayout = arr()
offset = 0

layout(cls, offset):
    for parent in cls.parents :
        layout(parent)
    
    for var in cls.member_variable:
        memlayout.append({var.name, offset, offset+var.size() - 1})
        offset += var.size()
```

> 실제로 컴파일러가 각 클래스별로 이렇게 상속구조 트리를 순회하진 않는다. 일단 느릴뿐더러, 부모클래스가 없는 말단 클래스들부터 계산하면서 상위로 올라가면, 중복을 피할 수 있기 때문이다. 하지만 원리자체는 피보나치 수열을 점화식으로 계산하는지, 아니면 재귀적으로 계산하는지 차이라서 동일하다.

그리고 로직을 통해 직렬화된 클래스의 레이아웃은 아래 그림과 같이 계산된다.

![메모리상에서 직렬화된 클래스의 레이아웃](/assets/img/serialized_class.png)

즉 트리구조를 순회하면서 아래와 같이 직렬화된 실제 레이아웃을 계산할수 있는것이다. 하지만 레이아웃에는 멤버변수 뿐 아니라 가상함수를 호출하기위한 virtual table을 가르키는 vptr도 있어야한다.가상테이블에 대해서는 [모두의 코드 블로그](https://modoocode.com/211)를 참고하자. 이러한 virtual table은 다형 객체의 멤버함수를 호출하기 위해 필요한 정보라 직렬화된 객체 내부에 존재해야한다. 자세한 내용은 다음 목차에서 자세히 정리해보려 한다.

<br>
<br>

## **Polymorphic Class**

위 예제에서와 같이 단순한 클래스라면 모를까, Polymorphic Class는 부모 다형 클래스 포인터로 자식 객체를 가리키는 업캐스팅(Upcasting) 상황에서도 자식객체의 메모리 레이아웃과 멤버함수를 호출가능해야 한다. 그렇기 때문에 **virtual member를 가진 클래스의 객체는 virtual function table의 정보를 포함**해야 한다. virtual function table은 virtual member를 가진 클래스마다 하나씩 존재하고, 객체는 자신이 상속받은 Polymorphic Class에 대응되는  virtual function table를 참조하는 포인터인 vptr을 포함한다.

![업캐스팅 상황 vtable 로직](/assets/img/vtable_logic.png)

> override된 멤버함수도 virtual로 친다.

vptr은 member variable과 마찬가지로 객체에 포함되어야 한다. 그리고 child클래스의 객체는 **자신이 상속받은 Polymorphic Class 갯수 만큼의 vptr을 가져야 한다**는것과, 업캐스팅 상황에서 vtable에 접근하기 위해 **직렬화된 vptr이 객체상에서 부모 클래스의 멤버변수보다 빠르게 나와야 한다**는 것이다. 아래 그림은 다중 상속을 받은 Polymorphic Class의 객체 예시이다.

![다중 상속구조 vptr](/assets/img/multi_parent_class_vptr.png)

위 그림과 같이 **vptr은 "객체가 업캐스팅 된 경우 실제로 어떤 자식클래스에서 캐스팅된건지 결정해주는 정보"**가 된다. 그렇기 때문에 다중상속객체는 캐스팅 시에 base가 변경될 수 있다. 자신보다 앞선 레이아웃은 자신과 관계 없기 때문이다. 그러기 위해서는 멤버변수와 다르게, 자식객체의 부모를 향한 vptr은 항상 객체의 앞쪽에 위치해야하고, 상속트리 순환 과정에서 전위 순회를 통해 삽입되어야 한다.


다형성 클래스(Polymorphic Class) 객체는 **자신이 상속받은 "가상 함수를 포함한 부모 클래스"의 개수만큼 vptr을 보유**한다.또한, 업캐스팅 상황에서 별도의 주소 연산 없이 즉시 $O(1)$ 시간 내에 가상 함수 테이블(V-Table)에 접근하기 위해서는, 항상 업캐스팅된 부모를 가르키는 vptr이 레이아웃의 가장 첫번째에 배치된다. 아래 그림은 다중 상속을 받은 다형성 클래스 객체의 메모리 레이아웃 예시이다.

![다중 상속구조 vptr](/assets/img/multi_parent_class_vptr.png)

이처럼 **vptr은 "객체가 업캐스팅 된 경우 실제로 어떤 자식클래스에서 캐스팅된건지 결정해주는 정보"**가 된다. 그렇기 때문에 **부모 포인터로 업캐스팅되면 해당 부모에 대응되는 vptr의 위치로 포인터 주소가 이동**하게 되며, 컴파일러는 각 부모 파트의 최상단에 고유한 vptr을 배치하여 어떤 부모 타입으로 캐스팅되더라도 vtable 참조 및 가상 함수 호출이 가능하도록 보장한다. 반대로 다운캐스팅시에는 캐스팅한 타겟 클래스의 클래스 레이아웃에서 현재 클래스 타입의 vptr의 위치를 반대로 빼서 캐스팅한다. 그렇기 때문에 vptr은 앞선 구조체 상속트리 순회과정에서 전위 순회 부분에서 삽입되게 되고, 수도코드는 아래와 같다.

```python
memlayout = arr()
offset = 0

layout(cls, offset):

  for parent in cls.parents :
    if parent is not Polymorphic-Class:
      continue
    memlayout.append({parent.vptr, offset, offset + 8 - 1})
    offset += 8

  for parent in cls.parents :
    layout(parent)
    
  for var in cls.member_variable:
    memlayout.append({var.name, offset, offset+var.size() - 1})
    offset += var.size()
```
[ A_vptr | B_vptr ] [ A_멤버변수 | B_멤버변수 | Child_멤버변수 ] > [ A_vptr | A_멤버변수 ] [ B_vptr | B_멤버변수 ] [ Child_멤버변수 ]

아래 코드와 같은 상속구조가 존재할때 실제로 수도코드를 통해 어떻게 레이아웃이 생성되는지 정리해 보겠다.

```c++
// 구조체
```

아래는 수도코드로부터 파생되는 메모리 레이아웃이다. 실제

![다중 상속구조 vptr](/assets/img/multi_parent_class_layout.png)

<br>

---
<br>

# **후기**

이번 포스트에선 게임서버 프로세스를 죽인 heap corruption 이슈를 설명하기 위해 필요한 메모리상에서 객체가 어떻게 저장되는 방식, C++ Memory/Class Layout에 대해 알아봤다. 컴파일러 레벨에서 어떻게 클래스를 정의하고, 캐스팅이 어떻게 되는건지 근본 원리부터 다시 짚고 넘어 갈 수 있는 계기가 된 것 같다.