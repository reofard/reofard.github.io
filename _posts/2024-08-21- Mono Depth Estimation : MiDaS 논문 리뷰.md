---
title:  "Mono Depth Estimation : MiDaS 논문 리뷰"
excerpt: "Paper Summary"

categories:
  - A.I

tags:
  - [Deep Learning, Computer Vision, Paper Review]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2024-08-21
last_modified_at: 2024-08-21
---

# **Midas<sup>Mixing Datasets for Zero-shot Cross-dataset Transfer</sup>**

파이 토치에 보면 인텔에서 개발한 MiDaS라는 단일 이미지에서 상대 깊이 추정을 위한 모델이 있다. [Towards Robust Monocular Depth Estimation: Mixing Datasets for Zero-shot Cross-dataset Transfer](https://arxiv.org/abs/1907.01341v3)라는 논문을 기반으로 만들어진 모델인데, 3D 영화를 포함한 다양한 소스의 데이터에 대해 사용 가능한 손실함수를 설계하여 zero shot learning을 하여 일반화 성능을 높였다고 한다. 

<br>

## **간단한 설명**

Mono Depth Estimation이란 단안카메라 영상/이미지에서 깊이를 측정하는 분야이다. 깊이를 추정하는데는 여러가지 방법이 있지만, 이번 논문에서는 r learning-based techniques 즉 신경망 모델을 이용해 구하는 방법에 대해 다루는데, 다른 분야들도 딥러닝 모델을 이용한다면 동일하긴 하지만 이 방법에는 몇가지 단점이 있다.

학습 모델을 법용적으로 사용하기 위해서는 일반화 성능을 높여야 하는데, 이를 위해서는 다양한 환경(영상을 찍은 환경, 카메라 등)에서 수집된 양질의 데이터가 많이 필요하다. 이러한 데이터셋들은 Razer Scanner, Lidar, SFM, Stereo Camera등을 통해서 만들어지는데, 해당 논문에서는 이런 모든 종류의 데이터를 사용하여 학습을 시켜 충분한 양의 데이터를 획득했다고 한다. 하지만 이 방식에는 문제가 있는데, 데이터마다 축적, bias(카메라와 거리측정 센서 사이의 차이 등), 카메라의 종류등이 모두 달라 한 모델에서 같이 학습시키기 어렵게 되는 것이다. 저자들은 해당 문제점을 해결하기 위해 새로운 손실함수를 제안했다.

또한 해당 논문에서는 데이터를 어떤 식으로 조합하여 학습 시킬지에 대한 전략<sup>Mixing strategies</sup>에 대해서도 제안 한다. 해당 내용은 서로다른 여러개의 Data Set을 하나의 모델로 학습시킬 때 성능을 높일 수 있는 전략에 대한 내용이다.

<br>

## **1. Scale- and shift-invariant losses**

학습에 사용되는 Data Set들은 아래 표와 같이 형식이 모두 다르다. 이런 다양한 Data Set을 학습에 사용하기 위해서 MiDaS에서는 학습 데이터를 적절한 출력공간으로 변형하여 계산 할 수 있는 손실함수인 **Scale- and shift-invariant loss function**을 제안했다.

![학습에 사용되는 Data Set](/assets/img/variant_data_set.png)

**Scale- and shift-invariant loss function**은 3가지 문제를 해결하기 위해 설계되었다고 한다.

1. Depth 표현방식이 반대인 문제
2. 사용 카메라의 파라미터등의 정보의 부재나 Ground Truth 계산 방식의 문제로 인해 Depth의 scale이 다르거나 모호한 경우
3. 카메라와 거리측정 센서의 위치 차이등으로 인해 영상과 depth값 사이의 bias가 존재하는 경우

우선 손실함수는 아래 식과 같이 정의된다.

> $\mathcal{L}\_{ssi}(\hat{\mathbf{d}},\hat{\mathbf{d}}^*) = \frac{1}{2M} \sum_{i=1}^M \rho(\hat{\mathbf{d}}_i-\hat{\mathbf{d}}^*_i)$


해당 식에 대해 설명하면 먼저 $\mathbf{d} = \mathbf{d}(\theta_{모델 파라미터})$와 $\hat{\mathbf{d}}^*$은 각각 이동 및 스케일링 연산이 적용된 예측 뎁스맵과 실제 데이터의 Ground Truth, $M$은 Ground Truth가 있는 픽셀의 수, 마지막으로 $\rho$는 특정한 손실함수 타입<sub>(=기존 MSE등의 손실함수)</sub>을 의미한다.
 
즉 위의 손실함수는 예측값과 Ground Truth에 변형을 가해 정렬할 뿐 일반적인 딥러닝에서 사용하는 손실함수와 동일하다. 그렇다면 정렬을 위한 이동 및 스케일링 연산 예상값은 어떻게 할까?

우선 $s, t$를 각각 이동, 스케일링 연산 예상값이라고 하고 다음과 같이 정의한다.

> $\hat{\mathbf{d}} = s\mathbf{d} + t, \hat{\mathbf{d}}^* = \mathbf{d}^*$$

$s$와 $t$ 를 구하기 위한 가장 먼저 생각 할 수 있는 식은 아래 식과 같은 최소 자승법이다.

> $(s, t) = argmin_{s, t} \sum_{i=1}^M(\hat{\mathbf{d}} - \hat{\mathbf{d}}^*)^2 $$

그 외에도 여러가지 방법을 통해 $s,t$를 구하여 손실함수를 정의할 수 있다는대, 해당 내용을 나중에 업데이트 하겠다.(수식쓰기 너무 힘듬)


## **Final Loss**

위와 같은 방식으로 손실함수를 정의한 후, 해당 논문에서는 [정규화 손실](https://arxiv.org/abs/1804.00607)<sup>Gradient Matching Term</sup>이라는 항을 추가하여 손실함수를 마무리 한다.

Gradient Matching Term의 목적은 Segmentation과 같이 이미지 상의 단일 표면에 대해 연속적으로, 경계면 사이에서는 뚜렷한 depth의 차이를 생성하기 위함이다. 이러한 목적을 달성하기 위해 저자들은 정규화 손실 항을 정의하였다.

> $\mathcal{L}\_{reg}(\hat{\mathbf{d}},\hat{\mathbf{d}}^*) = \sum\_{k=1}^K \sum\_{i=1}^M \left(\left\vert \nabla_x R_i^k \right\vert + \left\vert \nabla_y R_i^k \right\vert\right)$

해당 식에 대해 설명하면 $R_i^k = \hat{\mathbf{d}}_i^k-(\hat{\mathbf{d}}^*)_i^k$ 이미지를 k배 해상도로 샘플링 했을 때의 손실 즉 prediction 값과 Ground Truth의 차이이다.
해당 항에서는 앞서 설명한 손실값을 각각 x, y방향으로 미분한 그레디언트에 곱하여 합하게 되는데, 그 결과 이미지 상 경계에서는 손실이 크게, 경계가 아닌 표면에서는 손실이 작게 계산되게 된다.

그 결과 아래 사진과 같이 영상의 정보를 잘 활용하여 경계면을 잘 포착할 수 있게 된다. 아래 사진은 MiDaS 논문에서 인용한 [MegaDepth](https://arxiv.org/abs/1804.00607)라는 논문의 사진이다.

![Depth_Gradient](/assets/img/depth_gradient.png)

## **2. Mixing strategy**

아직 덜 읽었음


## 실제 구현
나중에 시간나면 간단하게 구현해보려고 한다.
[구현 참고 링크](https://gist.github.com/dvdhfnr/732c26b61a0e63a0abc8a5d769dbebd0)


## **후기**

해당 논문 읽으면서 좀 이해가 안갔던 부분이 몇가지 있었는데

1. Scaling, Transition 값을 모델을 통해 계산
2. Loss Function에 Scaling, Transition 값을 적용하여 계산
3. 계산된 Loss 를 이용하여 모델 학습

여기서 Scaling, Transition 값을 계산하기위해서는 학습된 모델이 필요한데, 학습에는 Scaling, Transition 값이 필요하니까 정확한 로직이 이해가 가지 않았는데, 실제 [구현 코드](https://gist.github.com/dvdhfnr/732c26b61a0e63a0abc8a5d769dbebd0)를 보니까 이해가 됐다.

일단 처음에 잘못 생각했던 부분은 Scaling, Transition 값이 훈련셋마다 고정된 값이라고 생각했었던 부분이었다. 이렇게 생각하니까 각 데이터셋마다 고정된 Transfomation 값이 필요하다고 생각했던건데, 실제로는 Scaling, Transition값은 단순히 모델의 아웃풋을 Ground Truth에 정합하는 값이기 때문에, 모델의 아웃풋 계산 직후 그때 그때 계산하는 값이었다.

MiDaS모델은 어디까지나 단일 이미지에 대한 모델이기 때문에 real World에서의 Depth를 구하는 모델이 아니다. 더군다나 학습을 시킬 때도 Predicion 결과를 Ground Truth를 기반으로 정합을 시켜버리고 나서 Loss를 계산하기 때문에 정말 상대적인 값만 계산이 가능한 모델이라 Real World의 값을 얻고싶다면 적절한 선택은 아닐 것 같다.

이와 별개로 이 논문이 대단하다고 생각한 점은 다양한 형식의 데이터셋을 간단한 아이디어인 최소자승법과 geometry transformation만을 이용해 Zero Shot Learning을 구현했다는 점이라고 생각했다. 나도 이렇게 듣고나면 간단해도 생각하기는 어려운 아이디어가 마구마구 나왔으면 좋겠당