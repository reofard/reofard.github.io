---
title:  "Paper Review"
excerpt: "Deep Learning Summary"

categories:
  - A.I

tags:
  - [deep learning, machine learning, AI, School Summary]

toc: true
toc_sticky: true
layout: post
use_math: true
 
date: 2022-12-06
last_modified_at: 2022-12-06
---

# Image Colorizatioon

### 실험 결과

- DataSets 종류
  - coco-stuff dataset
  - pascal voc dataset
  cifar datasets
- 정량적 평가 방법
  - psnr
    - 복원된 영상과 원본영상 사이의 차를 최소제곱 에러를 구한다
  - ssim(structure similar identify ???)
    - 이미지의 밝기, 색상등을 비교 
    - 값이 클수록 잘 복원된 것
  - pcqi
    - 최대신호 잡음비, psnr으로 평가 
  - iqm
    - 구조적 유사성 구조
    - 색깔이 얼마나 선명하냐
    - 색깔이 얼마나 풍부하냐
    - 구조

### 객체 단위로 알때 컬러를 입히는법

- 객체단위로 입히는게 더 좋음
- 중간중간 객체간의 통일성을 위해 퓨전 진행
- 객체인식을 위해 이미 공개되고 훈련된 객체 인식 신경망 사용

# TTS<sup>text to speech</sup>

- 음성신호는 진폭데이터로 변환되여 1차원 벡터가 생성
- 보통 44100번의 진폭으로 데이터를 생성

논문 설명에서 모델 크기 조절 설명할수 있어야됨

하나하나 물어보기보단 딥러닝에 대한 개념이 나올 것
컬러라이제이션 메트릭(평가 기준) 이해해야됨
instance aware image colorization 자세하게 이해해야됨
  순서, 네트워크 프레임워크도 잘 이해 해야함

tts 2d를 3d로 변환하는 과정 외워야함

causal cnn외우기

아트록스 컨볼루션 개념
aspp 외우기

depthwise convolution 하는방법 알아두기