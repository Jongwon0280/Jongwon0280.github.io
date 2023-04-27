---

layout: single
title: KAFKA Connect 개요 및 실습
categories:
  - DataEngineering
author_profile: true
toc: true
toc_label: "목록"
toc_icon: "bars"
toc_sticky: true
date: 2023-04-27
---

## KAFKA Connect 개요 및 실습

Kafka Connect는 Kafka에서 외부 시스템으로 또는 외부 시스템에서 Kafka로 데이터를 스트리밍하는 프레임워크다. Confluent나 Debezium에서는 RDBMS와 HDFS 등 다양한 시스템과 Kafka 간에 데이터 스트리밍 파이프라인을 쉽게 구축할 수 있도록 Connector plugin을 제공하고 있다. Kafka Connect의 구성 요소는 다음과 같다.

- **Connector:** Task를 관리하고 데이터 스트리밍 모니터링
- **Task:** 데이터를 Kafka로 또는 Kafka에서 복사
- **Worker:** Connector와 Task를 실행하는 프로세스
- **Converter:** Connect와 데이터를 송수신할 때 데이터 역/직렬화
- **Transform:** Connector에서 생성하거나 Connector로 전송되는 메시지 내용 변환

## 1. mode

### 1.1 Standalone 모드

Standalone 모드는 단일 프로세스가 모든 Connector 및 Task 실행을 담당하는 가장 단순한 모드다. 단일 프로세스만 있기 때문에 최소한의 구성만으로도 작동할 수 있다는 장점이 있지만, 확장성이 낮고 기능이 제한적이다. 

### 1.2 Distributed 모드

Distributed 모드는 Kafka Connect에 대한 확장성을 제공한다. Distributed 모드에서는 `group.id`를 사용해서 Worker들을 클러스터링하고, 예기치 않게 Worker가 중단될 경우 자동으로 새로운 Connector와 Task를 배포해서 클러스터를 재조정한다.

## 2. Converter와 Transformer

Task는 Converter를 사용하여 데이터 타입을 바이트에서 Connect 내부 데이터 타입 또는 그 반대로 변환한다. 다음 그림은 JDBC Source Connecter를 사용하여 데이터베이스에서 read하고, Kafka에 write하고, HDFS Sink Connector를 사용하여 HDFS에 write할 때 Converter가 사용되는 방법을 보여준다.

![Untitled](https://user-images.githubusercontent.com/56438131/234896345-2e240339-7c4d-4031-b3ce-c007f64f3da7.png)

출처: [https://docs.confluent.io/platform/current/connect/concepts.html#converters](https://docs.confluent.io/platform/current/connect/concepts.html#converters)

Converter가 데이터 타입을 변환하는 도구라면, Transformer는 데이터 콘텐츠를 수정하는 도구다. Transformer는 간단한 로직을 사용해서 개별 메시지를 수정한다. 만약 복잡한 수정 로직을 여러 메시지에 적용하고 싶다면 ksqlDB 및 Kafka Streams를 사용하는 것이 좋다.

## 3. Kafka JDBC Connector

![Untitled 2](https://user-images.githubusercontent.com/56438131/234896494-919e6b92-fce7-463a-8346-6ff1b9ae8eee.png)

출처: [https://www.confluent.io/blog/no-more-silos-how-to-integrate-your-databases-with-apache-kafka-and-cdc/](https://www.confluent.io/blog/no-more-silos-how-to-integrate-your-databases-with-apache-kafka-and-cdc/)

![Untitled 1](https://user-images.githubusercontent.com/56438131/234896630-bb287c32-7783-4c8d-9ed8-be645beeea51.png)

출처: [https://www.confluent.io/blog/simplest-useful-kafka-connect-data-pipeline-world-thereabouts-part-1/](https://www.confluent.io/blog/simplest-useful-kafka-connect-data-pipeline-world-thereabouts-part-1/)

앞서 말한 것처럼 Confluent에서는 Kafka Connect의 plugin으로 사용할 수 있는 다양한 Connector를 제공하고 있다. ([https://www.confluent.io/product/connectors/](https://www.confluent.io/product/connectors/))

### 3.1 JDBC Connector 의 기능

Kafka Connect용 Confluent JDBC Connector를 사용하면 Kafka와 JDBC를 지원하는 모든 RDBMS 간에 데이터 스트리밍이 가능하다. 전체 데이터를 가져올 수도 있고, 숫자 컬럼이나 업데이트 타임스탬프를 사용하여 마지막 폴링 이후 변경된 데이터만 가져올 수도 있다. JDBC Connector는 내부적으로 Kafka Connect API를 사용하고 있으며, 다음과 같은 몇 가지 편리한 기능을 제공한다.

- 몇 가지 설정만으로 작동할 수 있으며, 추가적으로 코드를 구현할 필요가 없다.
- DB 스키마
    - 소스 DB에서 스키마는 보존된다.
    - Kafka에서 DB로 데이터를 스트리밍할 때 Connector는 해당 DB에서 DDL을 실행하여 스키마를 객체를 생성할 수 있다.
- Kafka Connect는 워커 수를 늘려 처리량을 향상시킬 수 있다.
- DB와 상호작용할 때 dialects 사용

### 3.2 JDBC Connector의 제약사항

한편으로 몇 가지 제약 사항도 있다.

- Connector는 JDBC를 통해 소스 DB에 쿼리를 실행하며, `poll.interval.ms` 간격으로 주기적으로 실행된다. 따라서 DB 하드웨어 성능 및 스키마 설계(인덱싱 등)에 따라 Connector를 수평으로 확장하기 어려울 수 있다.
- 최신 데이터만 스트리밍하고 싶다면, 업데이트 정보를 식별할 수 있는 컬럼(숫자 또는 타임스탬프 컬럼)이 존재해야 한다. 소스 DB에 컬럼을 추가할 수 있는 권한이 없다면,(다른 팀이 DB를 소유하고 있는 경우 등) 최신 데이터만 스트리밍하는 것은 어려울 수 있다.
- JDBC Connector는 쿼리로 동작하기 때문에 DB에서 삭제된 데이터는 가져올 수 없다.

### 3.3 CDC

JDBC Connector는 RDBMS와 Kafka를 통합할 수 있는 쉽고 빠른 방법이다. 그러나 이벤트 스트리밍 측면에서 JDBC Connector의 쿼리 기반 동작 방식은 한계가 있기 때문에 Kafka와 완전한 통합을 원한다면 로그 기반 CDC(Change-Data-Capture)를 사용하는 것이 좋다. CDC는 RDBMS의 모든 이벤트가 기록되는 Binary Log를 활용하여 DB에서 발생하는 이벤트를 추출한다. CDC는 JDBC Connector와 달리 쿼리 기반으로 동작하지 않기 때문에 DB 성능에 영향을 덜 받고, 대기 시간이 짧아서 Real-Time 스트리밍이 가능하다. CDC의 유일한 단점은 JDBC Connector보다 설치 및 실행이 복잡하다는 것이다. CDC Publisher의 경우 순서를 보장하기 위해 기본적으로 모든 이벤트를 토픽의 동일한 파티션으로 전달한다.

## 4. Kafka JDBC Connector 실습

### 4.1 Mysql서버

![Untitled 3](https://user-images.githubusercontent.com/56438131/234894703-997ad6f4-9d62-46c6-bb51-c3c1eea50fa6.png)

기본적으로 Mysql에서 타임스탬프(생성, 수정)단위로 불러올것이다.

EC2에 Mysql 8.0버전으로 설치하여 진행하였다.

- **EC2 Inst “KAFKA-RDB” Mysql Setting**

```java
apt-get update
# 업데이트

apt-get install mysql-server
# mysql server 다운로드

mysql --version
#버전정보 확인

sudo mysql -u root -p
#접속

use mysql
alter user "root"@"localhost" identified with mysql_native_password by "암호";
FLUSH PRIVILEGES;
# 비밀번호 설정

```

- **EC2 Inst “KAFKA-RDB” Setting (외부접속허용을 위한)**

```java
vi ./etc/mysql/mysql.conf.d
------------------------------
#**mysql.conf.d**

bind-address = 0.0.0.0
------------------------------

sudo service mysql restart
# 재시작하기

```

- **DB, Table 생성, 데이터삽입 및 User생성, 권한부여**

```java
mysql> create database kafka;
Query OK, 1 row affected (0.01 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| kafka              |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

mysql> use kafka;
Database changed
mysql> show tabels;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'tabels' at line 1
mysql> show tables;
Empty set (0.00 sec)

mysql> create user 'kafka'@'%' identified by 'kafka';
Query OK, 0 rows affected (0.02 sec)

mysql> GRANT ALL PRIVILEGES ON `kafka`.* TO 'kafka'@'%';
Query OK, 0 rows affected (0.00 sec)

mysql> CREATE TABLE test (c1 INT, c2 VARCHAR(255), create_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP, update_ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP);
Query OK, 0 rows affected (0.02 sec)

mysql> show  tables;
+-----------------+
| Tables_in_kafka |
+-----------------+
| test            |
+-----------------+
1 row in set (0.00 sec)

mysql> show create table test;
+-------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                                                                                                                                                                                                   |
+-------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| test  | CREATE TABLE `test` (
  `c1` int DEFAULT NULL,
  `c2` varchar(255) DEFAULT NULL,
  `create_ts` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `update_ts` timestamp NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
+-------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

mysql>  INSERT INTO test (c1, c2) VALUES (1, 'first');
Query OK, 1 row affected (0.01 sec)

mysql> select * from test;
+------+-------+---------------------+---------------------+
| c1   | c2    | create_ts           | update_ts           |
+------+-------+---------------------+---------------------+
|    1 | first | 2023-04-27 12:37:18 | 2023-04-27 12:37:18 |
+------+-------+---------------------+---------------------+
1 row in set (0.00 sec)
```

### 4.2 Broker 및 JDBC Connector 설정

각 브로커는 전에 포스팅하였던 브로커3대로 동일하기에 참고하면 된다. 

> [KAFKA 설치 및 실행 ](https://jongwon0280.github.io/dataengineering/KAFKA-%EC%84%A4%EC%B9%98-%EB%B0%8F-%EC%8B%A4%ED%96%89/)

### 4.2.1 Kafka JDBC Connector 설치

```java
sudo apt install unzip \
	&& wget https://d1i4a15mxbxib1.cloudfront.net/api/plugins/confluentinc/kafka-connect-jdbc/versions/10.6.3/confluentinc-kafka-connect-jdbc-10.6.3.zip \
	&& unzip confluentinc-kafka-connect-jdbc-10.6.3.zip \
	&& cp -r confluentinc-kafka-connect-jdbc-10.6.3/lib/* $KAFKA_HOME/libs/
# 설치 후 $KAFKA_HOME/libs 에 복사 붙여넣기한다.
```

### 4.2.2 JDBC Connector 설정

`$KAFKA_HOME/config/connect-standalone.properties`를 다음과 같이 수정한다.

```java
key.converter.schemas.enable=true
value.converter.schemas.enable=true

offset.storage.file.filename=/tmp/connect.offsets
# Flush much faster than normal, which is useful for testing/debugging
offset.flush.interval.ms=10000

# Set to a list of filesystem paths separated by commas (,) to enable class loading isolation for plugins
# (connectors, converters, transformations). The list should consist of top level directories that include
# any combination of:
# a) directories immediately containing jars with plugins and their dependencies
# b) uber-jars with plugins and their dependencies
# c) directories immediately containing the package directory structure of classes of plugins and their dependencies
# Note: symlinks will be followed to discover dependencies or plugins.
# Examples:
# plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins,/opt/connectors,
**plugin.path=$KAFKA_HOME/lib**
```

다운받아놓은 lib로 플러그인의 경로를 지정해주는 것이다.

`$KAFKA_HOME/config/connect-jdbc-source.properties` 설정 파일을 생성하고, 다음과 같이 수정한다. `connection.url`에서 `$DB_HOST`는 EC2 인스턴스의 **public ip주소**이다.

```java
name=mysql-jdbc-source
connector.class=io.confluent.connect.jdbc.JdbcSourceConnector
tasks.max=1
connection.url=jdbc:mysql://10.144.170.6:3306/kafka?user=kafka&password=kafka
table.whitelist=test
mode=timestamp
timestamp.column.name=update_ts
topic.prefix=mysql-
validate.non.null=false
```

### 4.2.3 MySQL JDBC Driver 설치

```java
wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.28/mysql-connector-java-8.0.28.jar \
	&& cp mysql-connector-java-8.0.28.jar $KAFKA_HOME/libs/
```

### 4.3 Kafka Connect 실행시키기

일단 원하는 결과로써는 DB에 생성한 table이 정의된것이 확인되고, insert한 내역이 올라오기를 기대한다. 실행코드는 다음과 같다.

```java
$KAFKA_HOME/bin/connect-standalone.sh $KAFKA_HOME/config/connect-standalone.properties $KAFKA_HOME/config/connect-jdbc-source.properties
```

connector실행시 자동으로 mysql-’테이블명’으로 토픽을 생성한다. 이를 확인하기 위해서 —describe로 metadata를 확인해준다.

```java
./bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic mysql-test --describe
```

실행결과로 mysql-test로 토픽이 생성된것을 확인할 수 있다.

**Topic: mysql-test       TopicId: 9NPJzOwITMSXWpW-FItM7g PartitionCount: 3       ReplicationFactor: 1  Configs: segment.bytes=1073741824
Topic: mysql-test       Partition: 0    Leader: 1       Replicas: 1     Isr: 1
Topic: mysql-test       Partition: 1    Leader: 2       Replicas: 2     Isr: 2
Topic: mysql-test       Partition: 2    Leader: 3       Replicas: 3     Isr: 3**

Kafka Connector는 다음과 같이 kafka db에 test테이블의 스키마 정보와 insert된 데이터를 출력하고있다.

```java
# Connector is Running
{"schema":{"type":"struct","fields":[{"type":"int32","optional":true,"field":"c1"},
{"type":"string","optional":true,"field":"c2"},{"type":"int64","optional":true,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"create_ts"},
{"type":"int64","optional":true,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"update_ts"}],"optional":false,"name":"test"},
"payload":{"c1":1,"c2":"first","create_ts":1682599038000,"update_ts":1682599038000}}
```

### 4.3.1 데이터 삽입 및 수정

```java
INSERT INTO test (c1, c2) VALUES (2, 'second');
UPDATE test SET update_ts = NOW() WHERE c1 = 1;
UPDATE test SET update_ts = (NOW() - INTERVAL 1 DAY) WHERE c1 = 1;

```

```java
# KAFKA Connector Running
"payload":{"c1":2,"c2":"second","create_ts":1682602617000,"update_ts":1682602617000}}
"payload":{"c1":1,"c2":"first","create_ts":1682599038000,"update_ts":1682602801000}}
```

앞에 두 쿼리는 제대로 반영되는 것을 확인할 수 있지만, 마지막 update구문 같은경우, 타임스탬프상 과거데이터가 되어버리므로, connector가 가져오지 못하는것을 알 수 있다.