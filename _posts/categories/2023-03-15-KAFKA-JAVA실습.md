---
title: KAFKA JAVA실습
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