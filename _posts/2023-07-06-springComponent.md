---
title: (Spring) 컴포넌트 스캔
author: pjh5365
date: 2023-07-06 09:00:00 +0900
categories: [BackEnd, Spring/Spring Boot]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: SpringLogo

---

## 컴포넌트 스캔

컴포넌트 스캔은 스프링이 직접 클래스를 검색해서 빈으로 등록해주는 기능이다. 설정 클래스에 빈으로 등록하지 않아도 원하는 클래스를 빈으로 등록할 수 있으므로 컴포넌트 스캔 기능을 사용하면 설정 코드가 크게 줄어든다.

## @Component 어노테이션

스프링이 검색해서 빈으로 등록할려면 `@Component` 어노테이션이 붙어있어야 한다. `@Component` 어노테이션은 해당 클래스를 스캔 대상으로 표시한다. `@Component` 어노테이션에 속성값을 정해줄 수 있는데 속성값이 주어질 경우엔 주어진 속성값으로 빈 이름을 등록한다.

```java
@Component
public class MemberDao {
	//이 경우 빈으로 등록할때 사용하는 이름은 클래스의 첫 글자를 소문자로 바꾼 memberDao 를 사용한다.
}
```

```java
@Component("infoPrinter")
public class MemberInfoPrinter {
	//이 경우 MemberInfoPrinter 클래스이지만 빈 이름으로 infoPrinter 를 사용한다.
}
```

## @ComponentScan 어노테이션

`@Component` 어노테이션을 붙인 클래스를 스캔하여 스프링 빈으로 등록하려면 설정 클래스에 `@ComponentScan` 어노테이션을 적용해야 한다.

```java
package pjh5365.exam;

import org.springframework.stereotype.Component;

@Component	//ApplicationConfig를 위한 어노테이션 
public class Engine {
	public Engine() {
		System.out.println("Engine 생성자");
	}
	public void exec() {
		System.out.println("exec 메소드");
	}
}
```

```java
package pjh5365.exam;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration	//설정파일로 등록
@ComponentScan(basePackages = {"pjh5365.exam"})	//반드시 스캔할 패키지 지정하기
public class ApplicationConfig {

}
```

`@Component` 어노테이션을 붙인 클래스만 컴포넌트 스캔 대상에 포함되는 것이 아니라 아래의 어노테이션을 붙인 클래스가 컴포넌트 스캔 대상에 포함된다.

- @Component
- @Controller
- @Service
- @Repository
- @Aspect
- @Configuration

`@Aspect` 어노테이션을 제외한 나머지 어노테이션은 실제로 `@Component` 어노테이션에 대한 특수 어노테이션이다. 

## 컴포넌트 스캔에 따른 충돌 처리

컴포넌트 스캔 기능을 사용해서 자동으로 빈으로 등록할 때에는 충돌에 주의해야 하는데 크게 빈 이름 충돌과 수동 등록에 따른 충돌이 발생할 수 있다. 충돌을 피하기 위해선 컴포넌트 스캔 과정에서 서로 다른 타입인데 같은 빈 이름을 사용하는 경우를 찾아 둘 중 하나에 명시적으로 빈 이름을 지정해주어야 충돌을 피할 수 있다.