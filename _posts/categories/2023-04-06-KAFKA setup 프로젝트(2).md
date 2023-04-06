---

layout: single
title: KAFKA setup 프로젝트(2)
categories:
  - Project
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-04-06
---


## 앱광고 유저 로그데이터 실시간으로 처리하기(2)

## 1. ImpClickGeneratorApp

(1)에서 각 Topic에 해당하는 event(Impression, Click, JoinedClick)에 해당하는 클래스를 생성하였고, 랜덤으로 ImpressionEvent의 필드들을 생성하고, 광고 id는 5개의 id로써, 랜덤으로 지정되게끔 지정하고, click이벤트는 impression이벤트의 impId를 공통필드로 생성할 수 있도록 파라미터를 지정하였다.

이제는 (1)에서 생성한 app들을 이용하여, 실제로 Event를 생성하고, Kafka Cluster에 넣어주는 작업을 진행하기위한 ImpClickGenratorApp을 작성한다. impressionEvent가 ClickEvent로 이어지는 것을 카운팅하는 것이 목적이기때문에, impression이 click으로 이어지는 비중을 임의로 50%로 설정하고, 생성한 Topic( Impression, Click)을 구분하여 Kafka Cluster에 넣어주는 작업을 작성한다.

### 1.1 build.gradle

```java
plugins {
    id 'java'
    id 'application'
// 어플리케이션으로 작동할것이므로, application으로 플러그인을 지정해준 후 , ClassName을
// application {'ClassName'} 작성해준다.
}

group = 'de-kafka'
version = '1.0-SNAPSHOT'

repositories {
    mavenCentral()
}

dependencies {

    implementation(project(":protocol"))
//(1)에서 작성한 라이브러리들이다.
}

application {
    mainClassName 'de.kafka.ad.generator.ImpClickGeneratorApp'
}

test {
    useJUnitPlatform()
}
```

### 1.2 ImpClickGenratorApp

```java
package de.kafka.ad.generator;

import de.kafka.protocol.Utils;
import de.kafka.protocol.constant.Topics;
import de.kafka.protocol.event.ClickEvent;
import de.kafka.protocol.event.ImpressionEvent;
import de.kafka.protocol.generator.RandomGenrator;
import de.kafka.protocol.serde.CustomJsonSerializer;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.RandomUtils;
import org.apache.commons.lang3.StringUtils;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

@Slf4j
public class ImpClickGeneratorApp {
    public static void main(String[] args) {
        if (args.length !=1 ){
            log.error("pass --bootstrapservers of Kafka Cluster as first argument");
            System.exit(1);

        }
				// application parameter로 EC2 Domain:9092로 넣어준다.
        String bootstrapServers = args[0];
        if (StringUtils.isBlank(bootstrapServers)){
            log.error("pass --bootstrapservers of Kafka Cluster as first argument");
            System.exit(1);

        }
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, CustomJsonSerializer.class.getName());
				// Value는 (1)에서 구현한 CustomJsonSearializer를 사용한다. (각 Topic별로 Serializing)

        try(Producer<String, Object>producer = new KafkaProducer<>(props)){
            while(true) {
                long currentTimeStamp = System.currentTimeMillis(); //현재시간을 ms로 받아온다.
                ImpressionEvent impressionEvent = RandomGenrator.genratorImpressionEvent(currentTimeStamp);
								// (1) RandomGenrator의 impressionEvent를 현 타임스탬프로 생성한다. 
                ProducerRecord<String, Object> impData = new ProducerRecord<>(Topics.TOPIC_IMP, impressionEvent.getImpId(), impressionEvent);
								// ProducerRecord타입으로 토픽은 impression, ImpId, impressionEvent로 레코드를 생성한다.
		
                producer.send(impData, (metadata, exception) -> {
								// Kafka Broker에 전송한다.
                            if (exception != null) {
                                log.error("fail to produce record " + impressionEvent);

                            } else {
                                Utils.printRecordMetadata(metadata);
                            }
                        }
                );
								// 0.5초
                Thread.sleep(500);
								
                if (RandomUtils.nextBoolean()) {
								// 50%의 확률로 클릭으로 이어진다 가정한다.
                    long impClickGap = RandomUtils.nextLong(100, 60 * 1000);
										// 광고를 시청하는 시간을 가정한다.
                    ClickEvent clickEvent = RandomGenrator.generatorClickEvent(currentTimeStamp + impClickGap, impressionEvent.getImpId());
										// clickEvent는 impressionEvent의 impId를 기준으로, 시간은 impressionTimeStamp + Gap으로 현실적이게 구성해준다.
			
                    ProducerRecord<String, Object> clickData = new ProducerRecord<>(Topics.TOPIC_CLICK, clickEvent.getImpId(), clickEvent);
                    producer.send(clickData, (metadata, exception) -> {
                                if (exception != null) {
                                    log.error("fail to produce record " + clickEvent);

                                } else {
                                    Utils.printRecordMetadata(metadata);
                                }
                            }
                    );

                }

                Thread.sleep(500);

            }

            }catch(Exception e){
            log.error(e.getMessage());
            e.printStackTrace();
        }

    }

}
```

## 2. Caches , ImpClickJoinApp

(1)에서 정의한 요구사항에 따르면, impressionEvent가 들어온후 특정 기간동안 impressionId를 유지하고 그 기간안에 다시 들어온 impId는 무시한다는 가정과 Click도 마찬가지로, 특정기간동안 중복해서 들어온 ImpId는 무시해야한다는 가정이 있으므로, 이것을 해결해야한다.

이를 위해 사용하는 것은 Cache를 사용할 것이다. 그중에서도 CaffeineCacheManager를 이용하여 캐시를 구현하였다.

### 2.1 Caches

```java
package de.kafka.ad.processor;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import com.github.benmanes.caffeine.cache.Expiry;
import de.kafka.protocol.event.ImpressionEvent;
import de.kafka.protocol.event.JoinedClickEvent;
import org.checkerframework.checker.index.qual.NonNegative;
import org.checkerframework.checker.nullness.qual.NonNull;

import java.time.Duration;
import java.util.concurrent.atomic.AtomicLong;

public class Caches {

    private static final Duration impValidWindow = Duration.ofMinutes(1L);

    private static final Duration clickDedupWindow = Duration.ofMillis(1L);
		// 두이벤트 둘다 정적인 윈도우 기간을 주어서 설정하였다.
    public static Cache<String, ImpressionEvent> CACHE_IMP =
            Caffeine.newBuilder()
                    .maximumSize(10_000)
                    .expireAfter(new Expiry<String, ImpressionEvent>() {
                        @Override
                        public long expireAfterCreate(@NonNull String key, @NonNull ImpressionEvent value, long currentTime) {
                            return impValidWindow.toNanos();
														// 생성후 파기될때까지의 duration을 정적으로 주었고, ms->ns로 맞춰 파라미터형식을 맞추었다.
                        }
												// 수정이나 읽은후에도 처음 생성시 지정된 duration을 유지하도록 구성하였다.

                        @Override
                        public long expireAfterUpdate(@NonNull String key, @NonNull ImpressionEvent value, long currentTime, @NonNegative long currentDuration) {
                            return currentDuration;
                        }

                        @Override
                        public long expireAfterRead(@NonNull String key, @NonNull ImpressionEvent value, long currentTime, @NonNegative long currentDuration) {
                            return currentDuration;
                        }

                    }).build();

    public static Cache<String, JoinedClickEvent> CACHE_CLICK =
            Caffeine.newBuilder()
                    .maximumSize(10_000)
                    .expireAfter(new Expiry<String, JoinedClickEvent>() {
                        @Override
                        public long expireAfterCreate(@NonNull String key, @NonNull JoinedClickEvent value, long currentTime) {
                            return clickDedupWindow.toNanos();
                        }

                        @Override
                        public long expireAfterUpdate(@NonNull String key, @NonNull JoinedClickEvent value, long currentTime, @NonNegative long currentDuration) {
                            return currentDuration;
                        }

                        @Override
                        public long expireAfterRead(@NonNull String key, @NonNull JoinedClickEvent value, long currentTime, @NonNegative long currentDuration) {
                            return currentDuration;
                        }

                    }).build();
		// Ad_id별 count를 셀 캐시로써, 따로 duration을 설정하지 않았다.
    public static Cache<String , AtomicLong>CACHE_CLICK_COUNT_BY_AD_ID=
            Caffeine.newBuilder()
                    .maximumSize(10_000)
                    .build();

}
```

### 2.2 ImpClickJoinApp

ImpClickJoinApp은

1.  **ImperssionEvent를 Consume하면 이를 CACHE_IMP로 put하고,** 
2. **만일 ClickEvent를 Consume하게된다면** 
    
    2.1 **CACHE_IMP에 현재 ClickEvent의 ImpId가 CACHE_IMP에 등록되어있는지 조회한다.** 
    
    2.2 **만일 널값이 아닌 ImpressionEvent를 반환받는다면, 이는 Counting해야하는 값이므로, 두이벤트를 조인해야한다.** 
    
    2.3 **(1)에서 생성한 JoinClickEvent 클래스의 from메소드를 통해 Event를 Return받고, 만일 click이 여러번(duplicated)되었을 수도 있기때문에, 이를 CACHE_CLICK에 조회하여 만일 null값이라면, Kafka Cluster에 JoinedClickEvent로 삽입해주고, CACHE_CLICK에 put해줘야한다.**
    

1. **그리고, KafkaCluster에 JoinedClickTopic으로 삽입해주었던 Event가 Consume된다면, 이는 AdID를 기준으로 count해준다.**

```java
package de.kafka.ad.processor;

import de.kafka.protocol.Utils;
import de.kafka.protocol.constant.Topics;
import de.kafka.protocol.event.ClickEvent;
import de.kafka.protocol.event.ImpressionEvent;
import de.kafka.protocol.event.JoinedClickEvent;
import de.kafka.protocol.generator.RandomGenrator;
import de.kafka.protocol.serde.CustomJsonDeserializer;
import de.kafka.protocol.serde.CustomJsonSerializer;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.RandomUtils;
import org.apache.commons.lang3.StringUtils;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;

import java.time.Duration;
import java.util.Arrays;
import java.util.Properties;
import java.util.concurrent.atomic.AtomicLong;

import static de.kafka.ad.processor.Caches.*;

@Slf4j
public class ImpClickJoinApp {
    public static void main(String[] args) {
        if (args.length !=1 ){
            log.error("pass --bootstrapservers of Kafka Cluster as first argument");
            System.exit(1);

        }
        String bootstrapServers = args[0];
        if (StringUtils.isBlank(bootstrapServers)){
            log.error("pass --bootstrapservers of Kafka Cluster as first argument");
            System.exit(1);

        }
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,bootstrapServers);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, CustomJsonDeserializer.class.getName());
        props.put(ConsumerConfig.GROUP_ID_CONFIG,"de-kafka-cousumer-3");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG,"earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG,"true");

        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,bootstrapServers);
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,StringSerializer.class.getName());
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,CustomJsonSerializer.class.getName());

        try(Producer<String, Object> producer = new KafkaProducer<>(producerProps);
            Consumer<String, Object> consumer = new KafkaConsumer<>(props);){
            consumer.subscribe(Arrays.asList(Topics.TOPIC_IMP,Topics.TOPIC_CLICK,Topics.TOPIC_JOINED_CLICK));

            while(true) {
                ConsumerRecords<String , Object> records = consumer.poll(Duration.ofMillis(500));
                for (ConsumerRecord<String, Object> record : records) {
                    Utils.printConsumerRecord(record);
                    if(record.value() instanceof ImpressionEvent){
                        ImpressionEvent impressionEvent = (ImpressionEvent) record.value();
                        CACHE_IMP.put(impressionEvent.getImpId(),impressionEvent);
                        log.info("successed to put imp {}",impressionEvent);

                    }else if (record.value() instanceof ClickEvent){
                        ClickEvent clickEvent = (ClickEvent) record.value();
                        ImpressionEvent impressionEvent = CACHE_IMP.getIfPresent(clickEvent.getImpId());
                        if (impressionEvent !=null) {
                            JoinedClickEvent joinedClickEvent = JoinedClickEvent.from(impressionEvent, clickEvent);
                            JoinedClickEvent duplicatedJoinedClick = CACHE_CLICK.getIfPresent(joinedClickEvent.getImpId());
                            log.info("successed to put imp {}", clickEvent);
                            if (duplicatedJoinedClick == null) {
                                ProducerRecord<String, Object> joinedClickData = new ProducerRecord<>(Topics.TOPIC_JOINED_CLICK,
                                        joinedClickEvent.getImpId(), joinedClickEvent);
                                CACHE_CLICK.put(joinedClickEvent.getImpId(),joinedClickEvent);
                                producer.send(joinedClickData, (metadata, exception) -> {
                                    if (exception != null) {
                                        System.err.println("Fail to produce Record" + joinedClickData);

                                    } else {
                                        Utils.printRecordMetadata(metadata);
                                    }
                                });
                                log.info("successed to send new joined click - {}", joinedClickEvent);
                            } else {
                                log.info("duplicated to joined click - {}", duplicatedJoinedClick);

                            }
                        } else {
                            log.info("cannot join click with imp = {}",clickEvent);
                            }

                        } else if (record.value() instanceof JoinedClickEvent) {
                        JoinedClickEvent joinedClickEvent = (JoinedClickEvent) record.value();
                        log.info("joined Click");
                        String adId = joinedClickEvent.getAdId();
                        AtomicLong count =CACHE_CLICK_COUNT_BY_AD_ID.getIfPresent(adId);
                        if(count ==null){
                            count = new AtomicLong();

                        }
                        log.info("click count of ad {} : {}",adId,count.incrementAndGet());
                        CACHE_CLICK_COUNT_BY_AD_ID.put(joinedClickEvent.getAdId(),count);

                    } else {
                        log.info("Unknown Event");
                    }

                }

                }

        }catch(Exception e){
            log.error(e.getMessage());
            e.printStackTrace();
        }

    }

}
```

 

## 3. 결과

1. **ImpClickGenerator App구동화면**

![click-generator-app-running](https://user-images.githubusercontent.com/56438131/230400059-9b569469-8086-4bd6-ba2f-d36f35b5ba47.PNG)

impressionEvent와 ClickEvent를 잘 생성하는 것을 볼 수 있다.

1. **ImpClickJoinApp을 구동화면**

![joined-click-count-app](https://user-images.githubusercontent.com/56438131/230400019-c3b64843-b210-455b-b70c-ba212f72e858.PNG)
성공적으로, Imression, Click, JoinedClick를 Consume하며, JoinedClickEvent에 해당하는 Consume값은 AdId를 기준으로 잘 카운트 하는 것을 알 수 있다.