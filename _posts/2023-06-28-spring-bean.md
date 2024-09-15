---
title: (Spring) Bean 등록하기
author: pjh5365
date: 2023-06-28 09:00:00 +0900
categories: [BackEnd, Spring]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: SpringLogo

---

## JAVA Bean?

JAVA Bean이란 자바 객체를 만드는 규약으로 다른 클래스에서 재사용 가능한 자바객체를 만들기 위한 규약이다. 자바 빈은 기본 생성자가 필요하며 모든 멤버 변수를 private로 선언하고 getter/setter메서드 (프로퍼티)를 통해서만 접근이 가능하다.

## Spring Bean?

스프링에 의해 생성되고, 라이프 사이클을 수행하고, 의존성 주입이 일어나는 객체를 말하는데 즉 스프링IoC 컨테이너에서 관리되는 객체를 빈이라고 부른다. 개발자가 관리하는 객체가 아닌 스프링에 제어권을 넘긴 객체이며 싱글톤 방식으로 관리된다.

## Bean 등록하기

### - 사용되는 클래스(Bean)

#### Car 클래스

```java
package pjh5365.exam;

public class Car {
	private Engine v8;
	
	public Car() {
		System.out.println("Car 생성자");
	}
	
	public void setEngine(Engine e) {
		this.v8 = e;
	}
	public void run() {
		System.out.println("엔진을 이용해 달린다.");
		v8.exec();
	}
}

```

#### Engine 클래스

```java
package pjh5365.exam;

public class Engine {

	public Engine() {
		System.out.println("Engine 생성자");
	}
	public void exec() {
		System.out.println("exec 메소드");
	}
}

```

### - XML을 이용한 등록

src/main/resources 폴더에 applicationContext.xml로 파일을 생성하고 Bean을 등록하고 ApplicationContextExam01 클래스를 생성하여 빈이 등록되었는지 확인하는 예제

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="engine" class="pjh5365.exam.Engine" />	<!--단순히 Engine 클래스 등록  -->
	<bean id="car" class="pjh5365.exam.Car" > 	<!--Car 클래스 등록과 setEngine()메소드까지 실행시켜서 v8에 값을 주입-->
		<property name="engine" ref="engine"></property>
	</bean>
</beans>
```

```java
package pjh5365.exam;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class ApplicationContextExam01 {
	
	public static void main(String[] args) {
		ApplicationContext ac = new ClassPathXmlApplicationContext("applicationContext.xml");	//xml설정파일 불러오기
//		Engine e = (Engine)ac.getBean("engine");	
		Car c = (Car)ac.getBean("car");	//불러올 때 형 변환이 필수
    
//		c.setEngine(e);	xml파일에서 bean사이에 property로 엔진을 넣어주어 setEngine(e)와 같은 효과를 낸다.
		c.run();
    
/*		Bean을 사용하지 않는다면 아래처럼 사용해야 하지만 Bean을 사용해 2줄로 끝난다.    
		Engine e = new Engine();
		Car c = new Car();
		c.setEngine(e);
		c.run();	*/
	}
}

```

### - Annotation을 이용한 등록 1

xml파일을 생성하지 않고 어노테이션만을 사용하여 빈을 등록한다. 등록을 위한 ApplicationConfig 클래스에 @Configuration 어노테이션을 사용해 설정파일로 이용한다고 알리고 각 클래스를 @Bean 어노테이션을 사용해 빈으로 등록한다.

```java
package pjh5365.exam;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ApplicationConfig {

	@Bean
	public Car car(Engine e) {
		Car c = new Car();
		c.setEngine(e);
		return c;
	}
	@Bean
	public Engine engine() {
		return new Engine();
	}
}

```

```java
package pjh5365.exam;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class ApplicationContextExam02 {
	
	public static void main(String[] args) {
		ApplicationContext ac = new AnnotationConfigApplicationContext(ApplicationConfig.class);	//설정파일 불러오기
		
//		Car car = (Car) ac.getBean("car");
		Car car = (Car) ac.getBean(Car.class);	//위의 방법보다 이 방법을 사용하여 오타에 의한 오류를 줄인다.
		car.run();
	}
}

```

### - Annotation을 이용한 등록 2

각각의 클래스에도 어노테이션을 붙여 더욱 간편하게 사용할 수 있다.

#### Car 클래스

```java
package pjh5365.exam;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component	//ApplicationConfig2를 위한 어노테이션
public class Car {

	@Autowired	//setEngine메소드 없이 값을 알아서 넣을 수 있는 어노테이션
	private Engine v8;
	
	public Car() {
		System.out.println("Car 생성자");
	}
	
//	public void setEngine(Engine e) {
//		this.v8 = e;
//	}
	public void run() {
		System.out.println("엔진을 이용해 달린다.");
		v8.exec();
	}
}
```

#### Engine 클래스

```java
package pjh5365.exam;

import org.springframework.stereotype.Component;

@Component	//ApplicationConfig2를 위한 어노테이션 
public class Engine {

	public Engine() {
		System.out.println("Engine 생성자");
	}
	public void exec() {
		System.out.println("exec 메소드");
	}
}
```

아래의 방법처럼 설정파일로 등록하고 자동으로 패키지를 스캔하여 빈으로 등록할 수 있다.

```java
package pjh5365.exam;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration	//설정파일로 등록
@ComponentScan("pjh5365.exam")	//반드시 스캔할 패키지 지정하기
public class ApplicationConfig2 {

}
```

```java
package pjh5365.exam;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class ApplicationContextExam03 {
	
	public static void main(String[] args) {
		ApplicationContext ac = new AnnotationConfigApplicationContext(ApplicationConfig2.class);
		
		Car car = (Car) ac.getBean(Car.class);
		car.run();
	}

}
```