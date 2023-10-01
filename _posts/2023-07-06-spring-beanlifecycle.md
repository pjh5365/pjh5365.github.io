---
title: (Spring) 빈 라이프 사이클
author: pjh5365
date: 2023-07-06 18:00:00 +0900
categories: [BackEnd, Spring/Spring Boot]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: SpringLogo

---

## 스프링 컨테이너의 초기화와 종료

스프링 컨테이너는 초기화와 종료라는 라이프사이클을 가진다.

```java
//컨테이너 초기화
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(ApplicationConfig.class);

//컨터이너에서 빈 객체를 가져와 사용
Member member = ac.getBean(Member.class);
member.insert("아이디", "비밀번호");

//컨테이너 종료
ac.close();
```

위 코드에선 `AnnotationConfigApplicationContext` 의 생성자를 이용해 컨텍스트 객체를 생성하는데 이 시점에 스프링 컨테이너를 초기화 시킨다. 스프링 컨테이너는 설정 클래스에서 정보를 읽어와 알맞은 빈 객체를 생성하고 각 빈을 연결(의존주입)하는 작업을 수행한다.

이후 컨테이너 초기화가 완료되면 컨테이너를 사용할 수 있다. 컨테이너를 사용하는 것은 `getBean()` 과 같은 메서드를 이용해 컨테이너에 보관된 빈 객체를 구하는 것이다.

컨테이너 사용이 끝나면 컨테이너를 종료한다.

- 컨테이너 초기화 : 빈 객체의 생성, 의존 주입, 초기화
- 컨테이너의 종료 : 빈 객체의 소멸

스프링 컨테이너의 라이프사이클에 따라 빈 객체도 자연스럽게 생성과 소멸이라는 라이프사이클을 갖는다.

## 스프링 빈 객체의 라이프사이클

스프링 컨테이너는 빈 객체의 라이프사이클을 관리한다. 컨테이너가 관리하는 빈 객체의 라이프 사이클은 **객체생성 -> 의존설정 -> 초기화 -> 소멸** 순서로 이루어진다. 스프링 컨테이너를 초기화 할 때 스프링 컨테이너는 가장 먼저 빈 객체를 생성하고 의존을 설정하는데 이 시점에서 의존 자동 주입을 통한 의존 설정이 수행된다. 이후 모든 의존 설정이 완료되면 빈 객체의 초기화를 수행한다. 이때 빈 객체를 초기화하기 위해 스프링은 빈 객체의 지정된 메서드를 호출한다. 마지막으로 스프링 컨테이너를 종료하면 스프링 컨테이너는 빈 객체의 소멸을 처리한다. 이때에도 지정한 메서드를 호출한다.

스프링 컨테이너는 빈 객체를 초기화하고 소멸하기 위해 빈 객체의 지정한 메소드를 호출하는데 스프링은 아래의 두 인터페이스에 해당 메서드를 정의하고 있다.

- org.springframework.beans.factory.InitializingBean
- org.springframework.beans.factory.DisposableBean

```java
public interface InitializingBean {	//초기화 과정
    void afterPropertiesSet() throws Exception;
}

public interface DisposableBean {	//소멸 과정
    void destory() throws Exception;
}
```

빈 객체를 생성하고 초기화나 소멸 과정이 필요하면 클래스를 작성할 때 각각의 인터페이스를 상속받고 안에 존재하는 메서드를 알맞게 구현하면 된다. 이러한 방법 말고도 `@Bean` 태그에서 `initMethod` 속성과 `destoryMethod` 속성을 사용하여 초기화 메서드와 소멸 메서드의 이름을 지정할 수 있다.

```java
public class Client {
    private String host;
  	
    public void setHost(String host) {
        this.host = host;
    }
  	
    public void connect() {
        System.out.println("connect() 실행");
    }
  
    public void send() {
        System.out.println("send() 실행");
    }
  	
    public void close() {
        System.out.println("close() 실행");
    }
}
```

위의 코드중 `connect()` 메서드와 `close()` 메서드를 초기화와 소멸과정에 사용할 메서드로 지정하기

```java
@Bean(initMethod = "connect", destoryMethod = "close")
public Client client() {
    Client client = new Client();
    client.setHost("host");
    return client;
}
```

> `initMethod` 속성과 `destroyMethod` 속성에 지정한 메서드는 파라미터가 없어야 한다. 이 두 속성에 지정한 메서드가 파라미터가 존재할 경우 스프링 컨테이너는 예외를 발생시킨다. 
{: .prompt-danger }

이처럼 초기화 관련 코드를 따로 작성했다면 초기화 메서드가 두 번 호출되지 않도록 유의해야 한다.

## 빈 객체의 생성과 관리 범위

일반적으로 스프링 컨테이너는 빈 객체를 한 개만 생성하여 싱글톤 범위를 갖는다. 빈 객체를 구할 때마다 매번 새로운 객체를 생성하고 싶다면 프로토타입 범위의 빈을 설정할 수 있다. 사용방법으로는 특정 빈에 `@Scope("prototype")` 어노테이션을 사용하면 된다.

```java
@Configuration
public class ApplicationConfig {
    @Bean
    @Scope("prototype")
    public Client client() {
        Client client = new Client();
        client.setHost("host");
        return client;
    }
}
```

싱글톤 범위를 명시적으로 지정하고 싶다면 `@Scope` 어노테이션에 값으로 `singleton` 을 주면 된다.

```java
@Configuration
public class ApplicationConfig {
    @Bean
    @Scope("singleton")
    public Client client() {
        Client client = new Client();
        client.setHost("host");
        return client;
    }
}
```

프로토타입 범위를 갖는 빈은 완전한 라이프사이클을 따르지 않는다. 스프링 컨테이너는 프로토타입의 빈 객체를 생성하고 프로퍼티를 설정하고 초기화하는 작업까지는 수행하지만, 컨테이너를 종료한다고 해서 생성한 프로토타입 빈 객체의 소멸 메서드를 실행하지는 않는다. 따라서 프로토타입 범위의 빈을 사용할 때는 빈 객체의 소멸 처리를 코드에서 직접 해야 한다.