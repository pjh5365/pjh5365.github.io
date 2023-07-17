---
title: (Spring Boot) Validation
author: pjh5365
date: 2023-07-16 09:00:00 +0900
categories: [BackEnd, Spring/Spring Boot]
tags: [backend, springboot]
image:
  path: /assets/img/springlogo.png
  alt: SpringLogo

---

## Validation

객체의 유효성을 검사하는 방법으로 객체의 요청이 들어올 때 서버에서 원하는 값으로 들어오는지 검증하는 기능이다. 원래 검증하기 위해선 if문 같은 조건문을 사용해 할 수 있지만 검증해야할 값이 많은 경우 코드의 길이가 매우 길어질 수 있기 때문에 스프링에선 어노테이션 기반으로 제공을 한다.

#### 어노테이션 종류

|     어노테이션      |            설명             |
| :-----------------: | :-------------------------: |
|        @Size        |       문자 길이 측정        |
|      @NotNull       |          null 불가          |
|      @NotEmpty      |        null, "" 불가        |
|      @NotBlank      |     null, "", " " 불가      |
|        @Past        |          과거 날짜          |
|   @PastOrPresent    |     오늘이나 과거 날짜      |
|       @Future       |          미래 날짜          |
|  @FutureOrPresent   |    오늘이거나 미래 날짜     |
|      @Pattern       |         정규식 적용         |
|        @Max         |           최대값            |
|        @Min         |           최소값            |
| @AssertTure / False |       별로 Logic 적용       |
|       @Valid        | 해당 Object validation 실행 |

### 사용 예시

#### - User

```java
package pjh5365.validation.dto;

import javax.validation.constraints.Email;
import javax.validation.constraints.Pattern;

public class User {
    private String name;
    private int age;
    @Email  //이메일 형식만 받도록 지정
    private String email;
    @Pattern(regexp = "^\\d{2,3}-\\d{3,4}-\\d{4}$") //휴대폰 번호 정규식
    private String phoneNumber;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String setEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }

    public void setPhoneNumber(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", email='" + email + '\'' +
                ", phoneNumber='" + phoneNumber + '\'' +
                '}';
    }
}
```

#### - ApiController

```java
package pjh5365.validation.controller;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import pjh5365.validation.dto.User;

import javax.validation.Valid;

@RestController
@RequestMapping("/api")
public class ApiController {

    @PostMapping("/user")
    public User user(@Valid @RequestBody User user) {   //validation 적용어노테이션과 json을 받기위한 어노테이션
        System.out.println(user);

        return user;
    }
}
```

위와 같은 방법을 사용하면 json 형태로 값을 받을 때 이메일형식과 전화번호 형식에 맞춰 들어올때만 정상적으로 작동한다.

## BindingResult 를 이용한 에러남기기

Validation만 사용한다면 따로 에러메시지를 남길 수 없기 때문에 BindingResult를 사용하여 에러메시지를 사용자에게 보낼 수 있다.

### 사용 예시

#### - User

```java
package pjh5365.validation.dto;

import javax.validation.constraints.Email;
import javax.validation.constraints.Pattern;

public class User {
    private String name;
    private int age;
    @Email  //이메일 형식만 받도록 지정
    private String email;
    @Pattern(regexp = "^\\d{2,3}-\\d{3,4}-\\d{4}$", message = "핸드폰 번호의 양식과 맞지 않습니다. xxx-xxx(x)-xxxx") //휴대폰 번호 정규식
    private String phoneNumber;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }

    public void setPhoneNumber(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", email='" + email + '\'' +
                ", phoneNumber='" + phoneNumber + '\'' +
                '}';
    }
}
```

#### - ApiController

```java
package pjh5365.validation.controller;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.BindingResult;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import pjh5365.validation.dto.User;

import javax.validation.Valid;

@RestController
@RequestMapping("/api")
public class ApiController {

    @PostMapping("/user")
    public ResponseEntity user(@Valid @RequestBody User user, BindingResult bindingResult) {   //validation 적용어노테이션과 json을 받기위한 어노테이션

        if(bindingResult.hasErrors()) {
            StringBuilder sb = new StringBuilder();
            bindingResult.getAllErrors().forEach(objectError -> {
                FieldError field = (FieldError) objectError;
                String msg = objectError.getDefaultMessage();

                System.out.println("field : " + field.getField());
                System.out.println(msg);

                sb.append("field : " + field.getField());
                sb.append("message : " + msg);
            });

            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(sb.toString());
        }
        System.out.println(user);

        return ResponseEntity.status(HttpStatus.OK).body(user);
    }
}
```

#### 잘못된 형식으로 값을 전달했을 때

![잘못된 형식으로 보냈을 때](/assets/img/2023-07-16-Validation/result1.png)


## Custom Validation 만들기

스프링에서 주어진 validation말고도 필요한 형식으로 validation을 만들 수 있다.

### 사용 예시

#### - User

```java
package pjh5365.validation.dto;

import javax.validation.constraints.*;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class User {
    @NotBlank
    private String name;
    @Max(value = 90)
    private int age;
    @Email  //이메일 형식만 받도록 지정
    private String email;
    @Pattern(regexp = "^\\d{2,3}-\\d{3,4}-\\d{4}$", message = "핸드폰 번호의 양식과 맞지 않습니다. xxx-xxx(x)-xxxx") //휴대폰 번호 정규식
    private String phoneNumber;

    @Size(min = 6, max = 6)
    private String reqYearMonth;    //yyyMM

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }

    public void setPhoneNumber(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    public String getReqYearMonth() {
        return reqYearMonth;
    }

    public void setReqYearMonth(String reqYearMonth) {
        this.reqYearMonth = reqYearMonth;
    }

    @AssertTrue(message = "yyyyMM 의 형식이어야 합니다.") //커스텀 validation으로 참일때만 정상작동
    public boolean isReqYearMonth() {   //AssertTrue를 사용하면 메서드가 is키워드로 시작해야 정상작동함.
        try {   //뒤에 날짜만 추가해서 형식맞추기
            LocalDate localDate = LocalDate.parse(this.reqYearMonth + "01", DateTimeFormatter.ofPattern("yyyyMMdd"));
        } catch (Exception e) {
            return false;
        }

        return true;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", email='" + email + '\'' +
                ", phoneNumber='" + phoneNumber + '\'' +
                ", reqYearMonth='" + reqYearMonth + '\'' +
                '}';
    }
}
```

위의 예시처럼 `@AssertTure` 어노테이션을 사용하여 커스텀할 수 있다. 하지만 이 방법을 사용하면 해당 클래스에서만 사용가능하므로 재활용이 불가능한데, 재활용 가능한 validation을 만들고 싶다면 어노테이션을 만들어 사용하면 된다.

### 어노테이션 예시

#### - YearMonth

```java
package pjh5365.validation.annotation;

import pjh5365.validation.validator.YearMonthValidator;

import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.*;
import static java.lang.annotation.ElementType.TYPE_USE;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

@Constraint(validatedBy = {YearMonthValidator.class})   //만들어준 validator 클래스 넣어주기
@Target({ METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER, TYPE_USE })
@Retention(RUNTIME)
public @interface YearMonth {
    String message() default "yyyyMM 형식에 맞지 않습니다.";

    Class<?>[] groups() default { };

    Class<? extends Payload>[] payload() default { };

    String pattern() default "yyyyMMdd";
}
```

#### - YearMonthValidator

```java
package pjh5365.validation.validator;

import pjh5365.validation.annotation.YearMonth;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class YearMonthValidator implements ConstraintValidator<YearMonth, String> {

    private String pattern;

    @Override
    public void initialize(YearMonth constraintAnnotation) {
        this.pattern = constraintAnnotation.pattern();
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        try {   //뒤에 날짜만 추가해서 형식맞추기
            LocalDate localDate = LocalDate.parse(value + "01", DateTimeFormatter.ofPattern(this.pattern));
        } catch (Exception e) {
            return false;
        }

        return true;
    }
}
```

#### - User

```java
package pjh5365.validation.dto;

import pjh5365.validation.annotation.YearMonth;

import javax.validation.constraints.*;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class User {
    @NotBlank
    private String name;
    @Max(value = 90)
    private int age;
    @Email  //이메일 형식만 받도록 지정
    private String email;
    @Pattern(regexp = "^\\d{2,3}-\\d{3,4}-\\d{4}$", message = "핸드폰 번호의 양식과 맞지 않습니다. xxx-xxx(x)-xxxx") //휴대폰 번호 정규식
    private String phoneNumber;

    @YearMonth	//커스텀 어노테이션
    private String reqYearMonth;    //yyyMM

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public String getPhoneNumber() {
        return phoneNumber;
    }

    public void setPhoneNumber(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    public String getReqYearMonth() {
        return reqYearMonth;
    }

    public void setReqYearMonth(String reqYearMonth) {
        this.reqYearMonth = reqYearMonth;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                ", email='" + email + '\'' +
                ", phoneNumber='" + phoneNumber + '\'' +
                ", reqYearMonth='" + reqYearMonth + '\'' +
                '}';
    }
}
```