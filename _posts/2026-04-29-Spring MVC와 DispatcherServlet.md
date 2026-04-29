---
title: Spring MVC와 DispatcherServlet
author: pjh5365
date: 2026-4-29 14:58:00 +0900
categories: [BackEnd, Spring]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: Spring
---

[이전 글](https://pjh5365.github.io/posts/Servlet%EA%B3%BC-Servlet-Container/)에선 `Servlet` 과 `Servlet Container` 에 대해 다뤘다면, 이번에는 `Spring MVC` 가 `Servlet` 기반 위에서 어떻게 동작하는지, 그리고 `DispatcherServlet` 이 어떤 역할을 하는지 알아보겠다.

## Spring MVC란?

`Spring MVC` 는 **Servlet 스펙 위에서 동작하는 웹 프레임워크**이다. 즉, `Apache Tomcat` 같은 `Servlet Container` 가 HTTP 요청을 받고, `Spring MVC` 가 그 요청을 구조화된 방식으로 처리한다.

### 처리 순서

```
브라우저 요청
   ↓
톰캣 (Servlet Container)
   ↓
DispatcherServlet (Front Controller)
   ↓
Controller
   ↓
Service
   ↓
Repository
```

**`Spring MVC`는 서블릿 자체를 대체하는 것이 아니라** 서블릿 위에서 동작하는 프레임워크이다. `Spring MVC`에서는 **디스패처 서블릿 (DispatcherServlet) 하나가 모든 클라이언트 요청의 중앙 진입점(Front Controller) 역할을 수행한다.** 

**즉, `Spring MVC` 서블릿이 디스패처 서블릿 하나만 존재한다** (개념적 기본 구조이고 커스텀 서블릿을 추가해서 사용할 수 있다).

따라서, `Spring MVC` 를 사용하면 개발자는 직접 여러 서블릿을 작성할 필요 없이, `Controller`, `Service` 등 비즈니스 로직 처리에만 집중하면 된다.

> 개발자가 구현하는 컨트롤러는 `HttpServlet` 을 상속받는 서블릿이 아닌 일반적인 `POJO`(Plain Old Java Object)이다.

> 스프링에서 Controller 를 POJO라고 부르는 이유는, Spring Framework의 어노테이션을 사용하더라도 특정 클래스를 상속하거나 인터페이스 구현을 강제받지 않는 **일반 자바 객체 형태를 유지하기 때문**이다.
{: .prompt-tip }

## DispatcherServlet이란?

디스패처 서블릿 (DispatcherServlet)은 Spring MVC의 핵심 서블릿으로, **모든 HTTP 요청을 중앙에서 받아 적절한 컨트롤러로 전달하는 `Front Controller` 패턴의 구현체이다.**

- 톰캣 -> 요청을 받는 서버
- 디스패처 서블릿 -> 총괄 관리자
- 컨트롤러 -> 실제 업무 담당자

### DispatcherServlet의 역할

1. **요청 수신:** 모든 클라이언트 요청의 단일 진입점 역할 수행한다.
2. **HandlerMapping:** 요청 URL에 맞는 컨트롤러(핸들러)를 탐색한다. (어떤 컨트롤러가 이 요청을 처리할지 찾는다)
3. **HandlerAdapter:** 선택된 컨트롤러를 실제 실행 가능한 방식으로 호출한다.
4. **Controller 실행:** 비즈니스 로직을 수행한다.
5. **ViewResolver:** 논리적 뷰 이름을 실제 View 객체로 변환한다. (HTML/JSP 등)
6. **응답 반환:** 최종 HTTP Response를 반환한다.

### 처리 순서

```
Client Request
   ↓
DispatcherServlet
   ↓
HandlerMapping
   ↓
HandlerAdapter
   ↓
Controller
   ↓
Service
   ↓
ViewResolver
   ↓
Client Response
```

즉, `DispatcherServlet` 은 요청을 받고 -> 컨트롤러를 찾아서 실행하고 -> 결과를 반환한다. (따라서 개발자는 서블릿을 직접 구현하기보다, `Controller·Service·Repository` 등 비즈니스 로직 구현에 집중할 수 있다.)

## Spring MVC 구조의 장점

### 중앙 집중식 요청 처리

- 공통 로직 처리가 쉽다.
- 인증, 인가, 로깅, 예외 처리 일관성을 확보할 수 있다.

### 책임 분리

- `DispatcherServlet` → 요청 제어
- `Controller` → 요청 처리
- `Service` → 비즈니스 로직
- `Repository` → 데이터 접근

### 전략 패턴 기반 확장성

- `HandlerMapping` 교체 가능
- `ViewResolver` 교체 가능
- `MessageConverter` 확장 가능

### 유지보수성 향상

기능 추가나 수정 시 전체 구조를 변경하지 않고 특정 계층만 수정 가능하다.