---
title:  "Generative Adversarial Network"
excerpt: "Deep Learning Summary"

categories:
  - A.I

tags:
  - [deep learning, machine learning, AI, School Summary]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2022-11-10
last_modified_at: 2022-11-10
---

# **GAN<sup>Generative Adversarial Network</sup>**

### 내쉬 균형

### 적대적 생성 신경망

- 생성자<sup>Generator</sup> 신경망과 판별자<sup>Discriminator</sup> 신경망을 zerosum환경에서 경쟁적으로 학습시켜 새로운 데이터를 생성하는 비지도 학습기반 생성모델
- zerosum 게임에서는 한 참가자가 입을 수 있는 최대손실을 최소화하는 전략이 최적의 전략

  ![GAN](/assets/img/Gan.png)

- 생성자는 판별자가 진짜라고 판별하는 가짜 이미지를 생성하도록 훈련
- 판별자는 생성자가 만든 이미지를 가짜라고 판별하도록 훈련

### GAN의 손실함수

- 판별자의 손실함수
  - 판별자는 진짜 데이터 분포 $P_x(x)$로 부터 관측된 진짜 데이터 $x$를 참으로 판별하고, 생성자가 노이즈 분포 $P_z(z)$에서 뽑은 노이즈 $z$로 생성한 데이터 $G(z)$에 대해 거짓으로 판별하도록 **?기대값 평균?** 방법으로 학습.   
    >$\Bbb{E}_{x\sim P_x(x)}[-\log D(x)] + \Bbb{E}_{z\sim P_z(z)}[-\log(1-D(G(z)))]$

- 생성자의 손실함수
  - 생성자는 노이즈 z를 통해 가짜 데이터 $G(z)$를 생성하고 판별자가 가짜데이터를 참으로 판별하도록 학습
    >$\Bbb{E}_{z\sim P_z(z)}[\log(1-D(G(z)))]$

- 생성자와 판별자는 각각 해당하는 손실함수를 최소화 하는 방향으로 학습을 진행하게 된다.
- 생성자의 손실함수는 판별자의 두번째 항과 부호만 바뀐 것이기 때문에 ZeroSum게임이 되게 된다.

# 기본 GAN의 문제점
- 비수렴성
  - 내쉬 균형 문제에서 경쟁적으로 학습하는것은 우수한 성능을 보이지만, 경사하강법은 특징공간이 볼록할때만 내쉬 균형을 보장하기 때문에 학습을 통해 내쉬 균형에 수렴되지 못하고 신경망의 매개변수가 진동
- 모드 붕괴
  - GAN은 판별자와 생성자가 서로 의존적으로 경쟁하는 최대/최소 문제이기 때문에 손실함수가 서로에게 의존적이기 때문에 명확하지 않다. 그렇기 때문에 생성자가 개선되는지 확인하기 어려워 생성자가 극소값에 빠져 같은 이미지만 생성하게 되는 모드 붕괴현상이 발생하기 쉽다.
- 기울기 소멸 문제<sup>수렴 실패</sup>
  - 판별자의 성능이 압도적이게 되면 생성자의 경사하강을 위한 기울기가 사라져서 학습이 전혀 이루어지지 못하고 수렴 실패가 발생한다.
  - 이 경우 생성자의 손실함수가 0에 수렴하고 판별자의 손실함수가 우상향 그래프를 그리게 된다. 
- 생성자, 판별자 사이의 불균형에 의한 과적합
- 하이퍼 파라미터에 매우 민감

# LSGAN/최소제곱 경쟁적 생경망
- 손실함수로 Least square loss를 사용하는 GAN

### Least square loss함수를 사용하는 이유
- 판별자는 결정경계<sup>decision boundary</sup> 오른쪽에 있는 데이터를 참으로 판단한다. 하지만 아래 사진에서 분홍색 별 데이터의 경우 실제 데이터 분포인 주황색 원과 멀리 떨어져 있음에도 참으로 인식되기 때문에 가짜 데이터임에도 불구하고 벌점이 부여되지 않음

  ![LSGAN](/assets/img/LSGAN.png)

- 손실함수를 Least square loss함수로 바꾸게 되면 아래 사진과 같이 결정평면과의 거리를 벌점으로 사용하게됨

  ![LSGAN](/assets/img/LSGANLOSSFunc.png)
### LSGAN의 손실함수

- 판별자 손실함수
  >$\Bbb{E}_{x\sim P_x(x)}[(D(x)-b)^2] + \Bbb{E}_{z\sim P_z(z)}[(D(G(z))-a)^2]$

- 생성자 손실함수
  >$\Bbb{E}_{z\sim P_z(z)}[(D(G(z))-c)^2]$

- LSGAN의 손실함수는 진짜데이터와 가짜 데이터에 대한 라벨로 $a$, $b$를 사용하고, $c$는 생성자가 만든 데이터를 판별자가 믿기를 바라는 값으로 사용된다.
- 데이터의 보다 정확한 분류를 하기 위해 기존 gan과 달리 라벨값을 $a$, $b$, $c$를 사용

# DCGAN
- 기존의 GAN에 CNN을 적용하여 고화질 이미지를 생성할 수 있는 모형
- 기존 GAN들보다 복잡한 이미지에 대해 안정된 결과를 보장

### 특징

- 판별자의 풀링층을 보폭이 2보다 큰 보폭을 사용하는 합성곱 층으로 대체
- 생성자의 풀링층을 보폭이 1보다 작은 분수 보폭을 적용하는 전치 합성곱층으로 대체
- 판별자와 생성자에 batch normalization 적용
- 완전 연결은닉층 제거
- 생성자는 출력층을 제외한 Layer에서 활성함수로 ReLU 사용, 출력층은 쌍곡탄젠트 함수 사용
- 판별자는 모든 층에서 Leaky-ReLU사용

### 전치 합성곱 연산
  - 이미지를 더 키우는 연산
  - 입력 이미지의 각 셀값은 필터의 크기만큼 특징도에 영향을 미치는 연산
  ![TransposeConv](/assets/img/TransposeConv.png)

- 합성곱 연산은 행렬의 곱으로 나타낼 수 있다(이거 외우기)

### 잠재공간<sup>노이즈 분포 공간</sup>활용
- 의미론적으로 생성자의 입력인 노이즈 벡터끼리 계산이 가능하다.
- 물론 잘 안됨


# W<sup>Wasserstein</sup>GAN
- Wasserstein 정보량을 이용한 gan
- discriminator를 critics라고 지칭
### Wasserstein 정보량
- 두 분포를 일치시키기 위한 최소 운동량
- WGAN에서는 generator가 생성하는 데이터 분포와 실제 데이터 분포가 있을 때 generator의 확률분포를 실제 데이터 분포와 동일하게 만들기 위한 최소 비용
  >$p_r = 실제 데이터 분포\\p_g = Generator가 생성하는 데이터 분포\\x = 실제 데이터 샘플\\y = G(z),생성자가 만든 가짜 데이터\\\gamma(x,y) = 특정 샘플 사이의 차이(분포 차이)\\\left\| x-y \right\| = 확률 사이 거리\\W(p_r,p_g) = \displaystyle\sum_{x,y}{\gamma(x,y)\left\| x-y \right\|}$

- **Kantorovich Rubintein duality**
  - 모든 $x,y$의 결합 분포를 계산하여 와셔슈타인 정보량을 구하는 것아 불가능 하기 때문에 주변확률 분포를 구하기 위해 사용한다.
  - 와셔슈타인 정보량 $W(p,q)$과 K-lipchitz 쌍 $KR(p,q)$는 항상 $KR(p,q)\le W(p,q)$를 만족하기 때문에 아래와 같이 $W(p_r,p_g)$를 $KR(p,q)$의 최대값<sup>$\sup$</sup>으로 대체
    > $
      W(p_r,p_g) = \frac{1}{K} \sup \Bbb{E}_{x\sim p_r}[f(x)]-\Bbb{E}_{x\sim p_g}[f(x)]\\
      이때\ f(x)는\ \left\| f \right\|_L \le K를\ 만족
    $
  - 주로 $K$는 1으로 사용

### 손실함수
- WGAN에서는 $W(p_g,p_r)$을 구하기 위해 위 1-lipchitz식의 값을 구하기 위해 $f(x)$를 경사하강법을 통해 근사
  > $W(p_r,p_g) = max_{w\in W} \Bbb{E}_{x\sim P_x(x)}[f_w(x)]-\Bbb{E}_{z\sim P_z(z)}[f_w(G_\theta(x))]$
- 위 식에서 $\left\| f \right\|_L \le 1$를 만족시키기 위해 특정한 값 c로 가중치 $w$를 $[-c,c]$ 범위로 제한

- **판별기<sup>critics</sup> 손실함수**
  - 두 확률 분포 사이의 거리를 최대한 키우기 위해 $-W(p_{data},p_g)$를 최소화
    > $L^(D) = -\Bbb{E}_{x\sim p_{data}(x)}D_w(x)+\Bbb{E}_zD_w(G(z))$

- **Generator 손실함수**
  - 두 확률 분포 사이의 거리를 최대한 좁히기 위해 $W(p_{data},p_g)$를 최소화
  - 실제 데이터는 Generator와 관련이 없으므로 삭제
    > $L^(G) = -\Bbb{E}_zD_w(G(z))$


# C<sup>Condition</sup>GAN
- 데이터 생성시 원하는 이미지 생성 할 수 있는 조절 기능 추가
- 기본 GAN에서 label정보가 들어간 GAN
- 판별자를 학습시킬대 특정 라벨의 데이터만 추출하여 학습]
- 생성자의 입력 잡음분포생성 수 뒤에 라벨 정보를 붙여서 생성자의 입력으로 사용한다.
  - 라벨 정보는 숫자이고 잡음분포는 실수인데 이거 그대로 붙이나?

# InfoGAN
- 데이터 생성시 원하는 이미지 생성
- 판별자를 건들진 않음
- 생성자 입력에 잡음벡터와 잠제코드 c의 결합에 기초하여 가짜 데이터 생성
- 잠재코드 c가 데이터에 존재하는 특징을 가질 수 있게 학습