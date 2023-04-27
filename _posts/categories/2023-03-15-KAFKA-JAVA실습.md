---
title: KAFKA (Producer,Consumer) 실습
layout: single
categories: 
   - DataEngineering
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-03-15
---
## KAFKA (Producer,Consumer) 실습

### Producer

intellij에서 JAVA를 통해 위에서 KAFKA 쉘에서 간단히 실행하였던 Producer를 구현한다.

### 1. build.gradle

```java
plugins {
id 'java'
}
group 'de.kafka'
version '1.0-SNAPSHOT'
repositories {
mavenCentral()
}
dependencies {
testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'
implementation 'org.apache.kafka:kafka-clients:3.2.3'
implementation 'org.slf4j:slf4j-simple:1.7.30'
}
test {
useJUnitPlatform()
}
```

### 2. Main

```java
package de.kafka.basic.producer;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.apache.kafka.common.serialization.StringSerializer;
import java.util.Properties;
import java.util.Scanner;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
public class Main {
	public static void main(String[] args) {
		Scanner scanner = new Scanner(System.in);
		Properties props = new Properties();
		// NODE1, NODE2, NODE3을 브로커 서버 IP로 변경
		String BOOTSTRAP_SERVERS = "NODE1:9092,NODE2:9092,NODE3:9092";
		String TOPIC_NAME = "quickstart-events";
		props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
		props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.g
		etName()); // 입력할 메시지의 key값의 형식
		props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.clas
		s.getName()); // 입력할 메세지의 형식
		Producer<String, String> producer = new KafkaProducer<>(props); // 메시지의 형식에 맞게 generics설정
		while (true) {
				String key = null;
				System.out.println("Do you want to set a partition key? (y/n)");
				String isKeyRequired = scanner.nextLine();
				if (isKeyRequired.equals("y")) {
						System.out.println("Enter partition key: ");
						key = scanner.nextLine();
				}
				System.out.println("Enter string to send: ");
				String data = scanner.nextLine();
				ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, k
				ey, data);
				Future<RecordMetadata> result = producer.send(record);
				producer.flush();
				try {
						RecordMetadata metadata = result.get();
						System.out.println("Message sent successfully to partition " + metadat
a.partition() + " with offset " + metadata.offset());
				} catch (InterruptedException | ExecutionException e) {
						e.printStackTrace();
				}
				System.out.println("Do you want to send another message? (y/n)");
				String isContinue = scanner.nextLine();
				if (isContinue.equals("n")) {
						break;
				}
		}
		producer.close();
	}
}
```

다음과 같이 JAVA로 Producer를 구현이 가능하다.

record를 전송할때 비동기처리를 하여, 아래의 예외처리 구문을 대체할 수 있게 구성하여도 된다.

```java
Future<RecordMetadata> result = producer.send(record,((metadata, exception) -> {
                if(exception!=null){
                    System.err.println("fail to record"+data);
                }
                else{
                    System.out.println("Message sent successfully to partition " + metadata.partition() + " with offset " + metadata.offset());
                }

                }
                )
            );
```

### Consumer

build.gradle은 동일하게 처리한다.

### 1.Main

```java
package de.kafka.basic.consumer;
import org.apache.kafka.clients.consumer.Consumer;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        Properties props = new Properties();

        String BOOTSTRAP_SERVERS = "13.125.34.64:9092,3.39.222.85:9092,43.201.86.33:9092";
        String TOPIC_NAME = "quickstart-events";
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test01");
				//consumer group id로 계속다르게 지정해주어야한다.
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
				// Kafka에 초기 오프셋이 없거나 서버에 현재 오프셋이 더 이상 없는 경우 수행할 작업
				//"What to do when there is no initial offset in Kafka or if the current offset does not exist any more on the server (e.g. because that data has been deleted): 
				//<ul><li>earliest: automatically reset the offset to the earliest offset<li>
				//latest: automatically reset the offset to the latest offset</li><li>
				//none: throw exception to the consumer if no previous offset is found for the consumer's group</li><li>anything else: throw exception to the consumer.</li></ul>";
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
				// 자동 커밋이 기본값이지만, 요구사항으로 0번 파티션은 commit하지 않을 것이다라면 false후 아래에서 consumer.commitSync을 이용해야한다.

        Consumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList(TOPIC_NAME));
        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.println(
                            "Received message: (" + record.key() + ", " + record.value
                                    () + ") at " + "topic "
                                    + record.topic() + " partition " + record.partition() +
                                    "offset " + record.offset());
                }
            }
        } finally {
            consumer.close();
        }
    }
}
```

### 2. 설정

**group.id** : consumer는 같은 group id 단위로 offset 관리를 한다. consumer 가 종료되더라도 group id 를 기준으로 commit 된 offset 정보는 kafka 에 유지되고, 이에 따라 같은 group id를 가진 consumer가 다음 consume 시 받아오는 데이터는 그 commit된 offset 이후부터 가져온다.

**auto.offset.reset** : 해당 consumer group id 에 대한 offset 정보가 없을 때, 현재kafka의 토픽에 존재하는 데이터중에 어디부터 가져올지.

1. **latest(default)**: 마지막 부터 가져온다. 즉, consumer 가 붙은 이후부터 produce된 데이터부터 가져온다.

2. **earliest**: 현재 kafka 에 존재하는 데이터 중 가장 빠른(오래된) 것부터 가져온다. 이때 주의할 점은 쌓인 데이터 양에 따라서 초반에 데이터 consume양이 많아지므로 consumer 에 부담이 될 수 있다.

### 1. Auto Commit = False

만일 자동 커밋을 하게 설정하지 않고, 특정 요구사항에 따른 메뉴얼 커밋을 해야한다는 상황을 가정했을때,

<aside>
💡 props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, “false");

</aside>

다음과 같이 설정해야한다. 하지만 이는 수많은 커밋으로 이어질 수 있고, 이는 시스템의 부하를 일으킬 수 있기 때문에 주의해야한다.

만일 동일 Topic에 대해서 가지는 파티션중에서 0번 파티션을 커밋을 하지말고, 계속 유지를 해야한다는 가정을 두고 Consumer코드를 작성해보자

```java
package de.kafka.basic.consumer;
import java.time.Duration;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;
import java.util.Properties;
import java.util.Scanner;
import org.apache.kafka.clients.consumer.Consumer;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.OffsetAndMetadata;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;

public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);
        Properties props = new Properties();
// NODE1, NODE2, NODE3을 브로커 서버 IP로 변경
        String BOOTSTRAP_SERVERS ="ec2-43-201-104-214.ap-northeast-2.compute.amazonaws.com:9092,ec2-3-36 -49-89.ap-northeast-2.compute.amazonaws.com:9092,ec2-3-38-153-68.ap-northeast-2.comput e.amazonaws.com:9092";
        String TOPIC_NAME = "quickstart-events";
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "test3");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
        Consumer<String, String> consumer = new KafkaConsumer<>(props);
        consumer.subscribe(Collections.singletonList(TOPIC_NAME));
        try {
            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

                Map<TopicPartition, List<Long>> processedTopicPartitionOffsets = new HashMap<>();

                for (ConsumerRecord<String, String> record : records) {
                    System.out.println(
                            "Received message: (" + record.key() + ", " + record.value() + ") at "
                                    + "topic "+ record.topic() + " partition " + record.partition() + "offset "
                                    + record.offset());
                    if (record.partition() != 0) {
                        TopicPartition topicPartition = new TopicPartition(record.topic(), record.partition());

                        if (processedTopicPartitionOffsets.containsKey(topicPartition)) {

                            List<Long> offsets = processedTopicPartitionOffsets.get(topicPartition);

                            offsets.add(record.offset());
                        }else {
                            List<Long> offsets = new ArrayList<>();
                            offsets.add(record.offset());
                            processedTopicPartitionOffsets.put(topicPartition, offsets);

                        }
                    }
                }
                Map<TopicPartition, OffsetAndMetadata> commitOffset = new HashMap();

                for (Entry<TopicPartition, List<Long>> entry : processedTopicPartition

                Offsets.entrySet()) {

                    List<Long> offsets = entry.getValue();
                    offsets.sort(Comparator.comparingLong(Long::longValue).reversed());

                    Long latestProcessedOffset = offsets.get(0);
                    System.out.println("latest processed offset of topic partition - "+ entry.getKey()
                            + ": " + latestProcessedOffset);
                    commitOffset.put(entry.getKey(), new OffsetAndMetadata(latestProcessedOffset + 1));
                }
                consumer.commitSync(commitOffset);
            }
        } finally {
            consumer.close();
        }
    }
}

```

```java
# 코드 중 핵심적인 부분
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");
# 컨슈머의 auto commit 설정을 false로 지정하였다.

Map<TopicPartition, List<Long>> processedTopicPartitionOffsets = new HashMap<>();
# HashMap으로 Generics를 1.토픽파티션 2.오프셋을 담는 Long형 리스트으로 구성하였다.
# 이는 TopicPartion별로 오프셋정보를 기록할 수 있다.

if (record.partition() != 0) {
# record는 루프를 돌면서 conusmer가 구독하는 값이다. (기본적으로 토픽파티션정보와 오프셋정보를 입력을 받기때문에 필드로 존재한다.)
# 요구사항인 파티션이 0번파티션이 아닌경우에만,
            TopicPartition topicPartition = new TopicPartition(record.topic(), record.partition());
						# 위에서 선언한 processedTopicPartionOffsets의 Generic을 보면 알 수 있듯이
						# TopicPartion이 키값으로 사용되므로, record가 지니고있는 토픽과 파티션을 이용하여,
						# 객체를 선언해준것이다.						
            if (processedTopicPartitionOffsets.containsKey(topicPartition)) {
						# 만일 이미 해시맵에 존재한다면, (이건 0이아닌 파티션이다.)

	          List<Long> offsets = processedTopicPartitionOffsets.get(topicPartition);
						# 이미 존재하기에, hashmap에서 topicpartion에 해당하는 offsets를 get으로 불러와
						
            offsets.add(record.offset());
						# 뒤에 add시켜주면된다.
            }else {
						# 0이아닌 파티션인데 존재 하지 않는다면,
                      List<Long> offsets = new ArrayList<>();
											# 새로운 offset리스트를 선언해줘야하며,
                      offsets.add(record.offset());
											# 현재들어온 레코드의 offset을 기록하고,
                      processedTopicPartitionOffsets.put(topicPartition, offsets);
											# 해시맵에 put해준다.

                   }

```

```java
Map<TopicPartition, OffsetAndMetadata> commitOffset = new HashMap();
# consumer의 commitSync의 파라미터로 들어가는 <TopicPartition, OffsetAndMetadata>타입으로 HashMap을 생성한다.

      for (Entry<TopicPartition, List<Long>> entry : processedTopicPartitionOffsets.entrySet()) {
			# hashMap을 iter하기 위한 Entry셋으로 캐스팅한다.
               List<Long> offsets = entry.getValue();
							 # 오프셋정보를 list로 받아오고,
               offsets.sort(Comparator.comparingLong(Long::longValue).reversed());
							# 내림차순 정렬한다. ( commitSync는 오프셋의 마지막 +1을 요구한다.)

               Long latestProcessedOffset = offsets.get(0);
							# 0번지(가장큰 오프셋정보)를 저장하고
               System.out.println("latest processed offset of topic partition - "+ entry.getKey()
                            + ": " + latestProcessedOffset);
               commitOffset.put(entry.getKey(), new OffsetAndMetadata(latestProcessedOffset + 1));
							# key값과 가장큰 오프셋정보+1를 commitOffset에 넣고,
           }
           consumer.commitSync(commitOffset);
					# 커밋한다.
```