---
title: "[스프링 REST 클라이언트]_01_RestTemplate과 WebClient"
toc: true
toc_sticky: true

categories:
  - SPRING/concepts-utilization
tags:
  - SPRING
  - LECTURE
  - RestTemplate
  - WebClient
---

# 스프링 REST 클라이언트 1부: RestTemplate과 WebClient

RestTemplate

* Blocking I/O 기반의 Synchronous API 
* RestTemplateAutoConfiguration
* 프로젝트에 spring-web 모듈이 있다면 RestTemplate**Builder**를 빈으로 등록해 줍니다.
* https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#rest-client-access

WebClient

* Non-Blocking I/O 기반의 Asynchronous API
* WebClientAutoConfiguration
* 프로젝트에 spring-webflux 모듈이 있다면 WebClient.**Builder**를 빈으로 등록해 줍니다.
* https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client

※ 스프링이 제공하는 기능으로 스프링부트는 빈으로 등록해주기만 한다. RestTemplate과 WebClient를 빈으로 등록해주는 것이 아니라 **Builder**를 등록해주는 것. 따라서 빌더를 주입 받아서 필요할 때마다 빌드를 해서 인스턴스를 생성해서 사용해야한다.

* RestTemplate과 WebClient 둘을 동시에 사용해도 된다.

---

## RestTemplate

* Blocking I/O 기반의 Synchronous API 

(예제)

/hello 요청은 5초가 걸리고 /world 요청은 3초가 걸릴 때, /hello 요청과 /world 요청을 순서대로 호출해서 걸리는 시간을 확인

```java
@RestController
public class SampleController {

    @GetMapping("/hello")
    private String hello() throws InterruptedException {
        Thread.sleep(5000l); //5초 슬립
        return "hello";
    }

    @GetMapping("/world")
    private String world() throws InterruptedException {
        Thread.sleep(3000l); //3초 슬립
        return "world";
    }
}
```

```java
@Component
public class RestRunner implements ApplicationRunner {

    @Autowired
    RestTemplateBuilder restTemplateBuilder;


    @Override
    public void run(ApplicationArguments args) throws Exception {
        RestTemplate restTemplate = restTemplateBuilder.build();

        StopWatch stopWatch = new StopWatch();
        stopWatch.start();

        String helloResult = restTemplate.getForObject("http://localhost:8080/hello", String.class); //이 요청이 끝날 때까지 다음 라인으로 넘어가지 않는다. // 5초 슬립이 걸려있다.
        System.out.println(helloResult);

        String worldResult = restTemplate.getForObject("http://localhost:8080/world", String.class); //이 요청이 끝날 때까지 다음 라인으로 넘어가지 않는다. // 3초 슬립이 걸려있다.
        System.out.println(worldResult);

        stopWatch.stop();
        System.out.println(stopWatch.prettyPrint());
    }
}
```

→ 한 라인의 명령이 끝날 때까지 기다리면서 다음라인을 순차적으로 진행한다. 

결과 : 총 8초의 시간이 걸림

```
2021-01-06 11:42:21.907  INFO 13385 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
hello
world
StopWatch '': running time = 8103483462 ns //<-- 총 8초의 시간이 걸림
---------------------------------------------
ns         %     Task name
---------------------------------------------
8103483462  100%  
```



## WebClient

* Non-Blocking I/O 기반의 Asynchronous API

```java
Mono<String> helloMono = webClient.get().uri("http://localhost:8080/hello") //요청
        .retrieve() //결과값을 가져옴
        .bodyToMono(String.class); //Body값을 Mono 타입으로 변경
```

※ **Mono**와 **Flux**에 대한 공부 필요

이건 Streaming API인데, Steam을 Subscribe하기 전까지 흐르지 않는다. 위의 코드는 물이 흐르지 않고, 물을 어딘가에 담아서 둔 것 뿐이다. 

**`subscribe()`를 해야 실제 물이 흐르게 되어있다.** 

실제 요청을 보내는 subscribe() 코드도 Non-Blocking이다. s → {} 코드에서 `{}`안의 코드가 Asynchronous하게 응답이 오면 그 제서야 코드가 실행된다. 응답처리를 기다리지 않음

```java
// 실제 요청을 보내는 코드 - 이 코드도 Non-Blocking이다.
helloMono.subscribe(s -> {
	  //subscribe()는 결과값이 String으로 나온다. 결과값을 출력하고 stropwatch를 잠시 멈추고 출력 후 재시작
    System.out.println(s);

    if (stopWatch.isRunning()) {
        stopWatch.stop();
    }

    System.out.println(stopWatch.prettyPrint());
    stopWatch.start();
});
```



(예제)

/hello 요청은 5초가 걸리고 /world 요청은 3초가 걸릴 때, /hello 요청과 /world 요청을 순서대로 호출해서 걸리는 시간을 확인

```java
@Component
public class RestRunner implements ApplicationRunner {

    @Autowired
    WebClient.Builder builder;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        WebClient webClient = builder.build();

        StopWatch stopWatch = new StopWatch();
        stopWatch.start();

        Mono<String> helloMono = webClient.get().uri("http://localhost:8080/hello")
                .retrieve()
                .bodyToMono(String.class);
        helloMono.subscribe(s -> {
            System.out.println(s);

            if (stopWatch.isRunning()) {
                stopWatch.stop();
            }

            System.out.println(stopWatch.shortSummary());
            stopWatch.start();
        });

        Mono<String> worldMono = webClient.get().uri("http://localhost:8080/world")
                .retrieve()
                .bodyToMono(String.class);
        worldMono.subscribe(s -> {
            System.out.println(s);

            if (stopWatch.isRunning()) {
                stopWatch.stop();
            }

            System.out.println(stopWatch.shortSummary());
            stopWatch.start();
        });
    }
}
```

```
world
StopWatch '': running time = 3344120450 ns // word는 3초 슬립이기 때문에 먼저 처리돼서 출력된다.
hello
StopWatch '': running time = 5314634377 ns // 그 후, 5초 슬립걸린 hello 출력
```





⇒ WebClient를 사용하면, 

다양하게 여러가지 API를 호출할 때, API 호출을 조합할 수 있다.

ex)  A api, B api, C api 를 모두 호출하고, 모든 응답이 다 왔을 경우에 다른 로직을 처리한다.

​		또는 A, B 응답이 오면 그 때 A, B 응답을 가지고 요청을 만들어서 C api로 요청을 보내서 처리한다.

유연하게 API Client를 만들어서 쓸 수 있다. 따라서 WebClient를 추천!! (하지만 공부 필요)

다양한 호출이 없는 경우, RestTemplate을 쓰는 것도 지장이 없으므로 요구사항에 맞게 선택해서 사용하자



---

##### 질문

> 이전에는 비동기로 처리하기위해 @async 어노테이션으로 새로운 함수를 만들고 그 함수 내에서 client로 요청했는데, 이렇게 하는 것과 webclient 를 사용하는 것에는 어떤 차이가 있나요?

@Async와 RestTemplate을 사용하면 별도의 쓰레드 풀로 Blocking I/O를 처리하기 때문에 결과적으로 WebClient를 사용하는 것과 같은 효과를 낼 수 있습니다. 차이는 @Async용 쓰레드풀 설정이 필요하다는 점과 코딩 스타일이 다를 뿐입니다.

