---
title:  "shared_ptr vs unique_ptr 동작 차이"
excerpt: "Programming"

categories:
  - Programming

tags:
  - [Computer Science, C++, Programming]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2025-03-23
last_modified_at: 2025-03-23
---

# **이상한점 하나**
---
이번에 회사에서 업무중에 이해가 안되는 현상을 마주쳤었다. shared_ptr로 구현된 객체를 unique_ptr로 변경하니까, 기존에 호출되던 child class의 소멸자가 호출이 안되는 것이었다. 물론 내가 parent class의 소멸자를 virtual로 선언하지 않아 발생한 문제긴 했는데, 사실 이 상황에서는 shared_ptr일때는 어떻게 virtual도 아닌데 부모클래스의 소멸자가 호출되었는지에 대한 의문이 들었다. 왜냐하면, [virtual 소멸자를 가지지 않은 parent class를 상속한 객체를 parent class의 포인터가 가르킬 때, 소멸자는 원래 호출되지 않는것이 정상이라고 배웠었기 때문](https://modoocode.com/211)이다. 그래서 c++레퍼런스와 실제 unique_ptr과 shared_ptr의 구현체를 보면서 왜 이런 문제가 발생하는지 정리를 해봤다.

<br>

# **shared_ptr vs unique_ptr?**
---
일단 무엇이 저런 차이를 만들었는지에 대해 알아보기전에 각각의 스마트 포인터가 어떻게 작동하는지 간단하게 짚고 넘어가려고 한다. 해당 내용은 [모두의 코드](https://modoocode.com/252)의 내용을 참고하였다. 스마트 포인터는 다른 여러 블로그에서도 설명하고 있는 글이 많아서, 간단하게만 설명하고 넘어가려고 한다.

우선 스마트 포인터에대해서 간단하게 설명하면 다음과 같다. C++에서 함수 내부의 지역변수는 해당 함수가 호출될 때 할당(=생성자의 호출)되고, 함수가 종료될때 삭제(소멸자의 호출)된다. 이를 **stack unwinding**이라고 한다. 스마트 포인터는 이러한 지역변수의 특성을 이용하여, **지역변수로 동적할당된 포인터를 wrapping하고, 지역변수의 소멸자에서 동적할당된 포인터의 소멸자를 호출하는 객체**이다.

이러한 디자인 패턴은 C++의 창시장 비야네 스트로스트룹에 의해 제안되었다고 한다. 이는 흔히 **RAII(*Resource Acquisition Is Initialization*)** 라 불리고, 이는 자원 관리를 Stack에 할당한 객체를 통해 수행하는 방법을 의미한다.

C++ 에서 메모리를 잘못된 방식으로 관리하였을 때 크게 두 가지 종류의 문제점이 발생할 수 있다. 첫 번째는 **메모리를 사용한 후에 해제하지 않은 경우(memory leak)**이다. 특히 서버 처럼 장시간 작동하는 프로그램의 경우 시간이 지남에 따라 점점 사용하는 메모리 양의 늘어나서 결과적으로 나중에 시스템 메모리가 부족해져서 서버가 죽어버리는 상황이 발생할 수 도 있다. 이러한 문제점은 다행히 앞서 말한 RAII패턴을 사용하면 해결할 수 있다.

두 번째로 발생 가능한 문제는, 이미 해제된 메모리를 다시 참조하는 경우이다. 이렇게 **할당 해제 후, 재참조**문제를 해결하는 방식에 따라 스마트 포인터의 종류가 달라진다. 예를 들어 아래의 코드와 같이. 하나의 포인터를 두개의 변수가 가르키는경우를 상정해보자.

```c++
Data* data = new Data();
Date* data2 = data;

// data 의 입장 : 사용 다 했으니 소멸시켜야지.
delete data;

// ...

// data2 의 입장 : 나도 사용 다 했으니 소멸시켜야지
delete data2;
```

이 경우 같은 메모리 공간에 대해 두 변수가 소유권을 모두 갖고있어, 위 예제처럼 두번 할당해제하거나, 접근해제된 이후에 다른 포인터가 접근하는 경우 등의 문제가 발생할 수 있다. 이러한 문제를 해결하기 위해서 C++에서는 unique, shared, weaked_ptr등의 여러 스마트포인터를 제공하는데, 간단하게 알아보자.

### **unique_ptr**

unique_ptr은 동적할당된 특정 메모리 공간에 대해 유일한 소유권을 부여하는 객체이다. 즉 unique_ptr이 가르기고 있는 포인터는 절대로 다른 변수에서 복사, 접근이 불가능하고, 그렇기 때문에 unique_ptr이 갖는 메모리 공간은 해당 unique_ptr을 제외하고는 사용할 수 없어, 앞서 설명한 소유권이 명확하다.

```c++
  std::unique_ptr<A> pa(new A());

  // pb 도 객체를 가리키게 할 수 있을까?
  std::unique_ptr<A> pb = pa;럼
```

이러한 특징은 위 예제와 같은 코드를 실행시켰을때 ```'std::unique_ptr<A,std::default_delete<_Ty>>::unique_ptr(const std::unique_ptr<_Ty,std::default_delete<_Ty>> &)': attempting to reference a deleted function```와 같은 에러를 발생시키면서 애초에 컴파일 자체가 되지 않는다. 이는 unique_ptr은 이동생성자만 존재하고, 복사생성자는 명시적으로 삭제되었기 때문인데, 그렇기 때문에 ```std::move``` 등의 소유권 이전은 가능하지만, 앞선 예제에서 처럼 단순한 대입 연산을 통한 복사 생성자 호출은 에러로 처리가 되는것이다.

### **shared_ptr**

shared_ptr은 여러곳에서 참조가 가능한 스마트 포인터이다. 그렇다면 앞서 말한 **할당 해제 후, 재참조**문제를 어떻게 해결할까? shared_ptr은 단순하게, RAII 패턴으로 관리할 메모리 공간을 wrapping하지 않는다. 그렇기 때문에 shared_ptr은 여러곳에서 하나의 메모리 공간을 동시에 직접접근할 수 있게 해준다. 

![shared_ptr](/assets/img/shared_ptr.png)

shared_ptr은 위 그림처럼 해당 메모리 공간을 접근 가능한 모든 객체의 갯수를 모두 count한다. 그렇기 때문에 아래와 같은 코드에서 메모리를 공유하는 shared_ptr이 종료될때, 현재 참조중인 메모리의 reference count를 확인하고 1일때만 실제로 메모리를 할당해제한다. 즉 아래와 같은 코드에서 마지막 원소가 소멸할때 실질적으로 reference count가 1이되는 것이다.

```c++
std::vector<std::shared_ptr<A>> vec;

vec.push_back(std::shared_ptr<A>(new A()));
vec.push_back(std::shared_ptr<A>(vec[0]));
vec.push_back(std::shared_ptr<A>(vec[1]));

// 벡터의 첫번째 원소를 소멸 시킨다.
std::cout << "첫 번째 소멸!" << std::endl;
vec.erase(vec.begin());

// 그 다음 원소를 소멸 시킨다.
std::cout << "다음 원소 소멸!" << std::endl;
vec.erase(vec.begin());

// 마지막 원소 소멸
std::cout << "마지막 원소 소멸!" << std::endl;
vec.erase(vec.begin());

std::cout << "프로그램 종료!" << std::endl;
```

이러한 방식으로 shared_ptr은 현재 공유 메모리를 참조하는 객체가 1개라도 존재할때는 실제 메모리의 할당해제를 진행하지 않기 때문에, 할당 해제 후, 재참조 문제를 발생시키지 않는다.

<br>

# **문제상황**
---
우선 발생했던 문제부터 간단한 예제를 만들어서 확인해보자. 우선 아래와 같은 class가 정의되었다고 해보자문제가 발생한 코드는 아래와 같다. 간단하게 생각했을때 Child객체를 생성 후 Parent type으로 캐스팅 하고, 삭제하면 Child class의 소멸자가 호출될까? 정답은 "아니요" 이다. 왜냐하면 [C++에서는 virtual이 아닌 소멸자를 부모클래스에서 추론할 수 없기 때문](https://modoocode.com/211)이다. 이는 링크한 모두의 코드 블로그에서도 **"클래스를 상속할때 소멸자를 항상 가상클래스로 만들어야 된다"** 라고 강조하는 부분인데, 자세한 내용은 해당 링크를 참고하여 확인해보면 좋을 것 같다.

```c++
class Parent {
  public:
    ~Parent() {
      std::cout << "~Parent()" << std::endl;
    }
};

class Child : public Parent {
  public:
    ~Child() {
      std::cout << "~Child()" << std::endl;
    }
};
```

하지만 이번 포스트에서는 왜 가상소멸자를 사용해야하는지에대한 내용이 아닌, shared_ptr이 **"어떻게 가상소멸자가 아닌 소멸자를 호출하는지"** 이다. 그렇기 때문에 위와 같은 class가 있다고 하고 아래의 예제를 생각해보도록 하자.

```c++
Child* child = new Child();
delete child;
```

물론 child와 parent 클래스의 소멸자 모두 호출이 된다. 하지만 다음 예제에서는 다르다.

```c++
Child* child = new Child();
delete (Parent*)child;

std::unique_ptr<Parent> unique_child = std::make_unique<Child>();
delete unique_child;
```

이번 예제서는 **Parent의 소멸자만 호출**이 된다. 이는 Parent의 소멸자가 virtual이 아니기때문에, 삭제할 때, 해당 객체의 실제 타입을 추론할 수 없기 때문이다. **즉 이게 정상이다**. 이는 unique_ptr의 예시코드도 실행시켜보면 마찬가지로 Parent의 소멸자만 호출되는 것을 확인 할 수 있다. 하지만 shared_ptr은 다르게 움직인다. 아래의 예시는 shared_ptr의 예시 코드와 그 실행 결과이다. 이건 좀 충격적이라서 실행 결과까지 같이 작성한다.

```c++
std::shared_ptr<Parent> shared_child = std::make_shared<Child>();
shared_child.reset();
```

> 실행결과
> ```bash
> ~Child()
> ~Parent()
> ```

shred_child를 reset할때, shared_ptr이 자기 맘대로 실제 포인터의 자료형을 추론하여 Child의 소멸자를 호출하고 있는것이다. 이는 아까도 언급했다시피, 호출이 안되는게 정상이고, shared_ptr이 비정상 동작, 즉 unexpected behavior를 하고 있는 것으로 생각된다.

<br>

# **그럼 어떻게 shared_ptr은 소멸자를 호출할까?**

사실 이 문제의 답은 이미 [공식 레퍼런스](https://en.cppreference.com/w/cpp/memory/unique_ptr)에 있었다. 간단하게 설명하면, 먼저 **두 스마트 포인트가 동작하는 방식이 다른 결정적인 이유는 control block의 유무라고 보인다.** 먼저 공식 레퍼런스를 읽어보자.

> If T is a derived class of some base B, then unique_ptr\<T\> is implicitly convertible to unique_ptr\<B\>. The default deleter of the resulting unique_ptr\<B\> will use operator delete for B, leading to undefined behavior unless the destructor of B is virtual. Note that std::shared_ptr behaves differently: std::shared_ptr\<B\> will use the operator delete for the type T and the owned object will be deleted correctly even if the destructor of B is not virtual.


말 그대로 "unique_ptr\<T\>의 기본 Deleter는 T type에 대해서 소멸자를 호출하고, 이때, **T의 기저클래스의 소멸자가 virtual이 아니라면 기저클래스의 소멸자를 호출하지 않는다"**라는 내용이다. 뭐 맞는말이다. 하지만 여기서 봐야하는 부분은 ```std::shared_ptr behaves ...```이후 부분이다. shared_ptr은 virtual 소멸자가 아니더라도 정상적으로 소멸자를 모두 호출한다고 하는데, 이는 shared_ptr의 공식 레퍼런스를 보면서 알아보자.

unique_ptr의 공식 레퍼런스에서는 shared_ptr이 shared_ptr\<T\>가 shared_ptr\<B\>로 변환될 때도 T의 deleter를 사용하여 객체를 제대로 삭제한다고 한다. 어떻게 이게 가능할까? 먼저 아래 공식 문서의 일부를 읽어보자.

> In a typical implementation, shared_ptr holds only two pointers:
> - the stored pointer (one returned by get());
> - a pointer to control block. 
>
> The control block is a dynamically-allocated object that holds:
> - either a pointer to the managed object or the managed object itself;
> - the deleter (type-erased);
> - the allocator (type-erased);
> - the number of shared_ptrs that own the managed object;
> - the number of weak_ptrs that refer to the managed object. 

unique_ptr과 shared_ptr의 가장 큰 차이는 control block의 유무이다. 공식 레퍼런스에 따르면, control block에서는 deleter를 같이 저장한다고 한다. 이때 ```type erased```가 중요한데, 이는 shared_ptr이 여러 기반 클래스의 타입으로 복사되더라도, type에 영향을 받지 않고, control block에서 실제 삭제할때 사용할 deleter가 이미 shared_ptr의 생성 단계에서 결정이 될 수 있다는 뜻이다.

실제로 아래 사진과 같이 unique_ptr이 종료될때 아래 사진과 같이 그냥 저장해둔 포인터를 직접 delete하는것을 볼 수 있다.

![unique_ptr 소멸자 호출](/assets/img/delete_unique_container.png)

하지만 shared_ptr은 단순히 delete 연산만 수행하고 끝나는것이 아닌 추가적으로 저장된 deleter를 이용하여 작업을 수행하는것?을 볼 수 있다.(사실 대충 찍어서 이 부분이 아닐 수 도 있다.)

![shared_ptr 소멸자 호출](/assets/img/delete_shared_container.png)

<br>

# **후기**
---
진짜 C++을 하다가 별에별 문제를 모두 만나보는것 같다. 처음에 issue tracking을 할때는 답답하고 뭐가 문제인지 모르겠지만, 그렇다고해서 메모리릭이 발생하니까 가만히 냅둘 수도 없고 좀 스트레스를 받았던 것 같다. 하지만 이렇게 버그를 만날때마다, 깊게 파고들어서 정확히 무슨 문제가 발생한거고, 원인은 뭐였는지 알아보는 과정에서 C++를 더 자세히 알아가게 되는 기분이다.

Type Erasure는 이렇게 shared_ptr에서 reset할때 데이터형 추론뿐 아니라 std::function등의 로직에서도 사용되는 방식이라고 한다. 다음번 포스트에서는 어떤 방식으로 사용되고, 정확히 어떻게 작동하는건지 알아보려고 한다.