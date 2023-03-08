---
title: 이상 URL탐지웹구현(1. 아키텍처 및 데이터 소스)
layout: single
categories: 
   - Project
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-03-08
---

# 이상 URL탐지웹구현(1. 아키텍처 및 데이터 소스)

### 개요

최근에 실시간 로그를 수집하고, Elastic Search 및 Kibana로 시각화하는 솔루션을 구현하면서, 실제로 시스템에 적용시켜보고싶다는 생각을 가지게되었고, Kaggle의 다양한 경연대회와 데이터들을 보던중 불법url데이터셋을 찾게 되었다.
![kaggle_MALURL](https://user-images.githubusercontent.com/56438131/223626759-96090b90-1800-4ec7-9141-cb410ef03c9a.PNG)
[Malicious URLs dataset](https://www.kaggle.com/datasets/sid321axn/malicious-urls-dataset)


이데이터셋을 이용하여 불법url탐지에 활용할 것이며, Flask 파이썬 웹서버 프레임워크를 활용하여 사용자가 url을 입력하면 url을 파싱하여 피처를 추출하고 사전학습된 모델을 이용하여 예측치를 얻어 사용자에게 뿌려주고 로그를 생성하여 ELK스택을 이용하여 Opensearch에 적재 및 OpenDashBoard에서 시각화할 것이다.

URL의 판독결과를 ELK로 실시간 로그처리를 하는 이유는 불법 url배포를 목적으로 확인을 하는 악의적인 유저를 식별하기 위함이다.

 

### 아키텍처 및 디렉토리 구조

![Untitled](https://user-images.githubusercontent.com/56438131/223625883-e49bc176-0e52-4987-a9c8-c8f945f1bd9e.png)

**서버1**

웹에서 사용자에게 입력을 받아야하므로, 파이썬 웹서버 프레임워크 Django와 Flask 중에서 비교적 간단히 구현이 가능한 Flask를 선택했다.

사용자의 입력(url)을 불러오고 그값을 파이썬을 통해 파싱한 후 특징을 추출한 후 예측을 하며 예측값은 사용자 화면에 출력되고, 로그생성을 수행하여 logstash에서 opensearch로 forwarding한다.   

**서버2**

logstash에서 forwarding한 값을 opensearch로 받아온 후 open-dashboard를 연동하여 실시간으로 로그를 수집할 수 있는 서버이다.

**디렉토리구조**

**서버1**

![Server_1](https://user-images.githubusercontent.com/56438131/223626827-2e3a4231-b9e3-4b30-9e24-1be6a7891bf1.PNG)

**서버2**
![Server_2](https://user-images.githubusercontent.com/56438131/223626836-b8e1cd00-2c73-47de-bf29-701ef3dcdece.PNG)
### 데이터 요약

651,191개의 URL로 구성된 거대한 데이터 세트를 수집했으며 그 중 428103개의 양성 또는 안전한 URL, 96457개의 손상 URL, 94111개의 피싱 URL 및 32520개의 맬웨어 URL이 있다. 

![table](https://user-images.githubusercontent.com/56438131/223626946-91ce523c-319b-472e-9d45-a90fb5e48586.PNG)


데이터 자체가 65만개정도로 대부분의 url 42만개가 정상url이므로, 일반화 성능이 안나올 가능성이있다. 추후작업으로 비정상url 샘플링 작업이 필요할것으로 보인다.

이번 포스팅에서는 전체적인 프로젝트의 개요와 아키텍처 구조 및 데이터를 간략하게 살펴보았다. 다음 글에서는 이 프로젝트의 핵심이되는 파트인 50만개해당하는 url을 파싱하고 학습시키는 것을 포스팅 할 것이다.