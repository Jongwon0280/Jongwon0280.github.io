# AWS TECHNICAL ESSENTIALS
---
title:  "AWS TECHNICAL ESSENTIALS"
excerpt: "AWS 기초내용 정리입니다. "

categories:
  - Cloud
tags:
  - [Blog, AWS, Cloud]

toc: true
toc_sticky: true
 
date: 2023-01-20
last_modified_at: 2023-01-20
---



# **클라우드 컴퓨팅**

- 온디맨드로 서비스에 엑세스
- 필요에 따라 컴퓨팅 리소스 프로비저닝
- 사용한 만큼만 요금 지불

### 장점

- 종량제
- 규모의 경제로 얻게 되는 이점
- 용량 추정 불필요
- 속도 및 민첩성 향상
- 비용 절감 실현
- 전세계배포

### 리전

- AWS가 서비스하는 지역 ( 데이터센터의 위치)
    - 최소 3개영역의 가용영역 → 한국은 서울(4개)
1. 대기시간
2. 요금
3. 서비스 가용성 → 200개이상의 서비스 (ex : 서울 170개이상)
4. 데이터 규정 준수 → 정보보호 관리체계

### 가용영역

**고가용성**

: 데이터 센터가 클러스터 형태

**(클러스터 : 여러대 컴퓨터를 한개의 형태를 띔)**

장애시 다른 가용영역이 서빙할 수 있도록 함.

### 엣지 로케이션

: Cacheing이 가능하도록 함.

네트워크적 측면에서 캐싱이란? 

Origin Server ↔ End User

미리 리소스를 저장해놓음.

**Contents Delivery Service(CDS)**

**리전 - 가용영역 - 데이터센터 - 서버**

### IAM

Access Management는 AWS계정 및 리소스에 대한 보안 액세스를 관리하는데 도움이 되는 웹 서비스.

계정도 여러 팀원들이 접근 가능하게 함. (접근권한 제어를 해주는 서비스를 IAM이라고 한다.)

→ ROOT User는 사용하지 말고 IAM서비스에서 admin권한을 가지는 IAM User를 생성하여 사용해라.

**ROOT → IAM**

**컴포넌트** 

- **IAM USER**
    
    IAM USER의 ADMIN권한을 통해 팀원들의 권한을 부여한다.(모니터링, 시각화 등등)
    
- **IAM POLICY**
    
    어떤 보안 정책을 줄 것인지에대한 JSON파일형식
    
- **IAM GROUP**
    
    그룹에 IAM정책을 할당하면 해당 그룹은 다 IAM 사용자
    
- **IAM ROLE**
    
    임시권한을 부여한다. 잠깐 다른 유저의 권한을 필요로 할때 해당 ROLE을 위임가능하다.
    
- **멀티 팩터 인증**

![Untitled](/assets/aws/AWS_1.png)

- **S3-SUPPORT - S3 READ ONLY 계정 (user-1)**
- **EC2-SUPPORT - EC2 READ ONLY 계정 (user-2)**
- **EC2-ADMIN - EC2 보기, 시작, 중지 권한을 가진 계정 (user-3)**

### EC2

- Amazon Machine Image(AMI)

→ Ubuntu Linux, OSX, 

- Instance (Virtual Server)
- Server vs Instance(VM) ⇒ 서버는 상시 돌아가는 것. / 인스턴스는 그때 그때 할당받고 사용할 수 있는 것.

**EC2 유형**

ex) **t2.micro** 

**→ t : 범용 c : 컴퓨팅최적화 ( instance family) / 2 : Processor’s generations  / micro : storage(memory, cpu 수)**

- 범용인스턴스 : 균형있는 지원
- 컴퓨팅 최적화 : 컴퓨팅파워를 높인것.
- 메모리 최적화 : 메모리사양을 높인것.
- 가속화 컴퓨팅 : 머신러닝에 최적화된 서버를 지원.
- 스토리지최적화 : 높은 디스크 처리량

![Untitled](/assets/aws/AWS_2.png)

### Container

![Untitled](/assets/aws/AWS_3.png)

가상머신은 가상화플랫폼이라는것이 존재한다. 컨테이너는 가상화플랫폼이 존재하지 않는다.

컨테이너 식으로 어플레이케이션을 확장한다. ( 쿠버네티스, 도커 환경)

- **ECS**
    
    도커 기반 실행
    
- **EKS**
    
    쿠버네티스 기반 실행
    

### ServerLess

- 서버가 없는 것이 아닌 아마존 자체에서 서버를 관리해주는 것.

→ **AWS Fargate / AWS Lambda(이벤트 기반)**

ex) 주말 트래픽이 갑자기 증가한다. → capacity확보가 필요함. → 서버리스는 알아서 해준다.(접근불가다.)

![Untitled](/assets/aws/AWS_4.png)

ex) User Managment System → Trigger 환경이 필요할때 사용한다.

Lambda는 자동화되어 있어서 알아서 꺼진다. 

### VPC(Virtual Private Cloud)

: AWS에서 Private하게 사용할 수 있는 네트워크

![Untitled](/assets/aws/AWS_5.png)

Subnet : 리소스를 효율적으로 관리하기 위함. 

1. 퍼블릭 : 인터넷에서 접근가능하게함.
2. 프라이빗 : 사설망

**고가용성**

### CIDR(클래스가 없는 도메인 간 라우팅)

- private_subent에서는 **nat게이트웨이**에 요청을 한다. (단방향이다.)

### 라우팅테이블

**public_routing_Table**

10.0.0.0/16 로컬

0.0.0.0/0 모든ip는 인터넷으로

**private_routing_Table**

10.0.0.0/16 로컬

0.0.0.0/0 NAT gateway에 연결

### ACL

- 블랙리스트 작성.
- Public Subnet에만 해당함. ( 네트워크ACL → 인터넷게이트웨이 → 인터넷)

![Untitled](/assets/aws/AWS_6.png)

- INBOUND
    
    
- OUTBOUND

→ 둘다 작성해야한다.

### 보안그룹

- INBOUND만 작성하면 자동으로 outbound에 적용할 수 있다.

### 스토리지

- 블록 스토리지 : 블록(바이트단위로 접근이가능한 단위)이라는 단위로 저장
    
    읽기/쓰기에 대한 오버헤드가 작다. (Root Volume)
    
    EBS(Elastic Block Storage) → **EC2 (단독적으로 사용할 수 없다.)**
    
    1. **인스턴스 스토어** (ssd) 2. **EBS(ssd는 맞지만 네트워크 latency 존재)**
        
        인스턴스 스토어는 휘발성이다. → 주기적으로 EBS나 S3에 백업을 해주는 것이 필요하다.
        
        Amazon EBS → **1. SSD 2. HDD**
        
- 파일 스토리지
    
    **공유파일시스템** 개념이며, 그것을 서비스한 것이 **EFS(Elastic File System)**
    
    +ebs는 최대 16개까지 동시접근 가능하다.
    

![Untitled](/assets/aws/AWS_7.png)

- 객체 스토리지(ex : S3)
    
    변경이 필요할 시 새로운 객체를 생성해야한다. ( 오버헤드가 크다.) → 읽기만 필요할때 사용.
    
    고유한 URL을 통해 접근가능하다. (이미지, 비디오파일 등) 
    
    s3보안 : IAM정책 / 버킷정책 / S3 ACL / S3 암호화
    
    버킷정책 : 나한테 올 수 있는애는 누가 있고~~
    

### 모니터링

EC2기반으로 만들었을 때 자동으로 스케일링 해주는 P/G은 어떻게 만드는지.

- Cloud Watch
    1. 운영문제에 선제적 대응
    2. 성능 및 안정성 개선
    3. 보안 위협 및 이벤트 인지
    4. 데이터에 기반한 의사결정
    5. 비용 효율적 솔루션 생성

### 로드밸런싱

:확장성있는 아키텍처를 사용.

컴퓨팅자원들에 균형적으로 배분해주는 역할이다.

- **ELB(Elastic Load Balancing)**
    1. 고가용성
    2. 보안
    3. 기능적용범위
    4. 강력한 모니터링
    5. 통합 및 글로벌 접근성
- 리스너
- 대상그룹
- 규칙
- ELB 유형
    
    Application(osi 7layer) / Network(osi 4layer) / Gateway / Classic
    

### EC2 Auto Scaling (크기조정)

:로드 밸런서의 배분후에 컴퓨팅 리소스를 실질적으로 늘리고 줄이는 역할을 한다.

- **수직조정**
    
    중지 / 크기 or 유형변경 / 인스턴스 시작
    
- **수평조정**
    
    인스턴스 개수 추가 및 줄이기.
    

시작템플릿 : AMI / 패밀리 정보 등등.