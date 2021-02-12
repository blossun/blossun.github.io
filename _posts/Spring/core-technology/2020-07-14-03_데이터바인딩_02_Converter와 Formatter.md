---
title: "[데이터 바인딩 추상화]_02 Converter와 Formatter"
toc: true
toc_sticky: true

categories:
  - SPRING/core-technology
tags:
  - SPRING
  - LECTURE
---

# 데이터 바인딩 추상화: Converter와 Formatter

[Converter](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/convert/converter/Converter.html)

	● S 타입을 T 타입으로 변환할 수 있는 매우 일반적인 변환기.
	● 상태 정보 없음 == Stateless == 쓰레드세이프
	● [ConverterRegistry](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/convert/converter/ConverterRegistry.html)에 등록해서 사용

```java
public class StringToEventConverter implements Converter<String, Event> {
  @Override
  public Event convert(String source) {
    Event event = new Event();
    event.setId(Integer.parseInt(source));
    return event;
  }
}
```

[Formatter](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/format/Formatter.html)

	● PropertyEditor 대체제
	● Object와 String 간의 변환을 담당한다.
	● 문자열을 Locale에 따라 다국화하는 기능도 제공한다. (optional)
	● [FormatterRegistry](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/format/FormatterRegistry.html)에 등록해서 사용

```java
public class EventFormatter implements Formatter<Event> {
  @Override
  public Event parse(String text, Locale locale) throws ParseException {
    Event event = new Event();
    int id = Integer.parseInt(text);
    event.setId(id);
    return event;
  }
  @Override
  public String print(Event object, Locale locale) {
    return object.getId().toString();
  }
}
```

[ConversionService](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/convert/ConversionService.html)

	● 실제 변환 작업은 이 인터페이스를 통해서 쓰레드-세이프하게 사용할 수 있음.
	● **스프링 MVC**, 빈 (value) 설정, SpEL에서 사용한다.
	● DefaultFormattingConversionService
			● FormatterRegistry
			● ConversionService
			● 여러 기본 컴버터와 포매터 등록 해 줌.



<img src="https://i.imgur.com/5dbVAu6.png" alt="converter formatter" style="zoom:50%;" />

스프링 부트  
```
● 웹 애플리케이션인 경우에 DefaultFormattingConversionSerivce를 상속하여 만든 **WebConversionService**를 빈으로 등록해 준다.  
​● Formatter와 Converter 빈을 찾아 자동으로 등록해 준다
```

---

# Converter

Property가 가지고 있던 단점때문에 Converter가 생겼다.

Object와 String 간의 변환만할 수 있었던 Property와 달리 서로 다른 타입간의 변환이 가능하다.

Property가 가지고있던 상태정보를 Converter는 가지고 있지 않다.

## 1. 컨버터 구현

source와 target을 매개변수로 Converter를 구현하면 된다. 

`converter()`라는 메서드만 구현해주면 된다.

앞서 만든 PropertyEditor와 같은 기능을 한다. 단지, 상태정보가 없기 때문에 얼마든지 빈으로 등록해서 사용해도 상관없다.

```java
public class EventConverter {

    //String -> Event 타입변환
    @Component
    public static class StringToEventConverter implements Converter<String, Event> {
        @Override
        public Event convert(String source) {
            return new Event(Integer.parseInt(source));
        }
    }

    //Evnet -> String 타입변환
    @Component
    public static class EventToStringConverter implements Converter<Event, String> {
        @Override
        public String convert(Event source) {
            return source.getId().toString();
        }
    }
}
```



## 2. 빈등록

ConverterRegistry에 등록해서 사용하면 된다. 우리가 ConverterRegistry를 직접 쓸 일은 거의 없다. 

* 스프링부트없이 스프링 MVC를 쓴다면, WebConfig를 쓴다고 가정
  Web용 Configuration을 만든 코드에서 `addFormatters()`메서드를 오버라이딩해서 `registry.addConverter(등록할 컨버터)`로 컨버터를 레지스터리에 등록해서 사용하면 된다.

  ```java
  @Configuration
  public class WebConfig implements WebMvcConfigurer {

      @Override
      public void addFormatters(FormatterRegistry registry) {
          registry.addConverter(new EventConverter.EventToStringConverter()); //<--
      }
  }
  ```



EventController에서 PropertyEditor를 사용하던 코드 삭제 후, 테스트 코드 실행

* EventController.java

  ```java
  @RestController
  public class EventController {
  
      @GetMapping("/event/{event}")
      public String getEvent(@PathVariable Event event) {
          System.out.println(event);
          return event.getId().toString();
      }
  }
  ```

* EventControllerTest.java

  ```java
  @RunWith(SpringRunner.class)
  @WebMvcTest
  public class EventControllerTest {
  
      @Autowired
      MockMvc mockMvc;
  
      @Test
      public void getTest() throws Exception {
          mockMvc.perform(MockMvcRequestBuilders.get("/event/1"))
                  .andExpect(status().isOk())
                  .andExpect(content().string("1"));
      }
  }
  ```

테스트 통과

```
Event{id=1, title='null'}
```

⇒ 컨트롤러에서 `/event/1`로 넘어온 url에서 `1`값을 String → Event 타입으로 형변환 하여 받을 수 있는 것이다.



Integer 타입같은 경우, 기본적으로 등록이 되어있는 컨버터나 포메터들이 자동을 변환을 해준다. 모든 타입을 컨버터를 만들지 않아도 된다.

따라서 스프링에 기본적으로 등록되어있지 않은 컨버터만 만들어서 사용하면 된다.

컨버터는 재너럴하다. 웹의 경우 거의 대부분의 경우 사용자 입력값이 문자열로 들어오고, 객체들을 문자로 내보내는 경우가 많다. 그런 문자들을 MessageSource(i18n)를 사용해서 다국화를 지원한다. (스프링에서 이 기능에 특화된 Formatter 인터페이스를 제공)



# Formatter (추천)

스프링이 조금 더 웹에 특화되어있는 인터페이스를 만들어서 제공한다. 

※ PropertyEditor와 다른점은 `Locale` 기반으로 문자열을 제공해줄 수 있다.

## 1. 컨버터 구현

처리할 타입을 매개변수로 Formatter를 구현하면 된다. 

`parse()`, `print()`라는 메서드만 구현해주면 된다.

* parse() : 문자열 → 객체

* print() : 객자 → 문자열



1. Thread Safe하기 때문에 빈으로 등록해서 사용한다.

"빈으로 등록할 수 있다"는 의미는 다른 빈을 주입받을 수 있다는 의미도 된다.

예시 ) `MessageSource`를 주입받아서 Locale정보를 써서 메시지를 만들 수 있다.

```java
@Component //빈으로 만들어도 ok
public class EventFormatter implements Formatter<Event> {

    @Autowired //빈으로 등록했기때문에 빈을 주입받는 것도 가능
    MessageSource messageSource;

    @Override
    public Event parse(String text, Locale locale) throws ParseException {
        return new Event(Integer.parseInt(text));
    }

    @Override
    public String print(Event object, Locale locale) {
        //객체에서 필요한 메시지코드 추출
        messageSource.getMessage("title", locale);
        return object.getId().toString();
    }
}
```



2. 빈으로 등록하지 않고 사용하는 방식

WebConfig(WebMvcConfigurer 구현체)에서 `registry.addFormatter()`로 포메터를 등록해주면 된다.

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addFormatter(new EventFormatter());
    }
}
```

( 참고 : 이전에 Converter를 빈으로 등록한 것 삭제 후, Formatter만 빈으로 등록해서 실행)

테스트 성공

```
Event{id=1, title='null'}
```



# ConversionService

이런 타입을 변환하는 작업은 `데이터 바인더` 를 통해서 PropertyEditor를 사용했던 것 대신에 `ConversionService`에 등록되어있는 Converter와 Formatter를 사용한 것이다.

* 실제 변환 작업은 이 인터페이스를 통해서 쓰레드-세이프하게 사용할 수 있음.
* **스프링 MVC**, 빈 (value) 설정, SpEL에서 사용한다.
* DefaultFormattingConversionService
  * FormatterRegistry
  * ConversionService
  * 여러 기본 컴버터와 포매터 등록 해 줌.



## DefaultFormattingConversionService

스프링이 제공해주는 여러 ConversionService 구현체들 중 하나로 ConversionService 빈으로 자주 사용이 된다. 

`FormatterRegistry` 와 `ConversionService` 두가지 인터페이스를 모두 구현하였다.

<img src="/assets/images/SPRING/core-technology/5dbVAu6.png" alt="converter formatter" style="zoom:50%;" />

Converter는 ConverterRegistry에 등록해야하고, Formatter는 FormatterRegistry에 등록해야한다. FormatterRegistry가 ConverterRigstry를 상속받고 있으므로 FormatterRegistry에 Converter를 등록해서 사용할 수도 있다.

```java
@Override
public void addFormatters(FormatterRegistry registry) {
  registry.addConverter(new EventConverter.EventToStringConverter());
}
```



## 스프링 부트에서 ConversionService 사용

```java
@Component
public class AppRunner implements ApplicationRunner {

    @Autowired
    ConversionService conversionService;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        //ConversionService을 빈으로 주입받을 수 있는지 확인
        System.out.println(conversionService.getClass().toString());
    }
}
```

실행 결과 확인

```
class org.springframework.boot.autoconfigure.web.format.WebConversionService
```

⇒  `WebConversionService`가 빈으로 등록되었다.



# 스프링부트가 지원하는 기능

## 1. 웹 애플리케이션인 경우에 DefaultFormattingConversionSerivce를 상속하여 만든 **WebConversionService**를 빈으로 등록해 준다. 

* **WebConversionService**는 **스프링부트**가 제공해주는 서비스이다.
* 더 많은 기능을 제공해준다.

ConversionService가 가지고 있는 Converter와 Formatter을 이용해서 타입변환을 할 수 있다.

`conversionService.canConvert()`등의 메서드로

프로그래밍으로 직접 이 인터페이스를 써서 컨버팅 해야한다면 conversionService를 가지고 컨버팅할 수 있다. 하지만 이런 경우도 거의 없고, ConversionService를 빈으로 주입받아서 사용하는 경우도 거의 없다.



※ **WebConversionService** 소스코드 확인

command + shift + F → [Scope] → WebConversionService 검색

Formatter들이 추가되어있다.

```java
//WebConversionService.java
public WebConversionService(DateTimeFormatters dateTimeFormatters) {
  super(false);
  if (dateTimeFormatters.isCustomized()) {
    addFormatters(dateTimeFormatters);
  }
  else {
    addDefaultFormatters(this);
  }
}
```

JSR_354_PRESENT ⇒ 돈 관련된 API가 ClassPath에 있으면 돈 관련 Converter를 추가

JODA_TIME_PRESENT ⇒ JODA_TIME 관련된 라이브러리가 ClassPath에 있으면 JODA_TIME 관련 Converter를 추가



## 2. Formatter와 Converter 빈을 찾아 자동으로 등록해 준다

따라서 WebMvcConfigurer를 따로 구현해서 Formatter를 구현해줄 필요가 없다.

※ [실습 1] Converter를 구현한 클래스만 빈으로 등록해준다.

기존에 WebConfig 파일 삭제, Formatter는 빈으로 등록하지 않음

Converter는 상태정보를 저장하지 않기 때문에 빈으로 등록해도 안전한다.

```java
// 주의 EventConverter를 빈으로 등록하는 것이 아님
public class EventConverter {

    @Component //빈으로 등록
    public static class StringToEventConverter implements Converter<String, Event> {
        @Override
        public Event convert(String source) {
            return new Event(Integer.parseInt(source));
        }
    }

    @Component
    public static class EventToStringConverter implements Converter<Event, String> {
        @Override
        public String convert(Event source) {
            return source.getId().toString();
        }
    }
}
```

애플리케이션 실행 후, 브라우저에서 URL 접속

![url접속확인](/assets/images/SPRING/core-technology/kNhwPiM.png)

※ [실습 2] Formatter만 빈으로 등록해준다.

Converter는 빈으로 등록하지 않음

```java
@Component //빈으로 등록
public class EventFormatter implements Formatter<Event> {

    @Override
    public Event parse(String text, Locale locale) throws ParseException {
        return new Event(Integer.parseInt(text));
    }

    @Override
    public String print(Event object, Locale locale) {
        return object.getId().toString();
    }
}
```

애플리케이션 실행 후, 브라우저에서 URL 접속

```
2020-07-14 14:41:50.428  INFO 71308 --- [           main] d.s.d.Demospring51Application            : Started Demospring51Application in 2.052 seconds (JVM running for 2.517)
class org.springframework.boot.autoconfigure.web.format.WebConversionService
...
Event{id=13, title='null'}
```

![포메터등록](/assets/images/SPRING/core-technology/PCx00Fr.png)

### 테스트코드 -  `@WebMvcTest()` 슬라이싱 테스트

* (스프링부트 기능)

* 계층형 테스트로 웹과 관련된 빈만 등록해준다.

* 컨트롤러가 주로 등록이 된다.

* Converter나 Formatter가 제대로 등록되지 않으면 테스트가 깨질 수 있다.

* 이런 경우 `@WebMvcTest()` 안에 테스트에 필요한 빈으로 등록을 해줄 수 있다.

* 그냥 클래스를 빈으로 등록해주지 않는다. ComponentScan이 가능한 클래스여야 한다.

  ex) @Component, @Controller

* 빈으로 등록되어야 컨버터 기능이 동작한다.

* 테스트에 필요한 빈들을 명시적으로 표시해주는 것도 좋다고 봄



※ [실습] 1 - Formatter 테스트

```java
@RunWith(SpringRunner.class)
@WebMvcTest({EventFormatter.class, EventController.class}) //<-- 추가
public class EventControllerTest {
  ...
}
```

EventFormatter를 빈으로 등록해서 테스트하기 때문에 테스트에 통과한다.



※ [실습] 2 - Converter 테스트

```java
@RunWith(SpringRunner.class)
@WebMvcTest({EventConverter.StringToEventConverter.class, EventController.class})
public class EventControllerTest {
  ...
}
```



### 등록되어있는 컨버터를 보는 방법

```java
@Component
public class AppRunner implements ApplicationRunner {

    @Autowired
    ConversionService conversionService;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        //등록되어있는 컨버터 출력
        System.out.println(conversionService);
    }
}
```

```
	@org.springframework.format.annotation.DateTimeFormat java.lang.Long -> java.lang.String: org.springframework.format.datetime.DateTimeFormatAnnotationFormatterFactory@21ae6e73,@org.springframework.format.annotation.NumberFormat java.lang.Long -> java.lang.String: org.springframework.format.number.NumberFormatAnnotationFormatterFactory@d4602a
	
	java.lang.Boolean -> java.lang.String : org.springframework.core.convert.support.ObjectToStringConverter@3dedb4a6
	java.lang.Character -> java.lang.Number : org.springframework.core.convert.support.CharacterToNumberFactory@67fe380b
	java.lang.Character -> java.lang.String : org.springframework.core.convert.support.ObjectToStringConverter@55f45b92
	java.lang.Enum -> java.lang.Integer : org.springframework.core.convert.support.EnumToIntegerConverter@49d98dc5
	java.lang.String -> java.util.TimeZone : org.springframework.core.convert.support.StringToTimeZoneConverter@74a195a4
	...
```



---

Formatter를 사용하는 방식을 추천한다.

이런 데이터 바인딩을 만들 때, 보통 웹과 관련해서 만들기 때문이다.

아니면 컨버터를 써도 상관은 없다. String → XXX 과 같은 형변환

그런데JPA와 관련해서 쓴다면 Entity들은 이미 컨버터가 들어있다.

