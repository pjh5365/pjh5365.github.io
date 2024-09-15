---
title: Naver 검색 API를 이용한 쇼핑몰 검색서비스 만들기
author: pjh5365
date: 2023-07-21 14:23:00 +0900
categories: [BackEnd, Spring]
tags: [backend, springboot]
image:
  path: /assets/img/springlogo.png
  alt: SpringLogo

---

패스트캠퍼스의 강의 속 네이버 검색 API를 이용한 맛집리스트만들기를 보고 그냥 따라하기만 하면 실력이 늘지 않을 것 같아 강의를 참고하여 네이버 쇼핑API를 이용하여 검색하는 예제를 만들어 보았다. 서버와 값을 주고받기위해 `request`와 `response`가 필요했고 서버와 통신하기 위한 클라이언트와 컨트롤러, 서비스를 만들고 테스트는 하나만 해 보았다. 또한 API에 필요한 키값들은 `application.yml` 파일을 이용하여 사용하고 `.gitignore` 파일을 이용하여 숨겼다. 원래 `.yml` 대신 `.properties` 파일로 하였으나 프로퍼티 설정이 정상적으로 적용되지 않아 에러가 발생하여 `.yml` 로 진행하였다.

## dto 설정

네이버 서버와 값을 주고받기 위해 [네이버 쇼핑몰 API](https://developers.naver.com/docs/serviceapi/search/shopping/shopping.md#%EC%87%BC%ED%95%91) 를 보고 request를 작성한다. 단순히 연습을 위해 만들기 때문에 query와 dispaly만 설정해서 간단하게만 검색한다.

![api문서 1](/assets/img/2023-07-21-NaverShoppingApi/1.png)

#### - ShoppingRequset

```java
package pjh5365.navershoppingapi.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;

@Data   //getter, setter 생성
@AllArgsConstructor //모든 필드가 들어간 생성자 만들기
@NoArgsConstructor  //기본 생성자 만들기
public class ShoppingRequest {
    private String query = "";   //검색어
    private Integer display = 1;    //검색결과 갯수는 1개만

    public MultiValueMap map() {    //파라미터를 넘기기 위한 맵
        LinkedMultiValueMap<String, String > map = new LinkedMultiValueMap();
        map.add("query", query);
        map.add("display", String.valueOf(display));

        return map;
    }
}
```

응답 역시 [네이버 쇼핑몰 API](https://developers.naver.com/docs/serviceapi/search/shopping/shopping.md#%EC%87%BC%ED%95%91) 를 보고 작성한다. 이때 아이템 리스트는 바로 static 클래스로 선언하여 사용한다.

#### - ShoppingResponse

```java
package pjh5365.navershoppingapi.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Date;
import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class ShoppingResponse {
    private Date lastBuildDate;     //검색결과를 생성한 시간
    private String total;   //총 검색 결과
    private List<ShoppingItem> items;   //아이템을 받을 리스트

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ShoppingItem {
        private String title;   //상품이름
        private String link;    //상품 url
        private String image;   //상품 이미지 url
        private Integer lprice; //최저가
        private Integer hprice; //최고가
        private String mallName;    //상품을 판매하는 쇼핑몰
        private String maker;   //제조사
        private String brand;   //브랜드
    }
}
```

## 서버와 통신하기 위한 클라이언트 설정

네이버 서버와의 통신을 위해 클라이언트를 설정해준다. 해더의 값은 API 문서에 나와있는 양식대로 넣어야 한다.

![api 문서 2](/assets/img/2023-07-21-NaverShoppingApi/2.png)

#### - NaverClient

```java
package pjh5365.navershoppingapi.shopping;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.util.UriComponentsBuilder;
import pjh5365.navershoppingapi.dto.ShoppingRequest;
import pjh5365.navershoppingapi.dto.ShoppingResponse;

import java.net.URI;

@Component  //빈으로 등록
public class NaverClient {
    @Value("${naver.url}")
    private String naverUrl;
    @Value("${naver.clientId}")
    private String clientId;
    @Value("${naver.clientSecret}")
    private String clientSecret;

    public ShoppingResponse search(ShoppingRequest request) {
        URI uri = UriComponentsBuilder.fromUriString(naverUrl)
                .queryParams(request.map())
                .build()
                .encode()
                .toUri();

        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Naver-Client-Id", clientId);
        headers.set("X-Naver-Client-Secret", clientSecret);

        HttpEntity httpEntity = new HttpEntity<>(headers);

        ResponseEntity<ShoppingResponse> entity = new RestTemplate().exchange(
                uri,
                HttpMethod.GET,
                httpEntity,
                ShoppingResponse.class
        );

        return entity.getBody();
    }
}
```

## Service 및 Controller 설정

#### - ShoppingService

```java
package pjh5365.navershoppingapi.service;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import pjh5365.navershoppingapi.dto.ShoppingRequest;
import pjh5365.navershoppingapi.dto.ShoppingResponse;
import pjh5365.navershoppingapi.shopping.NaverClient;

@Service
@RequiredArgsConstructor
public class ShoppingService {
    private final NaverClient naverClient;

    public ShoppingResponse search(String query) {
        ShoppingRequest shoppingRequest = new ShoppingRequest();
        shoppingRequest.setQuery(query);

        return naverClient.search(shoppingRequest);
    }
}
```

#### - ApiController

```java
package pjh5365.navershoppingapi.controller;

import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import pjh5365.navershoppingapi.dto.ShoppingResponse;
import pjh5365.navershoppingapi.service.ShoppingService;

@RestController
@RequestMapping("/api/shopping")
@RequiredArgsConstructor    //final 필드 자동주입
public class ApiController {

    private final ShoppingService shoppingService;

    @GetMapping("/search/{query}")
    public ShoppingResponse search(@PathVariable String query) {
        return shoppingService.search(query);
    }
}
```

## Test

슬리퍼를 검색하는 결과가 정상적으로 나오는지 테스트를 해 보았다.

#### - NaverClientTest

```java
package pjh5365.navershoppingapi.shopping;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import pjh5365.navershoppingapi.dto.ShoppingRequest;

@SpringBootTest
public class NaverClientTest {

    @Autowired
    private NaverClient naverClient;

    @Test
    public void searchTest() {
        var search = new ShoppingRequest();
        search.setQuery("슬리퍼");

        var result = naverClient.search(search);
        System.out.println(result);
    }
}
```

#### - 실행결과

![실행결과1](/assets/img/2023-07-21-NaverShoppingApi/3.png)

정상적으로 작동하는 것을 확인할 수 있다.

## 코드 실행

전체적인 프로젝트를 실행하고 `Talend API` 로 테스트해보았다. 검색어로는 과자를 넣었다.

![실행결과2](/assets/img/2023-07-21-NaverShoppingApi/4.png)

값이 정상적으로 출력이 되는 것을 확인할 수 있다. 여기서 더 나아간다면 프론트까지 만들어서 같은 제품을 여러개 나열하여 어디가 최저가인지 확인할 수 있는 서비스를 만들 수 있을 것 이다.