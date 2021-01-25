---
title: "[Issue Tracker] 에러 해결 정리"
excerpt: "프로젝트를 진행하면서 발생한 에러와 해결방법"
toc: true
toc_sticky: true

categories:
  - PROJECT/issueTracker
tags:
  - PROJECT
  - Error
---

## error - 1

* delete 실행 시 발생한 오류

### 오류메시지

```
No EntityManager with actual transaction available for current thread - cannot reliably process 'remove' call
```

### 해결책

delete를 수행하는 메소드에 `@Transactional`을 붙여줌

[참고 블로그](https://yoonho-devlog.tistory.com/61)

---

## error - 2

* Query 실행 에러

### 오류메시지

```
Caused by: java.lang.IllegalArgumentException: Validation failed for query for method public abstract void kr.codesquad.issuetracker09.domain.IssueRepository.updateMilestoneById(java.lang.Long,java.lang.Long)!
```

### 해결책

`@Query` 어노테이션을 사용해서 네이티브 쿼리를 수행하려면 `nativeQuery`속성을 `true`로 설정해줘야 한다.

기존코드

```java
@Repository
public interface IssueRepository extends JpaRepository<Issue, Long> {

    @Query(value = "UPDATE issue SET milestone_id = :milestone_id WHERE id = :issue_id")
    void updateMilestoneById(@Param("milestone_id") Long milestoneId, @Param("issue_id") Long issueId);
}
```

수정코드

```java
@Repository
public interface IssueRepository extends JpaRepository<Issue, Long> {

    @Query(value = "UPDATE issue SET milestone_id = :milestone_id WHERE id = :issue_id", nativeQuery = true)
    void updateMilestoneById(@Param("milestone_id") Long milestoneId, @Param("issue_id") Long issueId);
}
```

---

## error - 3

![image-20200616012707181](/assets/images/PROJECT/issue_tracker/007S8ZIlgy1gftfucubdmj31gw0aewqn.jpg)


1. 환경변수명이 불일치 했었음

   실제 환경변수명은 `$GITHUB_CLIENT_ID` , `$GITHUB_CLIENT_SECRET`이었는데, 코드 상에서 호출하는 환경변수 명은 

    `$CLIENT_ID` , `$CLIENT_SECRET`이었음

   ⇒ 실제 환경 변수 명을 `$CLIENT_ID` , `$CLIENT_SECRET` 으로 수정함

2. Github으로 부터 사용자의 정보를 가져오는 부분에서 코드 리팩토링을 하였는데, 이 과정에서 잘 못 매핑시키고 있었음.

   ⇒ 리팩토링 하기 이전 코드로 원복

   ```java
   public User getSocialUser(String code) throws JsonProcessingException {
     GithubToken accessToken = getAccessToken(code);
     ResponseEntity<String> response = new RestTemplate().exchange(USER_DATA_API, HttpMethod.GET,
                                                                   getHttpEntityWithAuthorization(accessToken), String.class);
     JsonNode jsonNode = objectMapper.readTree(response.getBody());
     return User.builder()
       .socialId(jsonNode.required("id").asLong())
       .name(jsonNode.required("name").asText())
       .email(jsonNode.required("email").asText())
       .build();
   }
   ```

   

   원인. 
   바로 User객체로 매핑시키려고 리팩토링을 진행하였다. 하지만 실제 github으로 부터 받아오는 `id`값을 User 클래스의 `socialId`에 매핑 시켜야하는데, `id`에 바로 매핑시키고 있었음

   ```java
   public User getSocialUser(String code) {
       GithubToken accessToken = getAccessToken(code);
       ResponseEntity<User> response = new RestTemplate().exchange(USER_DATA_API, HttpMethod.GET,
               getHttpEntityWithAuthorization(accessToken), User.class);
   
       if(response.getStatusCode() == HttpStatus.OK) {
           return response.getBody();
       }
   
       return null;
   }
   ```

---

## error - 4 : **★ JWT Long 타입 변환 에러**

![스크린샷 2020-06-16 오전 2.37.32](/assets/images/PROJECT/issue_tracker/007S8ZIlgy1gfti3uhte5j31ct0u0135.jpg)



	JWT 토큰을 넣고 요청을 보내면 다음과 같이 500 에러가 발생했다.

![스크린샷 2020-06-16 오전 2.46.10](/assets/images/PROJECT/issue_tracker/0081Kckwgy1glb01shwcyj319t0u07at.jpg)


> **참고**
> jwt integer to long
> [Class cast exception in JWT - Stack Overflow](https://stackoverflow.com/questions/49964955/class-cast-exception-in-jwt)



타입 변환 오류 문제

- id와 socialId 값을 Long 타입으로 변환하는 과정에서 에러 발생
  (Long) 으로 형변환하지 않고, get 메소드의 2번째 파라미터 값으로 requiredType을 지정해준다.

```java
//기존
Long id = (Long) claims.get("id");
Long socialId = (Long) claims.get("socialId");

//수정
Long id = claims.get("id", Long.class);
Long socialId = claims.get("socialId", Long.class);
```



정상적으로 동작되는 것 확인

![스크린샷 2020-06-16 오전 2.46.10](/assets/images/PROJECT/issue_tracker/007S8ZIlgy1gfti5x3806j30zn0u0jyy.jpg)



```
hotfix: jwt 파싱 에러 수정

- id와 socialId 값을 Long 타입으로 변환하는 과정에서 에러 발생
 (Long) 으로 형변환하지 않고, get 메소드의 2번째 파라미터 값으로 requiredType을 지정해준다.
- Github으로 부터 가져온 id 값이 User의 id가 아닌 socialId로 매핑시키기 위해 다시 JsonNode를 이용하도록 수정
```


해결
잘 받아와진다.

![스크린샷 2020-06-16 오후 1.51.42](/assets/images/PROJECT/issue_tracker/007S8ZIlgy1gfu1eia0s1j30w70u07e9.jpg)

