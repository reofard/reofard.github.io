---
title:  "MegaDepth: Learning Single-View Depth Prediction from Internet Photos 리뷰"
excerpt: "Paper Summary"

categories:
  - A.I

tags:
  - [Deep Learning, Computer Vision, Mono Depth Estimation, Paper Review]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2024-08-24
last_modified_at: 2024-08-24
---

# **MegaDepth: Learning Single-View Depth Prediction from Internet Photos**

해당 논문은 앞서 리뷰했던 [MiDaS](https://reofard.github.io/a.i/2024/08/21/Towards-Robust-Monocular-Depth-Estimation-Mixing-Datasets-for-Zero-shot-Cross-dataset-Transfer-리뷰.html)에서 언급했던 논문이다. 앞선 논문에서 핵심 아이디어로 **정규화 손실항**을 이용하여 이미지에서 경계면의 특징을 잘 포착하는 결과를 내었는데, 이게 해당 논문에서 제안한 아이디어이다.

해당 논문에서도 역시 양질의 데이터를 확보하는게 어렵운걸 큰 문제사항으로 정의하고, 이러한 문제를 해결하기 위해 여러가지 아이디어를 제안한다. 그리고 앞서 말한 **정규화 손실항** 역시 인터넷 이미지를 막 갖다 사용하면서 발생하는 문제를 해결하기 위해 제시된 아이디어였다.

해당 논문은 위에서 언급한 방법들을 통해 인터넷 이미지를 처리하여 양질의 데이터셋으로 만들수 있었다는데, 어떤식으로 데이터를 구성하였는지 한번 정리하면서 알아보자.

<br>

## **간단한 설명**

해당 논문에서는 양질의 데이터셋을 구성하고, 단안 이미지에서 depth estimation 모델을 학습하기 위해 다음과 같은 절차를 진행한다.

1. 인터넷에서 특정 장소에서 촬영된 이미지를 수집
2. 최신의 이미지 기반 3D Reconstruction 기법을 활용하여 SfM 모델 생성
3. SfM 모델을 기반으로 3D Reconstruction에 사용된 이미지마다 Depth Map 생성
4. 생성된 Depth Map의 노이즈와 outliner를 해결하기 위한 전처리

위와 같은 방식으로 전 세계 200개의 랜드마크에서 3D 모델을 재구성하고, 약 15만개의 이미지, Depth Map 쌍을 생성한다고 한다. 그중 13만개의 유효한 데이터셋이 생성되는데, 100만개는 Continuous Depth을 가지는 Euclidean Depth map, 30만개는 의미론적 세그멘테이션을 통한 descreted Depth값을 갖는 Depth Map이 생성된다.

그 이후에는 Deep Network에 위에서 생성한 데이터를 학습하여 Single Image Depth Estimation Model을 생성하게 된다.

<br>

## **1. Creating a dataset based Internet Photo**

가장 먼저 데이터셋을 생성하기 위해 [사진 공유 사이트](https://www.flickr.com)에서 사진을 모으게 되는데, 3D Reconstruction을 위해서는 같은 장소에서 촬영된 이미지 셋이 필요하다. 이때 해당 논문에서는 크롤링 된 이미지들을 [Landmark Query](https://link.springer.com/chapter/10.1007/978-3-642-33718-5_2)를 이용해 각각의 장소에서 찰영된 이미지 셋을 만든다.

그 이후 각각의 장소마다 분류된 이미지를 이용해서 [COLMAP](https://colmap.github.io)<sup>[SFM](https://openaccess.thecvf.com/content_cvpr_2016/papers/Schonberger_Structure-From-Motion_Revisited_CVPR_2016_paper.pdf)과 [MVS](https://demuc.de/papers/schoenberger2016mvs.pdf)을 포함하는 파이프 라인</sup>을 이용해서 3D Model을 재구성 하게 된다. 이때 추가적으로 각 이미지별 Depth Map, 카메라 내,외부 파라미터등의 정보들 역시 같이 계산된다고 한다.

![복원된 3D 모델](/assets/img/colmap_reconstruction.png)

![복원된 Depth Map](/assets/img/colmap_origin.png)
![복원된 Depth Map](/assets/img/colmap_depth.png)

<br>

## **2. Data Preprocessing**

COLMAP에서 생성된 Depth Map정보는 아래와 같은 문제를 가진다고 한다. 이러한 문제들은 Stereo Matching의 한계점 때문에 발생하게 되는데 이러한 에러에 대해 자세히 설명하면 다음과 같다.

1. 유동적인 물체에 대한 부정확한 깊이 정보
2. 노이즈에 의한 불연속적인 깊이정보
3. 전경과의 깊이값이 후경의 깊이값에 의해 침식당하여 발생한 에러

해당 논문에서는 앞서 인터넷 이미지를 기반으로한 데이터를 몇가지 전처리 과정을 통해 가공하여 이러한 에러를 해결하였다고 한다. 하나씩 알아보자.

<br>

### **Photo calibration and reconstruction**

첫번째로 제안하는 방법은 Depth Map에 smoothing을 적용하는것이다. 영상기반 3D Reconstruction을 진행하면 특징점이 완벽한 3D 모델이 생성되지 않고, 전경이 후경에 의해 침식되는 경우가 많다. 이를 해결하기 위해 해당 논문에서는 3D 모델을 기반으로 Depth Map 정보를 계산할 때 근처 픽셀 값을 참고하여 깊이값이 더 낮은(낮을수록 카메라에 가까움)값을 선택하고, 중위값 필터를 통해 outliner를 제거하는 방식으로 Depth Map을 구하게 된다. 이 경우 아래 사진과 같이 전경(조각상)의 깊이값이 후경(벽)에 의해 침식되는 에러를 막을 수 있다고 한다.

![전경 후경 비교사진](/assets/img/depth_bleeding.PNG)

<br>

### **Depth map refinement**

두번째로 제안하는 방법은 Semantic Segmentation을 이용해 Depth Map 정보를 보강하는것이다. 해당 논문은 인터넷에 올라온 무작위 데이터를 기반으로 하기 때문에 SfM, MVS를 수행하는데 큰 문제점이 존재한다. 바로 동일한 장소를 서로 다른 시간에 찍힌 이미지를 기반으로 하기때문에 특정 사진에만 존재하는 객체(사람, 자동차)나 동일한 객체의 시간에 따른 변화(신호등, 하늘, 나무)가 존재하여 이미지들이 100% 매칭이 될 수 없다는 것이다.

이러한 에러를 해결하기 위해 딥러닝에 의한 Semantic Segmentation를 사용했다고 한다. 이 segmentatioin 정보를 기반으로 Depth Map에 전처리 과정을 거치게 되는데, 우선 Semantic Segmentation을 통해 일시적인 오브젝트인지 여부를 판단하게 된다. 구해진 영역안에서 3D 모델로 부터 계산된 Depth값이 비연속적이면 일시적인 물체라고 판단하고, 아래 그림과 같이 해당 segmentation 영역에서의 깊이값을 depth map에서 제거하게 된다.

![일시적인 사람 사진](/assets/img/temp_object.PNG)

<br>

그 다음은 앞서서 판단한 정보에 따라 2가지로 학습데이터를 분류하는것이다. 앞서 설명한 일시적인 물체가 이미지상에 너무 많을 경우 자연스럽게 Depth Map의 신뢰도 역시 하락하게 되는데, 이러한 정보들은 Geometry정보를 가진 Euclidian Depth Map 데이터로 사용하지 않고, 전경, 후경의 순서로 이루어진 Ordinal Depth Map으로 사용하여 학습에 이용하게 된다. 이때 아래 사진과 같이 하나의 이미지에서 각 세그멘테이션 영역별 Depth 값이 비 연속적일수록 전경일 가능성이 높다고 한다.

아래 그림의 경우 파란부분이 전경(Depth값이 비연속적), 빨간부분이 후경(Depth값이 연속적)으로 구분되어 Ordinal Depth Map을 이루게 된다.

![전경 후경 판별](/assets/img/전경후경.PNG)

이러한 기법은 일시적인 물체에대한 부정확한 깊이값 픽셀을 없애버리기 때문에 학습할 때 일시적인 오브젝트에 대한 깊이값 추론 성능이 낮아질 수 있는데, Ordinal Depth Map 데이터를 통해 일시적인 오브젝트의 Depth 추론에 필요한 학습데이터의 부재를 완화시켜 줄 수 있다고 한다.

<br>

## **2. Depth Estimation Network**

앞선 데이터 처리 과정을 통해 Euclidian Depth Map, Ordinal Depth Map으로 이루어진 데이터 셋인 MegaDepth Dataset이 만들어지게 된다. 해당 논문에서는 여기서 멈추지 않고 여러 딥러닝 모델을 학습시켜 비교하고, 생성한 데이터의 특징에 맞는 손실함수를 설계하여 제안한다. 앞서 리뷰한 MiDaS 논문에서 소개한 손실함수도 해당 내용을 제안하는데, 본 리뷰에서는 그 부분을 자세하게 정리해보려고 한다. 해당 논문에서는 아래 식과 같이 3가지 손실항의 합으로 이루어지는 [**Scale-invarient Loss**(=$\mathcal{L}_{si}$)](https://arxiv.org/abs/1406.2283)를 사용한다.


> $$\mathcal{L}\_{si} = \mathcal{L}\_{data} + \alpha\mathcal{L}\_{grad} + \beta\mathcal{L}\_{ord}$$

### **Scale-invariant data term**

Stero Matching 방식의 3D Reconstruction 방식으로 구해진 Depth Map은 실제 Geometry 정보를 포함하지 못한다. 그렇기 때문에 앞서 말한 Landmark마다 생성된 3D 모델사이에 스케일관계를 알 수 없다는 문제가 있다. 이번 논문에서는 MiDas에서도 Related Work로 소개되었던 손실함수인 Log Depth 공간에서의 손실함수($\mathcal{L}_{data}$)를 첫번째 항으로 사용한다.

수식 깨지는 에러 수정후 내용 추가

### **Multi-scale scale-invariant gradient matching term**

수식 깨지는 에러 수정후 내용 추가

### **Robust ordinal depth loss**

수식 깨지는 에러 수정후 내용 추가

<br>

## **후기**

MegaDepth는 여러 싱글 이미지를 통해 3D Reconstruction을 진행 후 이를 바탕으로 Depth Map을 추출하고, 여러 이미지 의미론적 요소를 이용해 신뢰할 수 있는 데이터를 생성하고 학습하는 아이디어이다. 처음 읽을때는 SfM을 본인들이 처음 사용한거라고 강조하길래, 이 기술이 나온지가 거의 10년 단위가 될텐데 뭐가 특별하다고 자랑하는거지? 라고 생각했는데, 데이터 생성, 손실 계산 부분에서 앞서 말한 여러가지 영상의 의미론적 요소(의미론적 분할, 경계면 검출 등)의 요소를 적극적으로 활용하여 더 정확한 결과를 만들어 내는걸 보고 그때야 이 논문이 대단하다고 느꺘다.

MiDaS, MegaDepth 둘다 기존에 존재하는 이미지를 활용하여 양질의 학습 데이터를 만들어내어 모델을 학습시키는데, 둘다 새로운 학습 기법, 모델을 사용한게 아니라 기존에 있던것에 영상처리의 기본적인 지식들을 활용하여 변형하는 아이디어를 제시한 논문이었다. 나도 이렇게 기본기에 충실한 전문가가 되어 이런 아이디어를 생각할 수 있는 사람이 되고 싶다는 생각이 들었다.