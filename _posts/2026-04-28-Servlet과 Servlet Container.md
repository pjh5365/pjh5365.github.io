---
title: Servlet과 Servlet Container
author: pjh5365
date: 2026-4-28 14:51:00 +0900
categories: [BackEnd, Spring]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: Spring
---

Spring Framework를 제대로 공부하기 위해선 먼저 Servlet과 Servlet Container에 대해 이해해야 한다. Spring MVC가 Servlet 기반으로 동작하기 때문이다.

## Servlet이란?

서블릿(Servlet)이란 자바 기반 웹 애플리케이션에서 클라이언트의 요청(Request)을 처리하고, 응답(Response)을 생성하기 위한 **표준 기술(스펙)**이다.

즉, **서블릿 자체는 특정 기술을 구현한 구현체가 아닌 자바 웹 서버 환경에서 동작하기 위한 규칙이다.**

## Servlet Container란?

서블릿은 규칙일 뿐이므로, 이 **서블릿 스펙을 기반으로 작성된 서블릿 구현체를 실제 실행하려면 이를 관리하고 실행할 환경이 필요**하다. 서블릿을 생성, 실행, 관리하는 실행 환경(엔진)을 서블릿 컨테이너(Servlet Container)라고 부른다. 대표적으로 `Apache Tomcat` 이 있다.


> **Apache Tomcat?**
><br>
Apache Tomcat은 HTTP 서버 기능과 Servlet Container 기능을 함께 제공하는 대표적인 WAS(Web Application Server) 이다.
{: .prompt-info }

클라이언트의 요청이 들어오면 아래와 같이 동작한다.

```
클라이언트(브라우저) <-> 서블릿 컨테이너(톰캣) <-> 서블릿(구현체)
```
## Servlet Container의 주요 역할

### Servlet의 생명주기(Life Cycle) 관리

서블릿 컨테이너는 서블릿 객체의 전체 생명주기를 관리한다.

#### 생명주기

- `init()`
  - 서블릿 최초 생성 시 단 1회 실행
  - 초기 설정 수행
- `service()`
  - 클라이언트의 각 요청마다 실행
  - `GET` / `POST` 등 HTTP 메서드에 따라 분기된다.
- `destroy()`
  - 서버 종료 시 실행
  - 자원을 정리한다.

### 요청(Request)/응답(Response) 객체 생성

클라이언트의 요청이 들어오면 `HttpServletRequest` 와 `HttpServletResponse` 를 생성하고 서블릿은 이를 전달받아 처리한다.

### URL 매핑

서블릿 컨테이너가 요청 URL에 맞는 서블릿을 찾아 실행한다. 

- Ex) `/user/login` 요청 시  -> `LoginServlet` 실행

### 멀티 스레드 처리

대량의 요청 처리를 위해 각 요청을 별도의 스레드에서 처리한다.

1. 스레드 풀(Thread Pool)에 스레드를 생성한다.
2. 클라이언트의 요청이 들어오면 스레드 풀에서 스레드를 꺼낸다.
3. 꺼낸 스레드를 이용해 요청을 처리한다.
4. 요청이 처리된 스레드는 스레드 풀로 반환된다.
5. 만약, 스레드 풀에 여유 스레드가 존재하지 않는다면 큐에서 대기한다.

서블릿 컨테이너가 위의 작업들을 대신 수행하기에 개발자는 비즈니스 로직 개발에만 집중할 수 있다.

## Servlet의 특징

서블릿 컨테이너는 일반적으로 하나의 서블릿 객체를 생성하고 여러 요청에서 재사용한다. 

> 서블릿은 서블릿 컨테이너가 단일 인스턴스로 관리하지만, 이는 **GoF 싱글톤 패턴이 아니라 컨테이너 관리 방식이다.**
{: .prompt-tip }

**장점**

- 서블릿 당 하나만 생성되므로 메모리를 효율적으로 사용할 수 있다.
- 최초 생성 시에만 객체를 생성하므로, 객체 생성 비용이 감소된다.
- 객체 생성 및 초기화 비용이 줄어들기 때문에 빠른 요청을 처리할 수 있다.

**단점**

- 여러 스레드가 동시에 접근 가능하다.
- 인스턴스 변수 사용 시 동시성 문제가 발생할 수 있다.
따라서 위의 단점을 해결하기 위해 **지역 변수 사용 및 무상태(Stateless) 설계가 필수이며, Thread-Safe 한 코드를 작성**해야 한다.


## 정리

### Servlet

- 자바 웹 요청/응답 처리를 위한 표준 기술

### Servlet Container

- Servlet 실행 환경
- Servlet 생명주기 관리
- Servlet Thread 관리
- 클라이언트 요청 처리

### Servlet Container가 개발자 대신 처리하는 작업

- 클라이언트 요청 수신
- HTTP 파싱
- URL 분석 및 적절한 Servlet 선택
- Servlet 객체 생성 및 메서드 호출
- Servlet 스레드 관리
- 응답 반환