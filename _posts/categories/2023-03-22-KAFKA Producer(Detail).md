---

layout: single
title: KAFKA Producer(Detail)
categories:
  - DataEngineering
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-03-22
---


### 1. 아키텍처

![Untitled](https://user-images.githubusercontent.com/56438131/226888405-563f9f57-1706-4b90-b856-c7479432dec3.png)

1. send() : Producer가 send()를 호출하면, buffer에 레코드를 추가하고, 즉시 future객체로 반환한다. Future객체는 비동기 응답에 대한 처리방법을 제공하는 Java의 인터페이스이다.
2. Serializer : ProducerRecord타입을 bytes형태로 바꿀때 미리지정된 key.serializer.value.serializer를 이용해서 serialize한다.
- Partitioner : 각 ProducerRecord의 파티션 할당을 진행한다.
    
    Round-Robin Partitioning : ProducerRecord를 생성할때 Partition 파라미터가 null일때 라운드로빈 방식이 지정된다.
    
    Hash Key Based Partitioning : 값이 널이 아닌경우 그 값을 Hash한 결과로 파티션이 지정된다.
    
    Hash 방식은 skew가 될 가능성이 있기때문에 적절한 key 설계가 필요하다.
    
1. Buffer: 아직 kafka broker 로 보내지지 않은 serailze 된 레코드들이 topic-partition 별로 나뉘어 저장되어 있다. Topic Partition별로 존재하는 Dequeue 에는 batch.size 크기로 데이터가 묶어서 저장된다. 실제 데이터 전송시에는 여러개의 batch 덩어리가 전송된다. 총 사이즈는 buffer.memory 로 설정한다.
2. I/O Thread : send 메소드를 호출한 어플리케이션 스레드와 별도로 Background thread에서 buffer에 쌓인 메세지들로 데이터 전송을 처리한다.

<aside>
💡 1,2,3 → User Thread 4→ I/O Thread

</aside>

### 2. RecordAccumulator

Record를 버퍼에  쌓는 과정, 그리고 쌓인 메시지를 I/O 스레드가 보내는 과정은 RecordAccumulator를 중심으로 이루어진다. 

![Untitled 1](https://user-images.githubusercontent.com/56438131/226888415-8eb7099d-80e5-4113-8083-12ab72dfc4e4.png)

Buffer Pool에 데이터가 들어오면 append로 각 TopicPartion별로 존재하는 덱에 추가한다. 메시지들은 추가되면서 Batch Size에 맞게 묶인다. I/O Thread에서는 데이터를 drain이라는 과정으로 batch단위로 앞에서부터 꺼낸다.

### 2.1 append

![Untitled 2](https://user-images.githubusercontent.com/56438131/226888430-a3fc7fac-d80b-454d-856b-c81caab659d4.png)

TopicPartition별로 존재하는 큐에서 들어온 순서대로 전송한다. queue에 추가될때는 batch Size별로 추가하는데, 레코드가 batch에 쌓일때, compression.type에 지정된 압축방법이 적용되어 들어가게된다. 만약 남은 batchSize보다 새로들어온 압축된 레코드 사이즈가 크다면 새로운 batch를 생성하게 된다.  그래서 batchSize를 잘 설정하여야 단편화현상을 최소화 할 수 있다.

### 2.1 drain

request 리스트는 broker node 를 대상하는 node 와 노드별로 전송할 batch records 의 리스트의 쌍으로 구성되어있다. 왜냐하면 각 broker 당 하나의 connection으로 데이터를 전송하기 때문이다
drain 과정은 하나의 broker node 와 맺는 하나의 connection 에서 얼마나 많은 요청을 모아서 보낼 수 있는지를 정리하는 작업이라고 보면 된다.

**또한 아무리 한 토픽에 파티션이 많다고 하여도, Producer와 Broker간의 컨넥션은 무조건 하나이다.**

![Untitled 3](https://user-images.githubusercontent.com/56438131/226888446-6a1da20f-c94b-4e1a-acee-b9a2d1ab6806.png)

1. 각 node(브로커)에 대한 정보를 처음 가져온다.
2. 해당 브로커의 토픽파티션리스트를 파악하고, 각 파티션별로 덱을 조회한다.
3. 덱에서 첫번째에 있는 batch조각을 확인한다.
4. backoff와 sizelimit를 체크하여 해당 배치 조각이 다음전송에 포함될 수 있는지 확인한다.
5. 4번조건이 가능하면 ready list의 tail에 해당 배치를 추가하고, 기존 토픽 파티션 덱에서는 삭제한다.
6. 다음 파티션에 대해서도 같은 과정을 처리한다.
7. 다음 파티션에 대해서도 같은 과정을 처리한다.
8. check 단계에서 max.request.size 를 넘지 않는지 체크하고, 넘지 않아야 ready list에 batch 조각을 추가할 수 있다.
9. 하나의 노드에 대한 ready list 가 만들어지면, 노드별 ready list 에 추가한다.
10. 다음 노드에 대해서도 같은 작업을 반복한다.

### 2.2 전송

drain 과정에서 정리된 node 별 ready list 를 그대로 각 broker node 와의 connection 에서 보내면 된다. 이때 각 연결별로 sender thread 가 하나 뜨고 broker 와 통신한다. 메세지전송은 비동기로 동작한다.
데이터 전송은 한 연결에 동시에 여러개의 데이터(batch 조각)를 전송할 수 있다. 

전송중인데이터는 node 별 InFlightRequests Deque로 관리된다. 이 queue 의 사이즈는 **max.in.flight.requests.per.connection**  로 설정한다. KafkaProducer Client가 하나의Broker로 동시에 전송할 수 있는 요청 수를 의미한다.

![Untitled 4](https://user-
images.githubusercontent.com/56438131/226888487-e97fc64a-f7e6-459e-8cb7-be50c993e2d1.png)

### 3.1 Ack & Retry

### 3.1.1 Acks

producer client 가 보낸 메세지는 broker 를 ack 를 몇개를 받아야 성공으로 인정하는지를설정할 수 있다.

**acks = 0** : producer 는 ack를 기다리지 않고 성공으로 끝난다. 결과를 기다리지 않기 떄문에 가장 높은 throughput 을 얻을 수 있다. 다만, broker 가 정상적으로 받았는지 보장하지않으므로 데이터가 유실될 가능성이 있다.

**acks = 1**: 해당 데이터를 받는 topic partition 의 리더로부터만 ack 를 받으면 성공이다. 만약 해당 broker 가 데이터 복사를 완료하기 전에 죽는다면 데이터가 유실 될 가능성이 있다.

**acks = all**: 해당 데이터를 받는 topic partition 의 리더는 모든 replica 에 데이터 복사까지완료가 되어야 ack 메세지를 보낸다. 가장 안전한 옵션이지만, 그만큼 latency 가 증가한다.

### 3.1.2 Retry

acks=0인경우에는 retry할 이유가 없다. 하지만, 1,all인경우에는 retry를 하게되는데, 이과정에서 순서가 보장되지 않거나, 중복이 될 가능성이 있다. retry를 해도 순서를 보장하고 싶다면, max.inflight.request를 1로 제한해야하는데 이러면 처리량이 떨어진다.

### 3.2 idempotent Producer

Kafka Producer는 ack메커니즘과 retry동작 때문에 순서와 중복의 문제가 발생할 수 있다. 이것을 해결하는 방법으로 idempotent설정을 제공한다.

### 3.2.1 enable.idempotence

**enable.idempotence=true**로 설정하면 순서와 재처리시 중복제외가 보장이 된다. producer client를 특정 할 수 있는 producer id값, 각 메세지마다 부여하는 순차넘버링을 통해서 순서와 중복이 감지 될 수 있다. 프로듀셔 id별로 들어온 메세지에 대한 순차넘버링은 브로커가 메모리에 임시로 기억하고 있다가 retry되더라도 최종 partition에 저장할 때는 순서를 보장한다.

- enable.idempotence시 따라오는 설정
    
    acks = all
    
    retries = Integer.MAX
    
    max.inflight.requests= 5
    

병렬성을 포기하지 않고 idempotent를 달성할 수 있다.

### 4. Kafka Producer 성능 튜닝

### 4.1 순서를 보장하고 싶다면

위에서도 언급했듯이, max.inflight.requests=1로 두거나, enable.idempotence를 사용하면된다.

### 4.2 성능을 결정하는 기준

순서를 보장하지 않는다면, 여러가지 설정으로 throughput 을 증가시킬 수 있다.

아래 설정들은 그 특징을 파악하고, 내 어플리케이션의 평균 레코드 사이즈, 압축 후 레코드
사이즈, 초당 데이터 유입량(TPS), 기대 throughput, 최대 지연시간(latency) 등을 고려해서
**실험 방식**으로 튜닝해야한다.

### 4.2.1 acks

acks 를 줄일 수록 성능은 높아진다. acks=0 인 경우 비약적으로 향상하는데, 데이터 유실
율이 높아질 수 있으므로 특별한 경우를 제외하고는 사용하지 않는다.
acks=1 과 all 중에 선택해야한다. all 과 1 사이의 차이는 0만큼 크지는 않다.
대부분의 기업환경이라면 보수적으로 acks=all 을 많이 선택한다.
다른 모든 튜닝을 해도 안되고, 성능이 더 필요할 때 마지막으로 **acks=1** 을 고려한다.

### 4.2.2 batch.size

producer 는 한 번 데이터를 보내는 단위를 batch 조각으로 관리한다. 

하나의 batch 조각에는 압축된 record 여러개가 들어갈 수 있다. 

batch.size 가 클 수록 한 번 요청으로 보낼 수 있는 record 수가 많아진다.

**batch.size 가 너무 크다면 ,**

batch 조각의 단편화 으로 낭비되는 메모리 공간이 커진다.

batch 조각이 만들어지는데 더 긴 시간이 소요될 수 있다. ([ligner.ms](http://ligner.ms/) 가 꽤 크다면) 

유실이나 중복시 영향받는 레코드 수가 많아진다.

### 4.2.3 linger.ms

linger.ms는 하나의 batch 조각을 만들기 위해 얼마나 기다릴지 설정한다.
linger.ms = 0 이면 기다리지 않는다. 즉 하나의 메세지마다 하나의 batch 조각을 사용하게
된다.

linger.ms를 늘리면 그만큼 batch 에 담을 수 있는 record 수가 많아진다. 따라서 적은 요청
으로 더 많은 레코드를 보낼 수 있고 throughput 도 높아질 수 있다.
다만, linger.ms 만큼 전송이 지연될 수 있다는 의미도 된다.

예를 들어 linger.ms =5 라면, 해당 batch 조각의 첫번째 레코드 입장에서는 최대 5ms 를
기다린 뒤에야 전송이 시작되는 것이다.
latency 와 throughput 을 사이의 trade off 를 고려해서 값을 설정해야한다.

→ **요구사항 분석을 통해 조절 해가면서 실험적으로 조절.**

### 4.2.4 max.in.flight.requests.per.connection

하나의 broker 와의 연결당 in flight 상태(전송은 시작했고 아직 ack는 받지 못한 상태)로 최
대 몇개까지의 요청을 보낼 수 있는지 설정한다.

이 값이 1이라면, 모든 request (batch 조각)은 순서대로 보내진다. 이 값이 1보다 크면 동시
에 여러 요청이 전송된다. sender thread 는 전송을 비동기로 하기 때문에 동시에 여러 요청
을 보낼 수록 throughput 은 증가한다. [linger.ms](http://linger.ms/) 와 마찬가지로 이것이 늘어난다고해서 항
상 비례해서 throughput 이 증가하지는 않는다.

### 4.3 Delivery Time

send부터 broker까지 얼마나 걸리는지를 설정을 통해 유추 가능하다.

![Untitled 5](https://user-images.githubusercontent.com/56438131/226888466-17f5da00-ca95-45c5-91de-04629c366045.png)