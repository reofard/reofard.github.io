---
title:  "Transformer: Self-Attention"
excerpt: "Paper Summary"

categories:
  - A.I

tags:
  - [Deep Learning, Large Langugue Model, Paper Review]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2026-05-10
last_modified_at: 2026-05-10
---

---
# **맥락 섞기**

현재 상용 LLM 모델(Claude, jemini, GPT 등)들은 거의 대부분 트랜스포머를 핵심 기반으로 삼고 있다. 이번 포스트에서는 그런 트랜스포머의 핵심 파트중 Transformer Block에 대해 알아보려고 한다. 불과 몇년 전까지만해도 2~3년 전까지만 해도 텍스트를 생성하기 위해 RNN을 사용했었는데, 그때 생성되는 글의 품질이나 맥락이 그렇게 좋지 않다는 느낌을 강하게 받을 수 있었다. 하지만 트랜스포머가 나오고 LLM모델이 상용화되면서 만년 2000원에서 벗어나지 못하던 코스피가 메모리사업을 하는 삼성과 SK하이닉스의 견인을 받아 5000원을 넘을정도의 파급력을 전세계에 펼쳤다. 도대체 뭐가 박스피를 박살낼말큼 혁신적이었는지 이번 포스트에서 알아보자

<br>

---
# 1. 간략한 앞선 내용

이전 글에서 Transformer구조에서 Self-Attention 직전까지의 과정을 간략하게 알아봤다. 앞선 설명에서 다음과 같은 수식을 기반으로 행렬 $X$로 나타냈고, $X$의 각 행 $X_1, X_2, \cdots, X_N$은 입력된 토큰의 의미 벡터를 나타낸다는 것을 수식으로 알아보았다.

$$
E_i = \mathrm{Embedding}(x_i),\qquad X_i = E_i + PE_i \in \mathbb{R}^{d}
$$

$$
\mathbf{X} = \begin{bmatrix} X_1 \\ \vdots \\ X_N \end{bmatrix} \in \mathbb{R}^{N\times d}
$$

이제 이렇게 구해진 행렬 $X$를 기반으로 트랜스포머가 문맥정보에 따라 어떻게 값을 섞는지 확인해보자.

<br>

---
# 2. Transformer Block

트랜스포머 블럭은 토큰의 정보를 섞어 특정 토큰이 맥락 안에서 어떤 정보를 갖는지 정보를 계산해낸다. 이러한 계산을 위해 self-attention, feed-forward network에 대해 알아야하는데 한번 자세히 알아보자.

## Self-Attention

Self-Attention단계에서는 각각의 토큰이 주변의 관련된 토큰의 정보를 흡수하여, 주어진 입력 컨텍스트 내에서 실제로 어떤 의미인지를 추론해내는 단계이다. 이를 위해 아래 식과 같은 과정을 거치는데, 하나씩 수식을 살펴보자

$$
\mathbf{X}\xrightarrow{\times\mathbf{W}_{Q,K,V}}\mathbf{Q},\mathbf{K},\mathbf{V}
\xrightarrow{\mathbf{Q}\mathbf{K}^\top/\sqrt{d_k}}\mathbf{S}
\xrightarrow{\mathrm{softmax}}\mathbf{A}
\xrightarrow{\times\mathbf{V}}\mathbf{O}
\xrightarrow{+\mathbf{X},\mathrm{LN}}\mathbf{Z}
$$

### 선형 투영 + 어텐션 맵

먼저 Self-Attention단계에서는 아래 수식과 같이 의미벡터 $X_i$에 대해 미리 학습된 3가지 투영행렬 $\mathbf{W}_Q,\mathbf{W}_K\in\mathbb{R}^{d\times d_k}$, $\mathbf{W}_V\in\mathbb{R}^{d\times d_v}$을 각각 곱하여 각각의 의미공간으로 투영한다.

$$
\mathbf{Q}=\mathbf{X}\mathbf{W}_Q,\;\; \mathbf{K}=\mathbf{X}\mathbf{W}_K,\;\; \mathbf{V}=\mathbf{X}\mathbf{W}_V
$$

$$
Q_i = X_i\mathbf{W}_Q\in\mathbb{R}^{d_k}
$$

각각의 투영행렬은 $Q=찾는 기준, K=매칭용 특징, V=실제 전달 내용$ 이라고 보통 설명된다. 여기서 나는 좀 이해가 안되었었는데, 추후에 $Q,K,V$가 각각 사용되는 수식을 보고 이해됐다. 그래서 이후 식과 함께 $Q,K,V$에 대해 자세히 설명하려고 한다.

$$
\displaystyle \mathbf{S}=\frac{\mathbf{Q}\mathbf{K}^{\top}}{\sqrt{d_k}}\in\mathbb{R}^{N\times N},
\qquad
\displaystyle S_{ij}=\frac{Q_i\cdot K_j}{\sqrt{d_k}}

\\

\mathbf{A}=\mathrm{softmax}_{\text{row}}(\mathbf{S}),
\qquad
A_{ij}=\frac{\exp(S_{ij})}{\sum_{k=1}^{N}\exp(S_{ik})},\quad \sum_j A_{ij}=1
$$

위 식은 **토큰 i가 토큰 j에 의해 얼마나 바뀌어야 하는지에 대한 값($A_{ij}$=Attention Score)** 을 계산하는 수식이다. **i번째 토큰의 $Q$벡터** 와 **j번째 토큰의 $K$벡터** 의 코사인 유사도를 나타낸다. 비슷하면 높은값. 다르면 낮은값을 나타내게 된다.

여기서 $Q$, $K$의 각 의미를 짐작해 볼 수 있는데, 먼저 문법 요소들 사이의 관계를 간단하게 생각해보자.

가장먼저 "사과는 맛있다"라는 문장에 대해 생각해보자. 해당 문장은 명사 "사과", 조사 "는", 동사 "맛있다"로 이루어져있다. 이때 "사과"라는 명사는 조사 "는"에 의해 주어로써 의미를 갖게되고, "맛있다"라는 동사를 통해 어떠한 사과인지에 대해 물었을때 "맛있다는" 서술적인 의미를 갖게된다. 하지만 이 관계를 반대로 생각해보면 다르다, 조사 "는"은 어떤 명사 뒤에 붙더라도 해당 단어를 주어로 만들어 줄 뿐, 자신이 의미를 갖지는 않는다. 반면 "맛있다"는 동사는 어떤 주어를 만나냐에따라 의미가 미묘하게 달라질 수 있다. 음식을 나타내는 주어 뒤에서는 말그대로 "맛있다"라는 뜻을, 수익률, 게임 스킬샷 같은 다른 주어 뒤에서는 각각 "수익이 좋다", "스킬을 잘 맞췄다" 라는 의미를 가질수도 있다.

즉 "사과"는 "는"애는 영향을 주지 못하지만, "는"은 "사과"에 많은 영향을 미치는것처럼 **문법 요소간에 영향을 주고 받는 관계가 대칭적이지 않다**라는 것이다. 이를 Transformer논문에서는 Q,K 행렬을 분리하여 경험적으로 해결한것이라고 보였다.

이처럼 **$Q$는 해당 토큰이 어떤종류의 토큰에 미쳐야 하는지**, **$K$는 특정 토큰이 어떤 종류의 토큰에 영향을 미칠수 있는지** 정보를 담고있는 벡터라고 할수 있다. 이렇게 계산된 Attention Score는 바로 아래 식에서 사용된다.

<br>

### 가중합

서서 어텐션 맵 $A$를 생성하였다. 이제 i번째 토큰이 j번째 토큰을 얼마나 참고해야하는지가 정해졌으니, 이를 기반으로 다시 토큰을 재구성 해야 한다. 여기서 $V$값이 쓰이게 된다. 일단 아래 식을 봐보자.

$$
\mathbf{O}=\mathbf{A}\mathbf{V}\in\mathbb{R}^{N\times d_v},
\qquad
\displaystyle O_i=\sum_{j=1}^{N} A_{ij}\,V_j
$$

위 식에서 $X$로부터 투영된 $V$가 무슨 의미를 갖는지 알 수 있다! "는"이라는 문법요소가 주어에 영향을 미친 "문법상의 의미", "맛있다"라는 문법요소가 주어에 전달한 "서술상의 의미"등을 담고있는 의미 벡터인 것이다.

위 식에서 $O_i$는 가중치 $A_{ij}$만큼 모든 토큰의 Value를 섞은 새 표현. 즉 **$O_i$는 입력벡터 $X_i$가 주변 토큰들에 의해 변형된 값**을 나타내게 된다.

드디어 Transformer에서 어떻게 문맥 사이에서 각 단어가 무슨의미를 갖는지 계산하는법에 대해 알게 된것이다! 이러한 방법을 통해 "사과는 맛있다"에서 "사과"라는 토큰이 맛있고 주어라는 의미를 갖도록 변형할 수 있게 된것이다.

> 실제로 토큰이 나눠지는 방법은 형태소를 분리하는 방법을 그대로 사용하진 않지만, 학습과정에서 거의 비슷하게 되긴할것이다. 또한 논문에서 Q, K, V를 생성하는 과정 또한 이렇게 문법적으로 정의하여 생성하지 않고, 학습과정에서 학습으로 emergent하게 학습된다. 하지만 후대논문들에서 이와 비슷한 내용으로 해석한 부분을 추가적인 문법지식에 결합하여 이렇게 설명한 것이기 때문에 참고하자. 근데 목적함수가 저렇게 되어있으면 어차피 저렇게 학습되긴 할듯?

<br>

### 잔차 연결 + 정규화

어텐션 출력 $\mathbf{O}$를 그대로 다음 단계로 보내지 않고, **원래 입력 $\mathbf{X}$를 다시 더한 뒤(잔차 연결) 정규화(LayerNorm)** 한다.

$$
\mathbf{Z}=\mathrm{LayerNorm}(\mathbf{X}+\mathbf{O})
$$

이렇게 $\mathbf{X}$에 변형된 어텐션 레이어 출력 $O$를 더하는 방식에는 아래와 같은 이점이 있다고 한다.

- 어텐션은 원본을 통째로 갈아치우는 게 아니라 **"얼마나 바꿀지(변형분)"만 만든다**고 보면 된다. $\mathbf{X}+\mathbf{O}$ = "원본 + 변형분" 이라 정보가 점진적으로 갱신된다.
- 역전파 때 **그래디언트가 층을 건너뛰는 지름길**이 생겨, 깊은 망에서도 그래디언트가 사라지지(vanishing) 않아 레이어를 수십 개 쌓아도 학습아 용이하다 → 기존 입력벡터의 의미 유지가 쉽다.

또한 위 식에서 $\mathrm{LayerNorm}$ 함수는 아래와 같은 식을 따른다.

$$
\mathrm{LayerNorm}(z) = \gamma \odot \frac{z - \mu}{\sqrt{\sigma^2 + \epsilon}} + \beta,
\qquad
\mu = \frac{1}{d}\sum_{m=1}^{d} z_m,\;\;
\sigma^2 = \frac{1}{d}\sum_{m=1}^{d}(z_m - \mu)^2
$$

각 토큰 벡터를 **평균 0, 분산 1로 맞춘 뒤** 학습된 스케일 $\gamma$·이동 $\beta$를 적용한다. 이를 통해 값이 너무 커지거나 작아지는 걸 막아 **학습을 안정화**한다. $\gamma, \beta$ 는 학습되는 벡터라, 정규화로 잃을 수 있는 표현력을 모델이 되찾을 수 있다.

> 잔차 + LayerNorm 한 세트(흔히 "Add & Norm")가 트랜스포머를 **깊게 쌓을 수 있게** 하는 핵심 장치다. FFN 뒤에도 똑같이 한 번 더 적용된다.

<br>

### Multi-Head — 여러 관점으로 동시에 보기

지금까지 설명에서는 어텐션을 한 번만 했지만, 실제로는 **서로 다른 $\mathbf{W}_Q,\mathbf{W}_K,\mathbf{W}_V$ 를 가진 어텐션을 $H$개 병렬로** 돌린 뒤 합친다.

$$
\mathrm{head}_h=\mathrm{softmax}\!\left(\frac{\mathbf{Q}^{(h)}{\mathbf{K}^{(h)}}^{\top}}{\sqrt{d_k}}\right)\mathbf{V}^{(h)},
\qquad
\mathbf{O}=\mathrm{Concat}(\mathrm{head}_1,\ldots,\mathrm{head}_H)\,\mathbf{W}_O
$$

위 식과 같이 여러가지 투영행렬$W$을 사용하는 이유는 아래와 같다.

- 토큰 사이 관계는 한 종류가 아니다. 어떤 head는 **문법 관계**(주어-동사), 어떤 head는 **의미 관계**, 어떤 head는 **가까운 위치** 처럼 서로 다른 관계를 잡는다.
- 단일 head면 이 모든 게 하나로 평균돼 섞여버린다 (원논문: *"with a single attention head, averaging inhibits this"*).
- 각 head는 $d_k = d/H$ 차원으로 작업하고($H$개로 쪼갬), 결과를 **이어붙인(Concat) 뒤 $\mathbf{W}_O$ 로 다시 $d$차원**으로 합친다. → 전체 차원·계산량은 단일 풀(full) 어텐션과 비슷하게 유지하면서 다양한 관계를 동시에 포착한다.

<br>

---
# 정리

결과적으로 Self-Attention은 한 문장의 입력 $\mathbf{X}$에서 Q·K·V를 만들고 **각 토큰이 문장 전체를 참고해 새 표현을 만드는** 함수다. 때문에 동음이의어는 "섞는 문맥이 달라서" 의미벡터가 분화되는 방식으로 처리가 되는것이다. 다음 포스트에서는 이제 이렇게 맥락의 정보를 모두 포함하게 된 행렬 $Z$를 기반으로 어떻게 다음 단어를 추론하는지 확인해보자.