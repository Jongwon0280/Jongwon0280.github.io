---

layout: single
title: KAFKA setup 프로젝트(1)
categories:
  - Project
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-04-05
---


## 앱광고 유저 로그데이터 실시간으로 처리하기

## 1. Scenario

### 1.1 요구사항

1. mobile앱에서 광고를 요청시, 광고 데이터를 받는다.
2. 광고를 유저가 보면 impression data를 발생시킨다.
3. impression 데이터는 광고를 식별하기 위한 주요정보가 담겨있다.
    1. impId
    2. requestId
    3. adId
    4. userId
    5. deviceId
    6. inventoryId
4. 유저가 광고를 클릭하면 클릭데이터를 발생시킨다.
5. 클릭데이터는 impression Data이후 발생하는 데이터로써, 필수적인 정보만을 생성한다.
    1. impId
    2. clickURL

1. 정상 impression은 다음조건으로 판단한다.
    1. 한번 발생한 impId는 정해진 기간동안만 유효하다.
    2. 정해진기간은 정책에 따라 설정가능하다.
    
2. 정상 click event는 다음 조건으로 판단한다.
    1. 같은 impId인 impression이  impression기간동안 발생한적이 있어야한다.
        
        → click만 되었다는 것은 적절한 경로로 들어온것이 아니기때문에 관심대상이 아니다.
        
    2. 같은 impId로 발생한 click은 click기간 동안 하나만 유효하다.
    3. impression기간 및 click기간은 설정이가능하다.
3. 실시간 정상 click event는 adId기준으로 count한 결과를 확인할 수 있어야한다.

 

## 2. Architecture

![Untitled](https://user-images.githubusercontent.com/56438131/230126917-3a211c9f-b99c-4fe9-bf21-f886eb368370.png)

위와 같은 시나리오를 반영하기 위해 **1) , 2)**를 발생시키는 ImpClickGeneratorApp을 생성하고 Kafka Cluster에 impression관련 토픽과 click관련 토픽을 생성한다.

1. 광고를 유저가 확인하여 발생시키는 데이터인 Impression Data
2. Impression Data 발생 후 주어진 광고를 클릭하여 발생하는 ClickData

여기서 중요한 점은 producer로 send하기 위한 producerRecord <key , value>에서 키값을 impId로 지정한다는 것이 중요하다. impression Data와 Click Data와의 공통키 값은 impId이기때문이다. 이후 정상적인 절차(impression → click)를 거친 데이터들은 광고아이디 기준으로 counting해주어야하므로, impData와 ClickData를 역정규화시킨후 이를 kafka클러스터에 전송하고, adId를 기준으로 counting해줌으로써, 실시간으로 adID의 count결과를 확인할 수 있게한다.

## 3. Topic ~ Serializer … Utils

### 3.1 build.gradle & Topics

```java
plugins {
    id 'java-library'
}
```

```java
package de.kafka.protocol.constant;

public interface Topics {
    String TOPIC_IMP = "impression-event";
    String TOPIC_CLICK = "click-event";

    String TOPIC_JOINED_CLICK = "joined-click-event";

}

// 3개의 토픽 ( impression, click, joined-click)으로 구성하며, interface로 구성.
```

### 3.2 Event 클래스 정의

**3.2.1 Impression Event**

```java
package de.kafka.protocol.event;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.jackson.Jacksonized;

@Data
@Jacksonized
@NoArgsConstructor
@Builder
@AllArgsConstructor

// lombok은 getter나 setter와 같은 메소드를 annotation으로 자동생성.
// jackson은 kafka의 data를 serialization and deserialization할때마다, 토픽별로 만들어주는 것을
// 간단하게 도와준다.

public class ImpressionEvent {
    private String impId;
    private String requestId;
    private String adId;
    private String userId;
    private String deviceId;
    private String inventoryId;
    private long timestamp;

}
```

**3.2.1 Click Event**

```java
package de.kafka.protocol.event;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.jackson.Jacksonized;

import java.sql.Timestamp;

@Data
@Jacksonized
@NoArgsConstructor
@Builder
@AllArgsConstructor

// ImpressionEvent의 impId로 연결가능하기에, 필수 정보만 보관한다.
public class ClickEvent {
    private String impId;
    private String clickUrl;
    private long timestamp;

}
```

**3.2.3 Joined Event**

```java
package de.kafka.protocol.event;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.extern.jackson.Jacksonized;

@Data
@Jacksonized
@NoArgsConstructor
@Builder
@AllArgsConstructor
public class JoinedClickEvent {
    private String impId;
    private String requestId;
    private String adId;
    private String userId;
    private String deviceId;
    private String inventoryId;

    private long impTimestamp;
	// impression시의 timestamp
    private String clickUrl;
    private long timestamp;
	// click했을시 timestamp

    public static JoinedClickEvent from(ImpressionEvent impressionEvent, ClickEvent clickEvent){
        return new JoinedClickEvent(
                impressionEvent.getImpId(),
                impressionEvent.getRequestId(),
                impressionEvent.getAdId(),
                impressionEvent.getUserId(),
                impressionEvent.getDeviceId(),
                impressionEvent.getInventoryId(),
                impressionEvent.getTimestamp(),
                clickEvent.getClickUrl(),
                clickEvent.getTimestamp()

        );

    }

}
```

### 3.3 Random Generator

우리는 사용자가 impression 및 click했다는 것을 얻기위해 랜덤으로 생성하도록 할것이다.

일단 광고id는 10자리 랜덤 알파벳으로 5개정도로만 구성할 것이고, impEvent의 필드들도 랜덤으로 구성한다. clickEvent의 impId는 선행된 impEvent의 impId를 써야하므로, 파라미터로 넘겨주도록한다.

```java
package de.kafka.protocol.generator;

import java.util.Arrays;
import java.util.List;
import org.apache.commons.lang3.RandomStringUtils;
import org.apache.commons.lang3.RandomUtils;

import de.kafka.protocol.event.ClickEvent;
import de.kafka.protocol.event.ImpressionEvent;

public class RandomGenrator {
    private static List<String> AD_IDS = Arrays.asList(
            RandomStringUtils.randomAlphabetic(10),
            RandomStringUtils.randomAlphabetic(10),
            RandomStringUtils.randomAlphabetic(10),
            RandomStringUtils.randomAlphabetic(10),
            RandomStringUtils.randomAlphabetic(10)
    );

    // impressionEvent Class field info
//     private String impId;
//    private String requestId;
//    private String adId;
//    private String userId;
//    private String deviceId;
//    private String inventoryId;
//    private long timestamp;
//
    public static ImpressionEvent genratorImpressionEvent(long timestamp) {
        return new ImpressionEvent(
                RandomStringUtils.randomAlphabetic(10),
                RandomStringUtils.randomAlphabetic(10),
                AD_IDS.get(RandomUtils.nextInt(0,5)),
                RandomStringUtils.randomAlphabetic(10),
                RandomStringUtils.randomAlphabetic(10),
                RandomStringUtils.randomAlphabetic(10),
                timestamp
                );
    }
// ClickEvent Class Field info
//    private String impId;
//    private String clickUrl;
//    private long timestamp;

    public static ClickEvent generatorClickEvent(long timestamp, String impId){
        return new ClickEvent(
                impId,
                RandomStringUtils.randomAlphabetic(10),
                timestamp
        );
    }
}
```

### 3.4 Serializer & Deserializer (Custom)

Kafka Broker에 데이터를 집어넣기위해선 Serialize가 필요하다. 보통 <Key , Data>형태로 구성되어 있기에, Key와 같은경우 Kafka.common.serialization에는 구현되어있다.

```java

package org.apache.kafka.common.serialization;

import org.apache.kafka.common.errors.SerializationException;

import java.io.UnsupportedEncodingException;
import java.nio.charset.StandardCharsets;
import java.util.Map;

public class StringSerializer implements Serializer<String> {
    **private String encoding = StandardCharsets.UTF_8.name();**

    @Override
    **public byte[] serialize(String topic, String data) {
        try {
            if (data == null)
                return null;
            else
                return data.getBytes(encoding);
        } catch (UnsupportedEncodingException e) {
            throw new SerializationException("Error when serializing string to byte[] due to unsupported encoding " + encoding);
        }
    }**
}
```

하지만, Value 같은 경우 우리는 3가지 토픽을 구성할 것이고 위에서 각 Topic별로 삽입하는 Event클래스가 다르기에, custom하게 Serializer와 Deserializer를 만들어주어야한다. 위를 참고하여 토픽에 맞게 Serializing해야하므로, 다음과 같이 구현하였다.

```java
package de.kafka.protocol.serde;

import de.kafka.protocol.constant.Topics;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.commons.lang3.SerializationException;
import org.apache.kafka.common.serialization.Serializer;

public class CustomJsonSerializer implements Serializer<Object> {
    private ObjectMapper objectMapper= new ObjectMapper();

    @Override
    public byte[] serialize(String topic, Object data) {
        try {
            if (data == null) {
                System.out.println("Null recevied at serializing. topic : " + topic);
                return null;

            }
            System.out.println(
                    "Serializing..."
            );
            switch (topic) {
                case Topics.TOPIC_IMP:
                case Topics.TOPIC_CLICK:
                case Topics.TOPIC_JOINED_CLICK:
                    return objectMapper.writeValueAsBytes(data);
                default:
                    System.out.println("unknown topic : " + topic);
            }

            return null;
        }catch(Exception e){
            throw new SerializationException("Error with Serializing");

        }

    }
}
```

Deserializer같은 경우는 각 consumer가 Event클래스에 구성되어있는대로, 데이터를 변환해서 받아야하므로, 다음과 같이 구현하였다.

```java
package de.kafka.protocol.serde;

import de.kafka.protocol.constant.Topics;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ser.Serializers;
import de.kafka.protocol.event.ClickEvent;
import de.kafka.protocol.event.ImpressionEvent;
import de.kafka.protocol.event.JoinedClickEvent;
import org.apache.commons.lang3.SerializationException;
import org.apache.kafka.common.serialization.Deserializer;
import org.apache.kafka.common.serialization.Serializer;

public class CustomJsonDeserializer implements Deserializer<Object> {
    private ObjectMapper objectMapper= new ObjectMapper();

    @Override
    public Object deserialize(String topic, byte[] data) {
        try {
            if (data == null) {
                System.out.println("Null recevied at deserializing ");
                return null;

            }
            System.out.println(
                    "Deserializing...topic : "+ topic
            );
            switch (topic) {
                case Topics.TOPIC_IMP:
                    return objectMapper.readValue(new String(data,"UTF-8"), ImpressionEvent.class);

                case Topics.TOPIC_CLICK:
                    return objectMapper.readValue(new String(data,"UTF-8"), ClickEvent.class);
                case Topics.TOPIC_JOINED_CLICK:
                    return objectMapper.readValue(new String(data,"UTF-8"), JoinedClickEvent.class);
                default:
                    System.out.println("unknown topic : " + topic);
            }

            return null;
        }catch(Exception e){
            if(data!=null){
                System.err.println("error occured by data "+ new String(data));

            }
            throw new SerializationException("Error when deserializing byte[] to Class");

        }

    }
}
```

### 3.5 Utils

우리가 사용하는 모듈의 플러그인은 library로 다른 어플리케이션 모듈에서 가져다쓰는 역할이다. 그러므로 마지막으로 Utils에 관련노드정보들을 출력해주는 메소드까지 구현해놓는다.

```java
package de.kafka.protocol;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

public class Utils {
//producerRecord 전송 시, KafkaProducer의 metadata를 통해 메시지 전송여부 및 
// 파티션 ,오프셋, 토픽정보 출력 
    public static void printRecordMetadata(RecordMetadata metadata){
        System.out.println(
                "Message sent successfully to " + metadata.topic() + metadata.partition()
                + "with offset" +metadata.offset()
        );

    }
// Consumer입장에서 각각의 토픽, key, value, partition, offset 출력
    public static void printConsumerRecord(ConsumerRecord record){
        System.out.println(
                "Recv Message : (" + record.key() + ", " + record.value()+ ") at "+ "topic "+
                        record.topic()+ " partition : " + record.partition() + "offset : "+ record.offset()
        );
    }
}
```

여기까지가, 시나리오를 위한 기본 설정이 끝났다. 다음 포스팅에서는 위의 RandomGenerator와 CustomSerializer를 활용하여 impressionEvent와 ClickEvent를 생성하고 KafkaProudcer에 ImpId를 key값으로 해서 insert하는 것을 다루어보도록 하겠다.