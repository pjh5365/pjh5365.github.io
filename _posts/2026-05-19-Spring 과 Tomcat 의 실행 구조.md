---
title: Spring 과 Tomcat 의 실행 구조
author: pjh5365
date: 2026-5-19 21:09:00 +0900
categories: [BackEnd, Spring]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: Spring
---

스프링 애플리케이션은 흔히 톰캣 위에서 실행된다고 이야기한다. 둘은 어떤 관계로 동작하는지 자세히 알아보자.

## Tomcat은 무엇인가?
Tomcat(톰캣)을 웹 서버 정도로만 이해하지만, 사실 **톰캣 자체도 JVM 위에서 실행되는 하나의 자바 애플리케이션**으로, 크게 두 가지 역할을 수행한다.

### 1. HTTP Server
- TCP Socket 오픈
- HTTP 요청 수신
- HTTP 응답 반환
- Keep-Alive 관리
- Connection 관리

같은 네트워크 레벨 처리를 담당한다.

### 2. Servlet Container

- Servlet 객체 생성
- Servlet 생명주기 관리
- URL 매핑
- Request / Response 객체 생성
- Thread Pool 관리

등을 수행한다. 즉, **서블릿의 실행 환경을 제공**하며 **단순 HTTP 서버가 아니라 서블릿 기반 자바 웹 애플리케이션 실행 엔진**이다.

### 구조
```
OS
 └── JVM Process
      └── Tomcat(Java Program)
           └── Spring Application
```


## Tomcat도 JVM 위에서 실행된다.

톰캣 실행 스크립트를 간단히 보면 아래 형태다.
```bash
java org.apache.catalina.startup.Bootstrap
```

`java` 명령으로 JVM을 띄우고, 톰캣 클래스들을 실행하는 구조로 아래와 같은 결론에 도달한다.

- Tomcat은 JVM 내부 객체다.
- Spring도 JVM 내부 객체다.
- DispatcherServlet도 JVM 내부 객체다.

즉, 톰캣과 스프링이 서로 다른 프로그램처럼 보일 수 있지만 실제로는 **하나의 JVM 프로세스 내부 객체들**이다.

그래서 톰캣이 스프링 코드를 실행시킨다는 말의 의미는 실제로 톰캣의 `ClassLoader` 가 스프링 클래스들을 로딩하고, `Servlet` 객체를 생성하며, 클라이언트 요청 시 디스패처 서블릿의 메서드를 호출한다는 것이다.

### 구조
```
Tomcat
 ├── ClassLoader
 │    └── Spring Classes 로딩
 │
 ├── Servlet Container
 │    └── DispatcherServlet 생성
 │
 └── Thread Pool
      └── DispatcherServlet.service() 호출
```

## Spring Application은 Tomcat 내부에서 실행된다.
`Spring MVC` 의 핵심은 디스패처 서블릿(DispatcherServlet)이다.

스프링이 실행되면 디스패처 서블릿이 생성되고, 톰캣(서블릿 컨테이너)에 등록된다.

```java
tomcat.addServlet("/", dispatcherServlet); // 이런 느낌으로 등록
```

## 실제 HTTP 요청 처리 흐름

### 전체 구조
```
Browser
   ↓
Tomcat(Socket Accept)
   ↓
Tomcat Thread Pool
   ↓
DispatcherServlet
   ↓
HandlerMapping
   ↓
Controller
   ↓
Service
   ↓
Repository
   ↓
DB
```

### 요청에 따른 내부 처리 과정

#### 1. 브라우저 요청

```http
GET /users/1 HTTP/1.1
```

브라우저가 TCP 연결을 통해 요청을 전송한다.

#### 2. Tomcat이 요청 수신

톰캣은 내부적으로 `ServerSocket`을 열고 대기하고, 요청이 오면 `Acceptor Thread` 가 연결을 받는다.

#### 3. Tomcat Thread Pool 에서 Worker Thread 할당

톰캣은 요청마다 스레드를 새로 생성하지 않는다. 대신 `Thread Pool`을 사용한다. 즉, `Spring MVC` 동기 처리 기준에서는 **요청 1개 = Worker Thread 1개를 점유**한다. 이후 해당 스레드를 통해 디스패처 서블릿을 호출한다.

## Tomcat Thread와 Spring Thread의 차이?

기본적인 `Spring MVC` 의 요청은 `Tomcat Worker Thread`가 처리한다. 즉, **스프링이 요청용 스레드를 따로 생성하지 않고, 톰캣이 생성한 스레드를 사용**한다.

### 실제 흐름

```
Tomcat Thread (이 스레드를 사용해 로직을 처리한다.)
   ↓
DispatcherServlet
   ↓
Controller
   ↓
Service
   ↓
Repository
```

### Spring은 언제 별도 스레드를 생성할까?

스프링은 `@Async`, `CompletableFuture`, `@Scheduled` 같은 경우에 생성한다. 하지만 결국 모두 같은 JVM 내부 스레드다. 프로세스가 여러 개 생기는 게 아니다.

## Spring Boot에서는 왜 톰캣 설치가 필요가 없을까?

스프링부트(Spring Boot)의 핵심 기능 중 하나는 `Embedded Tomcat` 이다. 즉, 내부에 톰캣이 존재하기에 따로 설치할 필요가 없다.

### Spring Boot 실행 시 내부에서 벌어지는 일

```
JVM 시작
   ↓
Spring Boot Main 실행
   ↓
SpringApplication.run()
   ↓
내장 Tomcat 생성
   ↓
Tomcat.start()
   ↓
DispatcherServlet 등록
   ↓
HTTP 요청 대기
```

### 내장 톰캣 생성 과정

```java
Tomcat tomcat = new Tomcat();
tomcat.start();
```

스프링부트가 톰캣 객체를 직접 생성하기 때문에 외부 톰캣 설치가 필요 없다.

### 외장 톰캣 vs 내장 톰캣

- 외장 톰캣: Tomcat 설치 -> WAR 배포
- 내장 톰캣: JAR 내부에 Tomcat 포함으로 애플리케이션 자체가 Tomcat + Spring을 함께 실행한다.


## 결론

스프링과 톰캣은 완전히 분리된 독립 서버가 아니다. 실제로는 하나의 JVM 프로세스 내부에서 톰캣 객체와 스프링 객체들이 서로 메서드를 호출하며 동작한다.

> 즉, 톰캣과 스프링은 HTTP 요청 -> Tomcat 객체 -> DispatcherServlet 객체 -> Controller 객체 -> Service 객체로 이어지는 **JVM 내부 객체 호출 체인**이다.
{: .prompt-tip }

### Spring이 직접 HTTP 요청을 받는가?

HTTP 요청은 톰캣이 소켓 레벨에서 먼저 받고 디스패처 서블릿으로 넘겨주는 단계부터 스프링이 관여한다.

### Tomcat이 Spring 소스코드를 컴파일하는가?

`.java` 파일을 읽은 것이 아닌 이미 컴파일된 `.class(ByteCode)` 를 읽어서 `ClassLoader`가 로딩한다.

### Tomcat과 Spring 프로세스가 2개 실행되는가?

하나의 JVM 프로세스 내부에서 톰캣과 스프링이 함께 동작한다.

### Spring 모든 Thread를 관리하는가?

기본 요청 스레드는 톰캣 스레드 풀이 관리한다. 스프링은 `@Async`, `Scheduler`, 비동기 처리 같은 추가 스레드 풀만 관리한다.

### 전체 실행 구조 

```
OS
 └── JVM Process
      └── Tomcat
           ├── HTTP Server
           │
           ├── Thread Pool
           │
           └── Servlet Container
                │
                └── DispatcherServlet(HttpServlet)
                       │
                       └── Spring Container
                            ├── Controller
                            ├── Service
                            └── Repository
```