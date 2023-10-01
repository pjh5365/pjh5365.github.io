---
title: (Spring) JdbcTemplate
author: pjh5365
date: 2023-07-14 09:00:00 +0900
categories: [BackEnd, Spring/Spring Boot]
tags: [backend, spring]
image:
  path: /assets/img/springlogo.png
  alt: SpringLogo

---



## JdbcTemplate

스프링에서는 JDBC API 를 이용하면 구조적인 반복이 생기는데 이 반복을 줄이기 위해 템플릿 메서드 패턴과 전략 패턴을 엮은 JdbcTemplate 클래스를 제공한다. 스프링이 제공하는 DB 연동 기능은 DataSource를 사용해서 DB Connection을 구하는데 방법으로는 DB 연동에 사용할 DataSource를 스프링 빈으로 등록하고 DB 연동 기능을 구현한 빈 객체는 DataSource를 주입받아 사용한다.

> 커넥션 풀 (Connection pool)
>
> 커넥션 풀은 일정 개수의 DB 커넥션을 미리 만들어두는 기법이다. DB 커넥션이 필요하다면 생성하지 않고 커넥션 풀에서 커넥션을 가져와 사용한 뒤 커넥션을 다시 풀에 반납한다. 커넥션을 미리 생성해두기 때문에 커넥션을 사용하는 시점에서 커넥션 생성 시간을 아낄 수 있으며 동시 접속자가 많아도 커넥션을 생성하는 부하가 적기 때문에 더 많은 동시 접속자를 처리할 수 있다. 또한 커넥션도 일정 개수로 유지해서 DBMS에 대한 부하를 일정 수준으로 유지할 수 있게 해준다.
{: .prompt-tip}

#### 예시

```java
package pjh5365;

import org.springframework.jdbc.core.JdbcTemplate;

import javax.sql.DataSource;

public class MemberDao {
    private JdbcTemplate jdbcTemplate;

    public MemberDao(DataSource dataSource) {
        this.jdbcTemplate = new JdbcTemplate(dataSource);
    }
}
```

```java
package config;

import org.apache.tomcat.jdbc.pool.DataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import pjh5365.MemberDao;

@Configuration
public class ApplicationConfig {

    @Bean(destroyMethod = "close")  //소멸메서드로 close 메서드를 이용함. 해당 메서드는 커넥션 풀에 보관된 Connection을 닫음
    public DataSource dataSource() {
        DataSource dataSource = new DataSource();
        dataSource.setDriverClassName("org.mariadb.jdbc.Driver");
        dataSource.setUrl("jdbc:mariadb://localhost~");	//db주소 입력
        dataSource.setUsername("");	//사용자명 입력
        dataSource.setPassword("");	//비밀번호 입력

        return dataSource;
    }

    @Bean
    public MemberDao memberDao() {
        return new MemberDao(dataSource());
    }
}

```

## 트랜잭션

트랜잭션이란 데이터베이스의 상태를 변환시키는 하나의 논리적 기능을 수행하기 위한 작업의 단위로 더 이상 쪼갤 수 없는 최소 단위의 작업을 말한다. 다르게 말하면 두 개 이상의 쿼리를 한 작업으로 실행할 때 사용하는 것이라 볼 수 있다. 트랜잭션은 여러 쿼리를 논리적으로 하나의 작업으로 묶어준다. 한 트랜잭션으로 묶인 쿼리 중 하나라도 실패하면 전체 쿼리를 실패로 간주하고 실패 이전에 실행한 쿼리를 취소한다. 이렇게 쿼리 실행 결과를 취소하고 DB를 기존 상태로 되돌리는 것을 `롤백(rollback)` 이라 부른다. 반면 트랜잭션으로 묶인 모든 쿼리가 성공하여 쿼리 결과를 DB에 실제로 반영하는 것을 `커밋(commit)` 이라고 한다.

- commit : 트랜잭션의 작업을 마치고 DB에 반영
- rollback : 트랜잭션 작업 도중 오류가 발생할 경우 작업을 취소하고 시작하기 이전의 상태로 되돌림

스프링에선 `@Transactional` 어노테이션을 사용하면 트랜잭션 범위를 쉽게 지정할 수 있다. @Transactional 어노테이션이 제대로 동작하려면 2가지 설정이 추가되어야 한다.

- 플랫폼 트랜잭션 매니저 (PlatformTranscationManager) 빈 설정
- @Transcational 어노테이션 활성화 설정

#### 예시

```java
package pjh5365;

import org.springframework.transaction.annotation.Transactional;

public class ChangePasswordService {
    private MemberDao memberDao;

    @Transactional
    public void changePassword(String email, String oldPwd, String newPwd) {
        Member member = memberDao.selectByEmail(email);
      
        if(member == null)
            throw new MemberNotFoundException();

        member.changePassword(oldPwd, newPwd);
        memberDao.update(member);
    }

    public void setMemberDao(MemberDao memberDao) {
        this.memberDao = memberDao;
    }
}
```

```java
package config;

import org.apache.tomcat.jdbc.pool.DataSource;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.annotation.EnableTransactionManagement;
import pjh5365.MemberDao;

@Configuration
@EnableTransactionManagement    //트랜잭션을 사용하기 위한 어노테이션
public class ApplicationConfig {

    @Bean(destroyMethod = "close")  //소멸메서드로 close 메서드를 이용함. 해당 메서드는 커넥션 풀에 보관된 Connection을 닫음
    public DataSource dataSource() {
        DataSource dataSource = new DataSource();
        dataSource.setDriverClassName("org.mariadb.jdbc.Driver");
        dataSource.setUrl("jdbc:mariadb://localhost~");	//db주소 입력
        dataSource.setUsername("");	//사용자명 입력
        dataSource.setPassword("");	//비밀번호 입력

        return dataSource;
    }

    @Bean   //플랫폼 트랜잭션 매니저 빈 설정
    public PlatformTransactionManager transactionManager() {
        DataSourceTransactionManager tm = new DataSourceTransactionManager();
        tm.setDataSource(dataSource());
        return tm;
    }
    
    @Bean
    public MemberDao memberDao() {
        return new MemberDao(dataSource());
    }
    @Bean
    public ChangePasswordService changePasswordService() {
        ChangePasswordService changePasswordService = new ChangePasswordService();
        changePasswordService.setMemberDao(memberDao());
        return changePasswordService;
    }
}
```

스프링은 @Transactional 어노테이션을 이용해서 트랜잭션을 처리하기 위해 내부적으로 AOP를 사용한다. 스프링은 @Transactional 어노테이션을 적용하기 위해 @EnableTransactionManagement 어노테이션을 사용하면 스프링은 @Transcational 어노테이션이 적용된 객체를 찾아서 알맞은 프록시 객체를 생성한다.