---
title: "[REST API 및 프로젝트 소개]_03_Postman & Restlet"
excerpt: "Event API 설명"
toc: true
toc_sticky: true

categories:
  - SPRING/rest-api
tags:
  - SPRING
  - LECTURE
---

# Events API 사용 예제

1. (토큰 없이) 이벤트 목록 조회

   a. create 안 보임

2. access token 발급 받기 (A 사용자 로그인)

3. (유효한 A 토큰 가지고) 이벤트 목록 조회

   a. create event 보임

4. (유효한 A 토큰 가지고) 이벤트 만들기

5. (토큰 없이) 이벤트 조회

   a. update 링크 안 보임

6. (유효한 A 토큰 가지고) 이벤트 조회

   a. update 링크 보임

7. access token 발급 받기 (B 사용자 로그인)

8. (유효한 B 토큰 가지고) 이벤트 조회

   a. update 안 보임



REST API 테스트 클라이언트 애플리케이션

* 크롬 플러그인
  * Restlet
    ![img](/assets/images/SPRING/rest_api/rQXMvI1rRmbs4i-DM38SPy3-OB_FwqHXfPJjS7wdWmMitdafUiYzmPqxMCHW3o6LHF5oP9ww5JjWWvfxsceQmBkzTA8WtE3XNy-3BYCbuNK3YZPG2i-aUeZOUBr9F2RK6FmY8izX.png)
* 애플리케이션
  * Postman![img](/assets/images/SPRING/rest_api/BZxgoklk6WYjUskD3CAYd4daIXYg3__cku99vhpMvcA6l-4LFvuV9EnMBdepmSet_tBIXPzu8rUmrxwIhcFy1rQuRsDLDkH67yTCLH9A9sSXqGXg31ix3YRQwV5vheG8t3TJM-Fk.png)

---
