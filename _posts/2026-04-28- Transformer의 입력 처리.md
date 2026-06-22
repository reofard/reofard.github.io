---
title:  "Transformer의 입력 처리"
excerpt: "Paper Summary"

categories:
  - A.I

tags:
  - [Deep Learning, Large Langugue Model, Paper Review]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2026-04-28
last_modified_at: 2026-04-28
---

# **비둘기 유도 미사일**

최근에 Claude, Cursor 같은 Large Langugue Model을 활용한 엔지니어링이 엄청나게 빠른 속도로 발전하고 있다. 이직한 회사에서도 Claude Code를 최고사양으로 구독할수 있게 해줘서 나름 하네스 엔지니어링도 이것저것 시도해보고, 잘 사용해보고 있다. 근데 이걸 사용하면서 느낀점은 **"기존에 LLM모델이 로컬환경의 API(Bash, Curl등)를 부르는 방법을 추가하고, 부를수 있도록 LLM을 학습시켜놓은것이지, 근본적으로 인공지능 기술이 발전한것은 아니지 않나?"** 라는 생각이 들었다.

세계2차대전 당시에 비둘기 유도 미사일이라는게 있었다고 한다. 당시 컴퓨터나 전자기술의 한계로 원하는 수준의 미사일 유도 시스템을 만들기 힘들었었는데, 누군가 비둘기에게 목표물을 인지시키고 그것을 가르키게만하고, 기계가 그 방향으로 나아가도록 조절한다면, 미사일도 유도할 수 있을것이라고 아이디어를 냈다고 한다.

![비둘기 미사일](/assets/img/비둘기미사일.png)

이 시스템에서 비둘기는 화면에서 배를 쪼도록 학습되고, 미사일은 화면상에서 비둘기가 쪼은 반대방향으로 움직이면 된다.

사실 현재 LLM이 처한 환경도 미사일속의 비둘기와 비슷하다고 생각한다. LLM은 학습한대로 output을 생성하고, 그 output에 따라 하네스가 api를 호출하고 그 결과를 LLM의 input으로 피드백하는것이다. 우리 모두가 시대에 의해 자의든 타의든 비둘기 사육사가 된 지금, 비둘기를 잘 알고, 돌보기위해서는 LLM자체를 기본적으로 이해하는게 중요한 것 같다.

물론 저렇게 거창하고 헛소리로 LLM공부를 시작한건 아니고 "Transformer의 연산량은 입력된 토큰시퀸스 길이의 제곱에 비례하는데, 토큰사용량은 입력의 길에에 비례할까?"라는 궁금증에 시작했다.

<br>

## **Transformer 구조**

ㅇㅇ

---

### **Tokenization**

dd

### **Embedding**

내용

## **Positional Encoding**

내용

<br>

## **후기**

내용