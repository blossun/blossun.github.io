---
title: "로컬에서 도커이미지 만들어서 실행해보기"
excerpt: "Dockerfile, Kafka Application"
toc: true
toc_sticky: true

categories:
  - INFRA/kubernetes
tags:
  - KUBERNETES
  - LECTURE
---

# Kafka 실습 애플리케이션을 로컬에서 도커이미지 만들어서 실행해보기

# 1. Docker 설치

먼저 [Docker Desktop for Mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac) 링크로 이동해서 Docker.dmg 파일을 다운받고, 설치

설치 완료 후, Docker를 실행하면 상단메뉴바에 도커 아이콘으로 running 상태 확인

```bash
# 다음과 같이 뜨면 ok
$ docker container ls               
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```



# 2. 도커 이미지 생성을 위한 Dockerfile 작성

```bash
# Start with a base image containing Java runtime
# FROM reg.navercorp.com/base/centos7/jdk:11.0.6.x64
FROM docker.io/centos:7

RUN yum install -y java-11

# Add Author info
LABEL maintainer="blossunn@gmail.com"

# Add a volume to /tmp
VOLUME /tmp

# Make port 8080 available to the world outside this container
EXPOSE 8080

# The application's jar file
ARG JAR_FILE=build/libs/kafkatest-0.0.1-SNAPSHOT.jar

# Add the application's jar to the container
ADD ${JAR_FILE} kafka-springboot.jar

# Run the jar file™
ENTRYPOINT ["java","-jar","/kafka-springboot.jar"]
```

- FROM [docker.io/centos:7](http://docker.io/centos:7) [docker.io](http://docker.io) 라는 Docker Hub에서 centos:7 버전 Base Image를 가져온다.

- RUN yum install -y java-11

  여기에 애플리케이션을 구동시키기 위해 필요한 java 11버전을 설치

- LABEL [maintainer="blossunn@gmail.com](mailto:maintainer="blossunn@gmail.com)"

  Label은 말 그대로 라벨이다. 필요한 정보를 표시

  maintainer를 추가해 이 이미지를 관리하는 사람이 누구인지 명시

- VOLUME /tmp

  이제 VOLUMN이라는 디렉토리를 지정 해 준다. 이 디렉토리 /tmp아래에 이 컨테이너가 필요한 여러가지 데이터를 저장

- EXPOSE 8080

  이 어플리케이션은 도커 컨테이너 내부에서는 8080의 포트를 가지고 돌 것이다. 따라서 이 포트를 외부로 노출시킨다.

- ARG JAR_FILE=build/libs/kafkatest-0.0.1-SNAPSHOT.jar

  ARG JAR_FILE을 이용해 어떤 어플리케이션을 실행시켜야 하는지, 즉 어플리케이션의 실행파일을 연결

  - `./gradlew build` 로 빌드파일을 생성하면 `build/libs` 경로에 `jar` 파일이 생긴다.

- ADD ${JAR_FILE} kafka-springboot.jar

  이 JAR_FILE에 이름을 붙여준다. 여기서 이름은 `kafka-springboot.jar` 가 된다.

- ENTRYPOINT ["java","-jar","/kafka-springboot.jar"]

  - ENTRYPOINT가 바로 자바를 실행
  - 어플리케이션을 실행시키기 위한 명령어
  - 이 부분에 들어가는 명령어들은 실제 명령어를 스페이스로 나눠놓은 것과 같다.


# 3. 도커 이미지 생성

```bash
# -t {도커 이미지명}
$ docker build -t kafka-springboot .
# 생성된 도커 이미지 확인
$ docker images      
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
kafka-springboot    latest              74adc54ae999        3 minutes ago       950MB
```

![https://s3-us-west-2.amazonaws.com/secure.notion-static.com/218f87a2-1e28-4063-a841-e6975b06be2b/Untitled.png](/assets/images/INFRA/kubernetes/image-20210411021320688.png)

![image-20210411021333367](/assets/images/INFRA/kubernetes/image-20210411021333367.png)



# 4. 생성한 도커 이미지 실행

- (내부) 8080으로 돌아가는 `kafka-springboot` 서비스를 외부에 `5000` 포트로 노출시키겠다.

  → 외부에서 5000번 포트로 서비스에 접근하면 된다.

```bash
$ docker run -p 5000:8080 kafka-springboot

  .   ____          _            __ _ _
 /\\\\ / ___'_ __ _ _(_)_ __  __ _ \\ \\ \\ \\
( ( )\\___ | '_ | '_| | '_ \\/ _` | \\ \\ \\ \\
 \\\\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.4.4)

2021-04-10 13:45:25.909  INFO 1 --- [           main] d.solar.kafkatest.KafkatestApplication   : Starting KafkatestApplication using Java 11.0.10 on 7a9821b1208f with PID 1 (/kafka-springboot.jar started by root in /)
```



# 5. 서비스 동작 확인

1. `localhost: 5000/kafka` 로 POST 요청

   - body에 message를 담아서 보낸다.

   ![image-20210411021357251](/assets/images/INFRA/kubernetes/image-20210411021357251.png)

2. 컨트롤러가 요청을 받아서 Produce 하는 것 확인

   ![image-20210411021419867](/assets/images/INFRA/kubernetes/image-20210411021419867.png)

3. 컨슈머가 메시지를 받아오는 것 확인

   ⇒ response 200을 보내고, mongoDB에 message를 저장한다.

   ![image-20210411021437970](/assets/images/INFRA/kubernetes/image-20210411021437970.png)

4. MongoDB에 message가 저장된 것 확인

   ![image-20210411021459227](/assets/images/INFRA/kubernetes/image-20210411021459227.png)

5. 같은 토픽 (testsunyoung)을 구독 중인 컨슈머에도 해당 message가 출력되는 것 확인할 수 있다.

   ![image-20210411021534679](/assets/images/INFRA/kubernetes/image-20210411021534679.png)

   

------

# REF

[스프링 부트 도커에 올리기(Dockerizing Spring Boot App)](https://imasoftwareengineer.tistory.com/40)
