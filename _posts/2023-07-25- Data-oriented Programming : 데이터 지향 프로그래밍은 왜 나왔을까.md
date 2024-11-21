---
title:  "Data-oriented Programming : 데이터 지향 프로그래밍은 왜 나왔을까?"
excerpt: "Programming"

categories:
  - Programming

tags:
  - [Data-oriented Programing, Programming paradigm]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2023-07-25
last_modified_at: 2023-07-25
---

# 데이터 지향 프로그래밍이란?

의도한건 아니었지만, [이전 포스트](https://reofard.github.io/operating_system/2023/05/27/Multi-core%EC%97%90%EC%84%9C%EC%9D%98-Multi-Processing.html)에서 캐시에 대해 다루었었다. 관련된 내용을 다루면서 참고했던 [논문](https://www.puppetmastertrading.com/images/hwViewForSwHackers.pdf)에서 하드웨어 개발자들이 캐시를 도입하면서 생긴 문제에 대해 얼마나 힘들게 해결했는지 볼 수 있었다. 그렇기 때문에 우리 소프트웨어 개발자들이 캐시를 적극적으로 고려해서 프로그래밍을 하면 하드웨어 개발자들도 만족하지 않을까? 이러한 접근에서 나온 개발 패러다임이 있는데, 바로 **DOD(Data-oriented Design)** 이다. 어떤식으로 해야 데이터를 더 효과적으로 다룰 수 있을지 한번 간단히 알아보자.

<br>

우선 **DOD(Data-oriented Design)**의 기본적인 전제는 **우리<sup>개발자</sup>가 다루는 모든 것은 데이터**라는 것이다. 컴퓨터에서 보이는 이미지도, 음악을 저장한 mp3파일도, 게임에서 보이는 입자 하나하나 심지어는 명령어까지도 모두 하드웨어 어딘가에 저장되어있는 데이터다. 그리고 이 데이터들은 모두 하드웨어에 의해 사용자에게 유용하게 변환되어야 한다. 그렇기 때문에 **DOD(Data-oriented Design)**는 **하드웨어에서 데이터의 변환을 잘 유도**할 수 있도록, **데이터와 소프트웨어 구조를 구성**하여 **개발**하는 방법론에 대해 다룬다.

## **1st principle: Data is not the problem domain**

<div style="margin-left: 20px;">

<b>DOD(Data-Oriented Design)에서 데이터는 문제 영역의 개념과 연관되지 않는다.</b> 데이터 지향 설계는 문제 영역을 코드에 반영하지 않고, 데이터를 그저 처리할 정보의 단위로만 다룬다. 이는 객체 지향 설계처럼 클래스와 기능을 통해 데이터를 특정 의미나 맥락과 묶지 않으며, 그로 인해 <b>데이터</b>와 <b>문제 영역</b>의 <b>결합을 최소화</b>한다. 즉 DOD에서 데이터는 프로그램의 동작에 필요한 정보일 뿐이며, 설계 문서에 문제 정의를 남겨두고 프로그램에서는 데이터의 효율적 처리를 최우선 목표로 해결하는 것이다.


</div>

## **2nd principle: Data is type, frequency, quantity, shape and probability**

<div style="margin-left: 20px;">

DOD(Data-oriented Design)에서 Data를 다룰때는 단순한 구조로 다루지 않는다. 이는 cache hit rate를 극대화 하기 위한 데이터 구조는 DOD의 극히 일부분이라는 뜻이다. DOD는 더 나아가 <b>프로그램이 실행</b>될 때 <b>데이터가 어떻게 사용되는지</b>를 고려한다. 그러기 위해서 <b>데이터의 타입, 빈도, 양, 형태, 확률</b>에 대해 이해하고, 하드웨어의 동작방식을 고려하여 프로그램을 설계하게 되는 것이다.

</div>

<br>

그렇다면 이런 원칙에 의해 기존의 소프트웨어 설계 패러다임과는 어떤 차이가 있는지 알아보자. 대중적으로 널리 쓰이는 Object-oriented Design와 비교해보자.

<br>

# **Object-oriented design과 Data-oriented design의 차이**

우선 OOD(Object-oriented design)의 한계점에 대해 간단하게 정리하고, 이를 어떻게 DOD(Data-oriented design)에서 극복하고자 하는지 간단하게 정리해보자. 

## **데이터와 연산의 분리**

![OOD vs DOD](/assets/img/Data_login_rel.png)

<div style="margin-left: 20px;">

<b>OOD</b>는 <b>문제 영역과 구현을 강하게 결합</b>하는 특성 때문에 <b>"관성"이 발생</b>한다고 한다. 이러한 특성은 <b>설계 시 직관적</b>일 수 있지만, 데이터를 분리하거나 연산을 변경하는 일이 어려워 <b>설계 변경에 따른 수정 작업이 복잡</b>해지는 경향이 있다.

반면에 DOD는 <b>데이터를 한 곳에 모으고</b>, 데이터에 대한 <b>연산은 별도로 작성</b>한다. 그렇기 때문에 데이터의 변화를 통해 설계 변경 사항을 반영하며, <b>데이터와 변환(연산)</b>을 더 쉽게 결합하거나 분리할 수 있어 <b>유연성이 더 높다</b>고 한다. 하지만 연산에 필요한 <b>데이터 관리</b>에서 <b>추가적인 비용이 발생</b>할 수 있다.


</div>

## **데이터는 실제 세계에서 제너릭하지 않다**

<div style="margin-left: 20px;">

DOD에서는 세상 모든 <b>데이터</b>를 type에 관계없이 <b>Generic하게 다룰 수 없다</b>는것을 인정한다. 예를들어 string데이터 압축과 이미지 데이터의 경우 각각 데이터의 특징덕분에 서로 다른 방식과 다른 압축결과물(Huffman Coding, discrete cosine transform)을 갖는것이 효율적이다. 그렇기 때문에 <b>데이터</b>는 <b>타입에 따라 각각 적절하게 사용</b>되는 방식이 달라야 하고, 이런 특징은 종종 <b>하드코딩 사용</b>을 허용한다.

</div>

<br>

결국 모든 Design은 소프트웨어의 복잡도를 감소시키고자 한다. 그렇기 때문에 DOD와 OOP중 "무엇이 더 낫다, 더 개선된 패러다임이다" 이렇게 단정할 수는 없다. 하지만 DOD는 OOD에서의 일부 불편한점을 해결하기 위해 나온 패러다임이라는 것을 알 수 있다. 하지만 그 과정에서 다시 약점이 생기기도 하기 때문에, 상황에 따라 적절하게 사용하는게 중요하다.

<br>

# **마치며**

사실 이번 토픽은 컴퓨터구조를 잘 공부했으면 당연히 아는 상식선이라고 생각했었다. 당장 백엔드 개발자의 단골 면접 질문중에 "아래 두 코드 중 뭐가 더 빨라요?"도 있기 때문이다.

```c
for(int i=0; i<100; i++)
{
    for(int j=0; j<100; j++)
    {
        sum+=d[i][j];
    }
}
```

```c
for(int i=0; i<100; i++)
{
    for(int j=0; j<100; j++)
    {
        sum+=d[j][i];
    }
}
```

하지만 기본이라고 생각했던 지식에 체계화된 개발론, 심화 내용의 책까지 존재하는걸 보니 좀 생각이 많아지면서 한번 간단하게 정리해보게 되었다. 앞으로도 공부하며 관련 내용을 작성해보려고 한다.