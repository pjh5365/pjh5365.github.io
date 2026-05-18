---
title: Spring MVC의 DispatcherServlet은 어떻게 동시에 수천 개의 요청을 처리할까?
author: pjh5365
date: 2026-5-18 22:02:00 +0900
categories: [BackEnd, Spring]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: Spring
---

[이전 글](https://pjh5365.github.io/posts/Spring-MVC%EC%99%80-DispatcherServlet/) 에선 `Spring MVC` 가 **디스패처 서블릿 (DispatcherServlet)** 하나로 모든 HTTP 요청을 처리한다고 했었다. 

그럼 아래 같은 의문이 생길 수 있다.

- 디스패처 서블릿이 하나만 존재하니 클라이언트의 요청도 하나씩 처리할까?
- 싱글톤(Singleton) 패턴이니 여러 요청이 동시에 접근하면 동시성 문제가 발생하지 않을까?
- 사용자 수천 명이 동시에 접근하면 병목이 생기지 않을까?

하지만, 실제 `Spring MVC` 는 수천 개의 요청을 동시에 처리할 수 있다. 그 이유를 알아보자.

## Spring MVC의 핵심 구조
```
클라이언트 요청
      ↓
Tomcat Connector
      ↓
Tomcat Thread Pool에서 Thread 하나 할당
      ↓
DispatcherServlet 호출
      ↓
Controller 실행
      ↓
Service / Repository 호출 (비즈니스 로직 및 DB 처리)
      ↓
응답 반환
```

`Spring MVC` 가 동시에 많은 요청을 처리할 수 있는 핵심 이유는 디스패처 서블릿은 하나지만, **요청마다 별도의 스레드(Thread)가 할당되어 실행**하기 때문이다. 

즉, 디스패처 서블릿 자체가 클라이언트 요청을 순차적으로 처리하는 것이 아니라, 여러 스레드가 동시에 디스패처 서블릿의 메서드를 호출하는 구조이다.

## DispatcherServlet은 하나인데 어떻게 동시에 요청을 처리할까?

`Spring MVC` 에서는 디스패처 서블릿을 하나의 서블릿 인스턴스로 생성해 사용하며, **스프링 빈(Spring Bean)** 으로도 등록되어 사실상 싱글톤처럼 동작하기에 메모리에 객체가 하나만 생성된다.

객체가 하나인데 여러 요청을 동시에 처리해도 괜찮겠냐는 생각이 들 수도 있지만, 하나의 객체라도 상태(State)를 공유하지 않는다면 여러 스레드가 동시에 호출해도 안전하게 동작한다.

```java
public class Calculator {

    public int add(int a, int b) {
        return a + b;
    }
}
```

위와 같은 객체는 하나만 있어도 `Thread-A`, `Thread-B`, `Thread-C` 가 동시에 `add()` 메서드를 호출할 수 있다. `add()` 메서드가 외부 상태를 변경하지 않고, 오직 메서드 내부의 지역 변수만 사용하기 때문이다.

디스패처 서블릿도 이런 방식으로 동작하기에, 여러 스레드가 동시에 접근할 수 있다.

`Spring MVC` 에서는 여러 요청이 하나의 빈(Bean) 객체를 함께 사용하므로, 일반적으로 **요청 간 상태를 저장하지 않는 무상태(Stateless) 방식으로 설계**한다.

즉, **빈이 상태 공유하지 않도록 설계되어 있으므로** 하나의 객체를 여러 스레드가 동시에 사용해도 안전하게 동작할 수 있다.

다만, 이러한 Stateless 설계를 지키지 않으면 동시성 문제 같은 심각한 문제가 발생할 수 있다.

> `Spring MVC` 의 Thread는 `Apache Tomcat` 같은 **서블릿 컨테이너(Servlet Container)가 관리**한다.
{: .prompt-tip }

### Tomcat Thread Pool 구조

톰캣은 요청이 들어올 때마다 새로운 스레드를 생성하지 않고, 미리 스레드를 만들어 둔 뒤 재사용한다. 이를 **스레드 풀(Thread Pool)** 이라고 한다.

#### 구조
```
[클라이언트 요청]

사용자 A ─┐
사용자 B ─┼──> Tomcat
사용자 C ─┘
                ↓

           [Thread Pool]

              Thread-1
              Thread-2
              Thread-3
              Thread-4
              ...
```

클라이언트의 요청이 들어오게 되면 톰캣은 각 요청에 대해 스레드를 할당 해주고, **여러 스레드가 동시에 하나의 디스패처 서블릿에 접근**한다. 이를 **Spring MVC의 멀티스레드 기반 처리 구조**라고 한다.

즉, Spring MVC의 높은 동시성 처리 성능은

- Singleton 객체 재사용
- Stateless 설계
- Tomcat Thread Pool
- 멀티스레드 기반 요청 처리

가 함께 동작한 결과라고 볼 수 있다.


## DispatcherServlet 에서는 왜 병목이 잘 발생하지 않을까?

디스패처 서블릿은 직접 비즈니스 로직을 처리하지 않는다. 주요 역할은 아래와 같은 교통정리 역할만 담당한다.

1. 요청받기
2. 어떤 Controller로 보낼지 찾기
3. Controller 호출
4. 결과 반환

따라서, 대부분의 경우 **실제 병목은 디스패처 서블릿 자체보다는 DB 조회, 외부 API 호출, 네트워크 IO 같은 작업에서 발생**한다.

> 결론적으로 `Spring MVC` 는 단순히 디스패처 서블릿 하나가 요청을 처리하는 구조가 아니라, **Tomcat의 멀티스레드 환경 위에서 Stateless 하게 설계된 객체들을 효율적으로 재사용하는 구조**라고 이해할 수 있다.
{: .prompt-info }