---
title: (Spring) AOP
author: pjh5365
date: 2023-07-13 09:00:00 +0900
categories: [BackEnd, Spring/Spring Boot]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: SpringLogo

---



## AOP

AOP는 Aspect Oriented Programming 의 약자로, 여러 객체에 공통으로 적용할 수 있는 기능을 분리해서 재사용성을 높여주는 프로그래밍 기법이다. AOP는 핵심 기능과 공통 기능의 구현을 분리함으로써 핵심 기능을 구현한 코드의 수정 없이 공통 기능을 적용할 수 있게 해준다. AOP의 기본개념은 핵심 기능에 공통 기능을 삽입하는 것이다.

### 핵심 기능에 공통 기능을 삽입하는 방법

- 컴파일 시점에 코드에 공통 기능을 삽입
- 클래스 로딩 시점에 바이트 코드에 공통 기능을 삽입
- 런타임에 프록시 객체를 생성해서 공통 기능을 삽입

위의 3가지 방법중에 1,2 번은 스프링 AOP에서 지원하지 않으며 AspectJ와 같이 AOP전용 도구를 사용해서 적용할 수 있다. 스프링이 제공하는 AOP 방식은 프록시를 이용한 세번째 방법이다. 이때 **프록시는 핵심 기능의 실행은 다른 객체에 위임하고 부가적인 기능을 제공하는 객체**를 말한다. 프록시 방식은 중간에 프록시 객체를 생성하는 것으로 실제 객체의 기능을 실행하기 전・후에 공통 기능을 호출한다. 

### AOP 주요 용어

|   용어    |                             의미                             |
| :-------: | :----------------------------------------------------------: |
|  Advice   |   언제 공통 관심 기능을 핵심 로직에 적용할 지를 정의한다.    |
| Joinpoint | Advice를 적용 가능한 지점을 의미한다. 메서드 호출, 필드 값 변경 등이 Joinpoint에 해당하며 스프링은 프록시를 이용해서 AOP를 구현하기 때문에 메서드 호출에 대한 Joinpoint만 지원한다. |
| Pointcut  | Joinpoint의 부분 집합으로서 실제 Advice가 적용되는 Joinpoint를 나타낸다. 스프링에서는 정규 표현식이나 AspectJ의 문법을 이용하여 Pointcut을 정의할 수 있다. |
|  Weaving  |       Advice를 핵심 로직 코드에 적용하는 것을 말한다.        |
|  Aspect   |         여러 객체에 공통으로 적용되는 기능을 말한다.         |

#### Advice 의 종류

|          종류          |                             설명                             |
| :--------------------: | :----------------------------------------------------------: |
|     Before Advice      |      대상 객체의 메서드 호출 전에 공통 기능을 실행한다.      |
| After Returning Advice | 대상 객체의 메서드가 예외없이 실행된 이후에 공통 기능을 실행한다. |
| After Throwing Advice  | 대상 객체의 메서드를 실행하는 도중 예외가 발생한 경우에 공통 기능을 실행한다. |
|      After Advice      | 예외 발생 여부에 상관없이 대상 객체의 메서드 실행 후 공통 기능을 실행한다. (try-catch-finally 의 finally 역할) |
|     Around Advice      | 대상 객체의 메서드 실행 전, 후 또는 예외 발생 시점에 공통 기능을 실행하는데 사용된다. |

이 중 널리 사용되는 것은 Around Advice로 사용 이유로는 대상 객체의 메서드 실행 전/후, 예외 처리 발생 시점 등 다양한 시점에 원하는 기능을 삽입 할 수 있기 때문이다.

## 스프링 AOP 구현

스프링 AOP를 이용해서 공통 기능을 구현하고 적용하는 방법은 아래의 절차를 따른다.

- Aspect로 사용할 클래스에 @Aspect 어노테이션을 붙인다.
- @Pointcut 어노테이션으로 공통 기능을 적용할 Pointcut을 정의한다.
- 공통 기능을 구현한 메서드에 @Around 어노테이션을 적용한다.

개발자는 공통 기능을 제공하는 Aspect 구현 클래스를 만들고 자바 설정을 이용해 Aspect를 어디에 적용할지 설정하면 된다. Aspect는 `@Aspect` 어노테이션을 이용하여 구현하며 스프링 프레임워크가 알아서 프록시를 만들어 준다. @Aspect 어노테이션을 붙인 클래스를 공통 기능으로 적용하려면 `@EnableAspectJAutoProxy` 어노테이션을 설정 클래스에 붙여야 한다.

#### 예시

```java
package pjh5365;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;

@Aspect
public class ExeTimeAspect {

    @Pointcut("execution(public * pjh5365..*(..))") //해당 패키지의 모든 메서드에 적용
    private void publicTarget() {}

    @Around("publicTarget()")   //publicTarget()메서드에 정의한 Pointcut에 공통기능을 적용
    public Object measure(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();	//실행 전
        try {
            Object result = joinPoint.proceed();    //비지니스 로직 실행
            return result;
        } finally {
            long finish = System.nanoTime();	//실행 후
            Signature sig = joinPoint.getSignature();
        }
    }
}

```

```java
package pjh5365;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy
public class ApplicationConfig {
    
    @Bean
    public ExeTimeAspect exeTimeAspect() {
        return new ExeTimeAspect();
    }
}
```



## 프록시 생성 방식

스프링은 AOP를 위한 프록시 객체를 생성할 때 실제 생성한 빈 객체가 인터페이스를 상속하면 인터페이스를 이용해서 프록시를 생성한다.