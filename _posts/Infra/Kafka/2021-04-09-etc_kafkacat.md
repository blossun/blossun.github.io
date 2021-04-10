---
title: "Kafkacat을 이용해 kafka 동작 확인"
excerpt: "kafkacat으로 produce & consume"
toc: true
toc_sticky: true

categories:
  - INFRA/kafka
tags:
  - KAFKA
  - LECTURE
---

## kafkacat 설치
```
brew install kafkacat
```

## 옵션
- -b : 브로커 접속 정보 {domain}:{port}
- -t : 구독할 토픽명
- -K : Message Keys 구분자 
- -P : Produce data
- -C : Consume data
    - -G : 그룹ID 지정 필수

## Kafka Producer
```
kafkacat -b localhost:9092 -t testsunyoung -P
```

## Kafka Consumer
```
kafkacat -b localhost:9092 -t testsunyoung -G holar -C 
```

![img](/assets/images/INFRA/kafka/kafkacat.png)

## REF
* [Kafka Usage](https://dev.to/de_maric/learn-how-to-use-kafkacat-the-most-versatile-kafka-cli-client-1kb4)
