---
title:  "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis 리뷰"
excerpt: "Paper Summary"

categories:
  - A.I

tags:
  - [Deep Learning, Computer Vision, Novel View Synthesis, Paper Review]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2024-08-28
last_modified_at: 2024-08-28
---

# **NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis**

최근에 회사에서 영상처리쪽 부분을 하게되면서 관련 내용들을 다시 찾아보고있다. 그러던 중 [NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis](https://arxiv.org/abs/2003.08934)라는 논문을 보게되었는데, camera Coordinate System, Vocel Rendering등의 지식을 기반으로 되게 단순한 신경망을 학습시키는데, 결과가 되게 잘 나오는데, 되게 특이하고 신기해서 한번 정리해보게 되었다. 뭐 나중에 찾아보니까 2022년 타임지 선정 최고의 발명품으로 선정도 되었다는데, 찾아서 들어가보니까 부문별로 5~6개씩 총 200개나 뽑던데....

<br>

## **간단한 설명**

이미지를 통해 현실 세계를 재구성하는 기술은 SFM(Structure From Motion), Visual SLAM(Simultanous Localization and Mapping)등이 있다. 해당 기술들은 2D 이미지들을 이용해서 3차원 오브젝트를 만들거나, Cloud Point를 만들어 현실세계를 재구성하게 된다.

하지만 이번에 살펴본 **NeRF**는 **Novel View Synthesis**기술인데, 해당 기술은 앞서 말한 것 처럼 3차원 공간을 재구성 하는 기술이 아닌, 특정 위치에서 바라본 3차원 공간의 이미지를 생성해내는 기술이다. 이러한 방식의 재구성 기법에는 몇가지 장점이 있다고 한다.

먼저 MLP Network를 사용하기 때문에 입력공간과 출력공간이 Continuous하기 때문에 연속적인 입력에 따른 매끄러운 output을 얻을 수 있다는 것이다. 두번째 장점은 추론에 사용되는 정보가 Volume Rendering방식이기 때문에 복잡한 현실세계의 형상을 효과적으로 나타낼 수 있고, 전통적인 신경망 학습방식인 경사하강법에 잘 맞다고 한다. 논문에서 말하는 마지막 장점은 3D모델정보를 따로 저장하고 있지 않기 때문에 저장공간에 대한 제약이 없다는 것이다.

이렇게 쓰고나서 보니까 좀 이해가안되는 부분이 있는데 아래 내용에서 하나씩 자세하게 톺아보도록 하자.

<br>

## **1. Neural Radiance Field Scene Representation**

NeRF는 연속적인 위치정보를 $\mathrm{x}=(x, y, z)$, 방향 정보 $\mathrm{d}=(\theta, \phi)$를 색상 $\mathrm{c}=(r,g,b)$와 밀도 $\sigma$로 맵핑하는 MLP Network $\mathit{F}_\Theta : (\mathrm{x}, \mathrm{d}) \rightarrow (c, \sigma)$를 학습한다. 여기서 중요한건 $\mathrm{x}$는 카메라 정보가 아니라 아래 그림과 같이 카메라 원점과 이미지상의 픽셀을 통과하는 ray위의 임의의 점, 그리고 $\mathrm{d}$는 해당 ray의 방향(해당 픽셀을 바라보는 방향)이라는 것이다.

**사진**

즉 학습시킨 신경망은 특정위치에 boxel이 존재할 확률(=$\sigma$)과 해당 boxel을 바라보는 방향에 따른 색($c$)을 예측하는 함수인 것이다. 이때 $\sigma$는 $\mathrm{x}$에 대해서만 예측하고, $c$는 $(\mathrm{x}, \mathrm{d})$ 를 사용하여 예측하는데 이는 아래 사진과 같이 특정 점을 보는 방향에 따라 색깔이 달라질 수 있지만, 해당 위치에 boxel이 있을 확률은 동일하기 때문이다.

**사진**

모델 구조설명해야되는데 귀찮네....

## **2. Implementation details**

우리가 가지고 있는 정보는 아래 사진과 같이 각각 찍은 위치와 방향을 알고있는 복수의 이미지들이다. 근데 학습에 필요한 정보는 임의의 점과 그 점을 바라보는 방향, 그리고 밀도와 색깔이다. 해당 정보는 어떻게 알아내야 할까? 이를 위해서는 카메라 좌표계애 대해 알아야 하는데, 이는 [이전에 쓴 글](www.naver.com)을 참고하도록 하자.

1. Ray 기반 샘플링
2. 샘플링 된 점의 밀도 <- SFM으로 해결
3. 색은 뭐 그냥 무조건 넣으면 댐....

## **3. Volume Rendering with Radiance Fields**

아직 안읽음

## **4. Hierarchical volume sampling**

아직 안읽음

<br>

## 실제 구현

[참고 코드](https://github.com/yenchenlin/nerf-pytorch/blob/master/run_nerf_helpers.py#L67)

<br>

## **후기**


이번 논문은 처음에 읽을때 헤깔리는 부분이 좀 있었는데, 가징 이해하기 난해했던 부분은 모델의 학습 입력과 아웃풋이었다. 지금은 알고보니까 Ground Truth가 Data Set을 SFM으로 전처리를 하여 특정 위치에서 계산된 밀도와 Ray에 매칭되는 색이라는것을 알지만, 이 SFM에 대해 언급하는 부분이 없어서, 샘플링 해서 input을 만드는거 까지는 이해가 됐는데, 아웃풋을 비교할 GT가 뭔지 몰라서 좀 많이 헤맸다. 또 처음에 저장공간이 얼마 없어도 되는게 큰 장점이라고 해서 이것보다 성능이 더 뛰어나고, 그래야 하는거 아닌가? 라고 생각을 했었는데 이것도 NeRF가 SFM결과 + 이미지를 이용해서 다시 학습을 통해 재가공 하는 기술이란걸 알고 나서야 이해가 됐던 것 같다. 이런 중요한 정보는 좀 앞에 적어놔야 더 읽기 편할거 같은데