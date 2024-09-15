---
title: (Spring) MVC
author: pjh5365
date: 2023-07-01 09:00:00 +0900
categories: [BackEnd, Spring]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: SpringLogo

---

## MVC

Model-View-Controller의 약자로 어플리케이션을 세 가지 역할로 구분한 디자인 패턴이다.

- Model : 뷰가 렌더링하는데 필요한 데이터로 사용자가 요청한 상품목록이나 주문 내역이 해당한다.
- View : 웹에서 뷰는 실제로 보이는 부분으로 모델을 사용하여 렌더링한다. 뷰는 JSP, JSF, PDF, XML 등으로 결과를 표현한다.
- Controller : 사용자의 액션에 응답하는 컴포넌트로 모델을 업데이트하고 다른 액션을 수행한다.

## MVC 모델 2가지

### - 모델 1

브라우저가 요청을 보내면 해당 요청을 JSP가 받게 되고 요청만큼 JSP 페이지가 존재해야한다. 또 이런 JSP는 JAVA로 만들어진 클래스인 JAVA Bean을 통해 DB를 사용하고 출력하게 되므로 뷰와 컨트롤러의 역할이 합쳐진다고 볼 수 있다. 모델 1 방식으로 처리하게 된다면 JSP자체에 JAVA코드와 HTML 코드가 섞여 유지 보수가 어렵다.

![MVC모델1](/assets/img/2023-07-01-springMVC/MVC1.png)


### - 모델 2

모든 요청을 프론트 컨트롤러(Dispatcher Servlet)이 받고 실제 처리는 컨트롤러 클래스(핸들러 클래스)에게 위임한다. 이런 컨트롤러들은 JAVA Bean등을 통하여 결과를 만들어내고 만들어진 결과를 모델에 담아 프론트 컨트롤러에 보낸다. 프론트 컨트롤러는 알맞은 뷰에 모델을 전달하여 그 값을 출력하는데 이러한 웹 모듈을 보통 Spring MVC라고 한다.

![MVC모델2](/assets/img/2023-07-01-springMVC/MVC2.png)


## SpringMVC 예제

>/plusform이라고 요청을 보내면 2개의 값을 입력받는 창과 버튼이 존재하는 화면을 출력하고 값을 입력 후 버튼을 누르면 /plus URL로 2개의 입력이 POST 방식으로 서버에 전달해 서버에서 해당 값을 더하여 JSP에게 request scope으로 전달하여 출력
{: .prompt-tip }

### - web.xml 파일로 Dispatcher Servlet을 프론트 컨트롤러로 설정하기

#### DispatcherServlet이 실행될 때 읽어들이는 설정파일

```java
package mvcexam2.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.DefaultServletHandlerConfigurer;
import org.springframework.web.servlet.config.annotation.EnableWebMvc;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.ViewControllerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;
import org.springframework.web.servlet.view.InternalResourceViewResolver;

@Configuration	//자바 config파일임을 알려주는 어노테이션
@EnableWebMvc	//웹에 필요한 빈들을 대부분 자동으로 설정해주는 어노테이션
//다른 설정이 더 필요하다면 WebMvcConfigurer인터페이스를 받아와 필요한 메소드를 오버라이딩 해주면 된다.
@ComponentScan(basePackages = {"mvcexam2.controller"})	//컨트롤러가 존재하는 곳의 경로를 패키지를 포함하여 적어준다
//컴포넌트 스캔으로 Controller, Service, Repository, Componet 어노테이션이 붙은 클래스를 찾아 스프링 컨테이너로 관리한다.
public class WebMvcContextConfiguration implements WebMvcConfigurer {	

	//모든자원이 넘어오니 각종 파일들의 경로를 지정해줌
	@Override
	public void addResourceHandlers(ResourceHandlerRegistry registry) {
		
		registry.addResourceHandler("/css/**").addResourceLocations("/css/");
		registry.addResourceHandler("/img/**").addResourceLocations("/img/");
		registry.addResourceHandler("/js/**").addResourceLocations("/js/");
	}

	//매핑정보가 없다면 Spring의 DefaultServletHttpRequestHandler가 처리하도록 하는 메서드
	//DefaultServletHandler를 사용하게 설정함
	//이렇게 된다면 WAS의 DefaultSerlvet으로 일을 넘기고 WAS는 DefaultServlet의 static한 자원을 읽어 보여줌 
	@Override
	public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
		configurer.enable();
	}

	//특정 url에 대한 처리를 컨트롤러 클래스를 작성하지 않고 매핑할 수 있도록 해주는 메서드로
	// '/' 로 요청이 들어오면 main이라는 이름의 뷰로 보여줌 main을 찾을때는 아래의 viewResolver로 찾음
	@Override
	public void addViewControllers(ViewControllerRegistry registry) {

		System.out.println("addViewController 호출");
		registry.addViewController("/").setViewName("main");
	}
	
	@Bean
	public InternalResourceViewResolver getInternalResourceViewResolver() {
		InternalResourceViewResolver resolver = new InternalResourceViewResolver();
		resolver.setPrefix("/WEB-INF/views/");	//이름 앞에 경로를 붙이고
		resolver.setSuffix(".jsp");	//이름 뒤에 파일명을 붙임
		return resolver;
	}
	
}
```

#### web.xml 파일

```xml
<?xml version="1.0" encoding="UTF-8"?>

<web-app>
	<display-name>Archetype Created Web Application</display-name>
	<servlet>
	 	<servlet-name>mvc</servlet-name>
	 	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	 	<init-param>
	 		<param-name>contextClass</param-name>
	 		<param-value>org.springframework.web.context.support.AnnotationConfigWebApplicationContext</param-value>
	 	</init-param>
	 	<init-param>
	 		<param-name>contextConfigLocation</param-name>
	 		<param-value>mvcexam2.config.WebMvcContextConfiguration</param-value>
	 		<!-- 자바 설정파일의 경로를 패키지명을 포함하여 적어줘야 한다. -->
	 	</init-param>
	 	<load-on-startup>1</load-on-startup>
	</servlet>
  	<servlet-mapping>
		<servlet-name>mvc</servlet-name>
 		<url-pattern>/</url-pattern>
  		<!-- 모든 요청을 받을 수 있게 / 로 설정 -->
	</servlet-mapping>
</web-app>
```

web.xml파일은 다른 서블릿과 동일하게 등록하듯이 등록하면 되며, param-value는 내가 설정한 자바 config 파일을 지정해 준다. 또한 url패턴을 `/` 로 하는데 이는 모든 요청을 디스패쳐서블릿에서 처리하기 위함이다.

### - JSP파일 생성하기

모든 JSP파일은 WebMvcContextConfiguration 클래스에서 설정한 경로에 만들어주어야 정상작동한다. 값을 입력할 페이지와 결과를 출력할 페이지를 생성해준다.

#### plusForm.jsp 파일

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>
	<form method="post" action="plus">
		value1 : <input type="text" name="value1"><br>
		value2 : <input type="text" name="value2"><br>
		<input type="submit" value="확인">
	</form>
</body>
</html>
```

#### plusResult.jsp 파일

```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>
	${value1 } + ${value2 } = ${result }
</body>
</html>
```

### - 컨트롤러 만들기

WebMvcContextConfiguration 클래스에서 ComponentScan 어노테이션에 설정한 basePackages의 경로와 같은 곳에 생성해야한다.

#### 컨트롤러 파일

```java
package mvcexam2.controller;

import org.springframework.stereotype.Controller;
import org.springframework.ui.ModelMap;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;

@Controller	//컨트롤러라고 알려주는 어노테이션
public class PlusController {

	@GetMapping("/plusform")	//get방식으로 /plusform으로 요청이 들어왔을때 실행 할 메소드
	public String plusform() {
		return "plusForm";	//뷰의 이름을 넘겨줌
	}
	
	@PostMapping("/plus")
	public String plus(@RequestParam(name = "value1", required = true) int value1, @RequestParam(name = "value2", required = true) int value2, ModelMap modelMap) {
		int result = value1 + value2;
		//@RequestParam	어노테이션으로 넘어오는 파라미터의 값을 가져올 수 있다.
		//ModelMap을 이용하면 HttpServletRequest를 사용해서 생기는 종속적인 문제를 해결할 수 있다.
		//스프링이 알아서 request scope에 매핑시켜준다.
		modelMap.addAttribute("value1", value1);
		modelMap.addAttribute("value2", value2);
		modelMap.addAttribute("result", result);
		
		return "plusResult";
	}
	
}
```

### 실행결과

![실행결과1](/assets/img/2023-07-01-springMVC/result1.png)
![실행결과2](/assets/img/2023-07-01-springMVC/result2.png)
![실행결과3](/assets/img/2023-07-01-springMVC/result3.png)