---
title: "[데이터 바인딩 추상화]_01 PropertyEditor"
toc: true
toc_sticky: true

categories:
  - SPRING/core-technology
tags:
  - SPRING
  - LECTURE
---

# 데이터 바인딩 추상화: PropertyEditor

[org.springframework.validation.DataBinder](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/validation/DataBinder.html)

* 기술적인 관점: 프로퍼티 값을 타겟 객체에 설정하는 기능

* 사용자 관점: 사용자 입력값을 애플리케이션 도메인 모델에 동적으로 변환해 넣어주는 기능.

* 해석하자면, 입력값은 대부분 “문자열”인데, 그 값을 객체가 가지고 있는 int, long, Boolean, Date 등 심지어 Event, Book 같은 도메인 타입으로도 변환해서 넣어주는 기능.

**PropertyEditor**

*  스프링 3.0 이전까지 DataBinder가 변환 작업 사용하던 인터페이스
* 쓰레드-세이프 하지 않음 (상태 정보 저장 하고 있음, 따라서 싱글톤 빈으로 등록해서 쓰다가는...)
* Object와 String 간의 변환만 할 수 있어, 사용 범위가 제한적 임. (그래도 그런 경우가 대부분이기 때문에 잘 사용해 왔음. 조심해서..)

```java
public class EventPropertyEditor extends PropertyEditorSupport {
  @Override
  public String getAsText() {
    return ((Event)getValue()).getTitle();
  }
  @Override
  public void setAsText(String text) throws IllegalArgumentException {
    int id = Integer.parseInt(text);
    Event event = new Event();
    event.setId(id);
    setValue(event);
  }
}
```

---

> 애플리케이션 도메인 객체에 할당할 때, 바인딩이 필요한 이유 ?

사용자가 입력한 값은 보통 문자열이다. 이 문자열을 객체가 가지고 있는 다양한 프로퍼티 타입(int, long, Boolean)과 같은 타입으로 변환해서 넣어야한다.

* [org.springframework.validation.DataBinder](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/validation/DataBinder.html) : 스프링에서 제공해주는 데이터 바인딩 기능 인터페이스

# DataBinder

[org.springframework.validation.DataBinder](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/validation/DataBinder.html)

* 스프링이 제공해주는 데이터 바인딩 인터페이스
* 주로 웹 MVC 에서 사용

**PropertyEditor를 사용하는 DataBinder**

* 스프링 웹 MVC 뿐만아니라

* 이전에 xml 설정파일로 ApplicationContext를 구성하던 시기에 

  xml 설정파일에 입력한 문자열을 빈이 가지고 있는 적절한 타입으로 변환해서 넣어줄 때도 사용되었다.

* SpEL(Spring expression language)에서도 사용된다.

데이터 바인딩에 관련된 기능을 여러 인터페이스로 추상화 시켜 놓았다.



# 고전적인 방식의 데이터 바인딩

## 1. 도메인 클래스 생성

```java
public class Event {

    private Integer id;

    private String title;

    public Event(Integer id) {
        this.id = id;
    }

  	// ... getter, setter
}
```



## 2. 컨트롤러 생성

`/event/1`, `/event/2` 와 같이 url에 event id 값으로 요청하면, event id에 해당하는 값을 `Event`객체로 바인딩해서 받는다.

그리고 Event 도메인을 가지고 코딩을 하면 된다.

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



## 3. 테스트코드 작성

테스트코드 작성 시 import하는 패키지 주의!!

```java
package dev.solar.demospring51;

//import org.junit.jupiter.api.Test; //이거 아님
import org.junit.Test; //<-- *
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;

import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content; //<-- *
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status; //<-- *

@RunWith(SpringRunner.class)
@WebMvcTest
public class EventControllerTest {

    @Autowired
    MockMvc mockMvc;

    @Test
    public void getTest() throws Exception {
        mockMvc.perform(MockMvcRequestBuilders.get("/event/1"))
                .andExpect(status().isOk()) //상태코드값이 200(OK)인지 확인
                .andExpect(content().string("1"));
    }
}
```

```
에러 코드 : Failed to convert value of type 'java.lang.String' to required type 'dev.solar.demospring51.Event'; nested exception is java.lang.IllegalStateException: Cannot convert value of type 'java.lang.String' to required type 'dev.solar.demospring51.Event': no matching editors(<- property) or conversion strategy(<- conversion service) found]

MockHttpServletRequest:
      HTTP Method = GET
      Request URI = /event/1
       Parameters = {}
          Headers = []
             Body = <no character encoding set>
    Session Attrs = {}
    
    ...
MockHttpServletResponse:
           Status = 500
    Error message = null
          Headers = []
     Content type = null
             Body = 
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
          
java.lang.AssertionError: Status 
Expected :200
Actual   :500
```

⇒ 500 서버 에러가 발생하고 있다.

url에 있는 String 타입의 id값을 Event 타입으로 바인딩하지 못했기 때문에 발생한 에러이다.



## 4. 프로퍼티 에디터 생성

(1) PropertyEditor

* 자바 클래스인 `PropertyEditor` 인터페이스를 직접 구현해도 된다.

* 하지만, 구현해야할 메서드가 많다.

* ```java
  public class EventEditor implements PropertyEditor { ... }
  ```

  ![PropertyEditor](/assets/images/SPRING/core-technology/8HlNS6L.png)



(2) PropertyEditorSupport

* 대신, `PropertyEditorSupport` 클래스를 상속받아서 사용하면 된다.
* 구현할 메서드를 선택해서 구현할 수 있다.

* 보통 `getAsText()`, `setAsText()` 두 개를 구현하면 된다.

  

우리에게 필요한 경우는 text를 Event로 변환하는 기능만 필요하므로 setAsText()만 구현하면 된다.

값이 들어오는 것은 문자열이지만 이를 숫자 타입으로 변환할 것이다.

```java
public class EventEditor extends PropertyEditorSupport {

  @Override
  public String getAsText() {
    //PropertyEditor가 받은 객체를 getValue()로 가져올 수 있다. (반환타입은 Object) Event로 형변환 필요
    Event event = (Event)getValue();
    return event.getId().toString();
  }

  @Override
  public void setAsText(String text) throws IllegalArgumentException {
    setValue(new Event(Integer.parseInt(text))); //text를 Integer로 변환 후 Event 객체의 id값으로 해서 Event 객체 생성
  }
}
```

 

# PropertyEditor는 빈으로 등록해서 사용하면 안된다. ★중요★ 

 `getValue()`, `setValue()` 가 공유하는 값이 PropertyEditor가 가지고 있는 값이다.

그래서 이 값이 서로 다른 스레드에게 공유가 된다. 따라서 `Stateful` 하다.

상태정보를 저장하고 있다. 따라서 **`Thread Safe 하지 않다.`**

그렇기 때문에 PropertyEditor의 구현체들은 여러 스레드에 공유해서 사용하면 안된다.

**즉, 구현체를 `@Component`와 같은 어노테이션으로 그냥 빈으로 등록해서 사용하면 안된다.!!!!**

```java
@Component //<---- 이렇게 빈으로 등록하지 말 것!!
public class EventEditor extends PropertyEditorSupport {
  ...
}
```

⇒ 1번 회원이 2번 회원의 정보를 변경하는 등. 심각한 문제를 발생시킬 수 있다.



그냥 빈이아니라 `Thread 스코프`의 빈으로 만들어서 사용하면 괜찮다.

한 스레드 내에서만 유효한 스포프의 빈으로 정의해서 쓰면 그나마 괜찮긴하지만 절대 등록하지 않고 사용하는 것을 권장한다.

어떠한 경우에라도 절대로 다른 스레드와 공유하면 안되므로 빈으로 등록해서 사용하지 못한다. 그것이 안전하다.



> 그러면 어떻게 PropertyEditor를 사용할까?

컨트롤러에서 `@InitBinder`로 데이터 바인더들을 등록해서 사용하면 된다.



`webDataBinder`에 Event.class 타입을 처리할 PropertyEditor를 등록할 수 있다.

`WebDataBinder`는 DataBinder의 구현체 중 하나이다.

```java
@RestController
public class EventController {

    @InitBinder
    public void init(WebDataBinder webDataBinder) {
        webDataBinder.registerCustomEditor(Event.class, new EventEditor());
    }

    @GetMapping("/event/{event}")
    public String getEvent(@PathVariable Event event) {
        System.out.println(event);
        return event.getId().toString();
    }
}
```

```
Event{id=1, title='null'}
```



컨트롤러가 어떠한 요청을 처리하기 전에, 컨트롤러에서 정의한 데이터 바인더에 들어있는 PropertyEditor를 사용하게된다. 따라서 PropertyEditor를 사용했기 때문에 문자열로 들어온 "1" 값을 숫자로 변환해서 Event 객체로 변환하는 일이 이루어졌기 때문에 테스트가 성공적으로 이루어진 것이다.



# 단점

* 이러한 PropertyEditor를 구현해서 사용하는 방식은 편리하지 않고, 스레드 세이프하지도 않아서 빈으로 등록해서 사용하기도 위험하다.

* Object와 String 간의 변환만 할 수 있어, 사용 범위가 제한적 임. (그래도 그런 경우가 대부분이기 때문에 잘 사용해 왔음. 조심해서..)

스프링 3 부터는 또 다른 이런 데이터 바인딩과 관련된 데이터바인딩과 관련된 인터페이스와 기능들이 추가되었다. 다음 시간에 추가된 컨버터와 포메터에 대해서 살펴보자

