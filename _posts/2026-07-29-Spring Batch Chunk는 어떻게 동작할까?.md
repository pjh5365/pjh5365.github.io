---
title: Spring Batch Chunk는 어떻게 동작할까?
author: pjh5365
date: 2026-7-29 21:22:00 +0900
categories: [BackEnd, Spring Batch]
tags: [backend, spring, batch]
image:
  path: /assets/img/spring-batch-logo.png
  alt: Spring Batch
---


이전 글에서는 Spring Batch가 왜 필요한지와 Job, Step, Chunk, Tasklet의 기본 구조를 알아보았다.

Spring Batch에서 대량 데이터를 처리할 때 가장 자주 사용하는 방식은 Chunk 기반 처리다.

```java
.chunk(1000, transactionManager)
```

설정 자체는 간단하다.

하지만 실제로 사용하다 보면 여러 가지 궁금한 점이 생긴다.

> Reader는 Chunk Size만큼 데이터를 한 번에 조회하는 것일까?

> Reader와 Processor가 `null`을 반환하면 배치가 종료되는 것일까?

> Writer도 데이터를 한 건씩 처리하는 것일까?

> 외부 API에서 List로 데이터를 받았다면 왜 Iterator가 필요할까?

이번 글에서는 Spring Batch의 Chunk가 내부적으로 어떻게 만들어지는지와 Reader, Processor, Writer에서 `null`이 어떤 의미를 가지는지 알아보자.

## Chunk 기본 동작

Spring Batch의 Chunk 구조는 데이터를 일정 단위로 모아서 처리하고, 트랜잭션을 커밋하는 방식이다.

예를 들어 다음과 같이 설정했다고 가정하자.

```java
.chunk(1000, transactionManager)
```

Spring Batch는 Reader의 `read()`를 반복해서 호출한다.

```text
read() → Item 1건 반환
read() → Item 1건 반환
read() → Item 1건 반환
...
read() → Item 1000번째 반환
```

Reader가 반환한 Item이 1000건 모이면 Processor와 Writer를 거쳐 트랜잭션을 커밋한다.

전체 흐름은 다음과 비슷하다.

```text
Reader에서 최대 1000건 읽기
        ↓
Processor에서 각 Item 처리
        ↓
처리 결과를 출력 Chunk에 저장
        ↓
Writer에 Chunk 전달
        ↓
트랜잭션 Commit
        ↓
다음 Chunk 처리
```

여기서 중요한 점은 다음과 같다.

> Chunk Size가 1000이라는 것은 Reader가 DB를 한 번 호출하여 반드시 1000건을 조회한다는 의미가 아니다.

Chunk Size는 Spring Batch가 입력 Item을 몇 건 단위로 처리하고 커밋할지 정하는 값이다.

Reader가 실제 데이터를 어떻게 조회하는지는 Cursor Reader, Paging Reader, 파일 Reader, 커스텀 Reader 등 Reader의 구현 방식에 따라 달라진다.

### Chunk Size의 정확한 의미

Chunk Size는 흔히 Writer가 한 번 실행되는 기준 건수라고 설명한다.

큰 흐름에서는 맞지만, 조금 더 정확하게는 **commit interval**이라고 이해해야 한다.

```java
.chunk(1000, transactionManager)
```

위 설정은 다음 의미에 가깝다.

```text
Reader에서 입력 Item을 최대 1000건 읽는다.
Processor를 통해 출력 Item을 만든다.
출력 Item을 Writer에 전달한다.
Writer 실행이 완료되면 트랜잭션을 Commit한다.
```

Processor에서 모든 데이터가 그대로 통과한다면 다음과 같이 동작한다.

```text
Reader가 읽은 데이터: 1000건
Processor가 반환한 데이터: 1000건
Writer가 받은 데이터: 1000건
```

하지만 Processor에서 100건을 필터링했다면 결과가 달라진다.

```text
Reader가 읽은 데이터: 1000건
Processor에서 null 반환: 100건
Writer가 받은 데이터: 900건
```

Writer가 900건만 받았다고 해서 Spring Batch가 추가로 100건을 더 읽어 Writer에 1000건을 채우지는 않는다.

```text
readCount:   1000
filterCount: 100
writeCount:  900
commitCount: 1
```

즉, Chunk Size는 Writer에 반드시 전달되는 출력 데이터의 개수가 아니라 **한 번의 Chunk 트랜잭션에서 읽는 입력 Item의 기준 개수**다.

## Reader 동작 방식

Reader는 데이터를 읽어 Spring Batch에 한 건씩 전달하는 역할을 한다.

```java
public interface ItemReader<T> {

    T read() throws Exception;
}
```

Spring Batch는 Chunk Size만큼 데이터를 모으기 위해 `read()`를 반복 호출한다.

```text
read() 1회 호출
→ Item 1건 반환
```

예를 들어 데이터가 다음과 같이 있다면,

```text
User1
User2
User3
```

Reader는 다음과 같이 호출된다.

```text
read() → User1
read() → User2
read() → User3
read() → null
```

마지막에 반환되는 `null`은 일반 데이터가 아니라 특별한 의미를 가진다.

### Reader의 `null`은 데이터의 끝이다

Reader가 `null`을 반환하면 Spring Batch는 다음과 같이 판단한다.

> 더 이상 읽을 데이터가 없다.

즉, Reader의 `null`은 현재 Item을 건너뛴다는 뜻이 아니다.

전체 입력 데이터가 끝났다는 의미다.

```java
@Override
public Item read() {

    if (더_이상_데이터가_없음) {
        return null;
    }

    return item;
}
```

Reader가 `null`을 반환했다고 해서 무조건 즉시 모든 처리가 종료되는 것은 아니다.

이미 현재 Chunk에 읽어 둔 데이터가 있다면 해당 데이터는 버리지 않고 Processor와 Writer에 전달한다.

예를 들어 Chunk Size가 1000이고 실제 데이터가 700건이라면 다음과 같이 동작한다.

```text
read() 700회 수행
read() → null

Processor 700건 처리
Writer에 최대 700건 전달
Commit
Step 종료
```

데이터가 2350건이라면 Writer는 일반적으로 다음 단위로 호출된다.

```text
첫 번째 Chunk  → 1000건
두 번째 Chunk  → 1000건
세 번째 Chunk  → 350건
```

마지막 Chunk가 Chunk Size보다 작더라도 정상적으로 처리된다.

### Reader에서 특정 Item을 제외하면 안 될까?

다음과 같은 코드를 작성했다고 가정하자.

```java
@Override
public User read() {

    User user = getNextUser();

    if (!user.isActive()) {
        return null;
    }

    return user;
}
```

개발자의 의도는 비활성 사용자를 한 명만 제외하는 것일 수 있다.

하지만 Spring Batch는 다르게 해석한다.

```text
User1 → 반환
User2 → 반환
User3 → null
```

User3에서 `null`이 반환되는 순간 Spring Batch는 데이터가 모두 끝났다고 판단한다.

```text
User4
User5
User6
...
```

이후 데이터는 더 이상 읽지 않는다.

따라서 Reader에서 `null`은 특정 Item을 제외하는 용도로 사용하면 안 된다.

특정 Item만 제외하려면 Processor에서 `null`을 반환해야 한다.

## Processor 동작 방식

Processor는 Reader가 반환한 데이터를 한 건씩 받아 가공하거나 검증하는 역할을 한다.

```java
public interface ItemProcessor<I, O> {

    O process(I item) throws Exception;
}
```

Processor에서는 보통 다음과 같은 작업을 수행한다.

```text
데이터 검증
DTO 또는 Entity 변환
값 계산
비즈니스 규칙 적용
저장 대상 여부 판단
```

예를 들어 외부 시스템의 사용자 데이터를 내부 저장 객체로 변경할 수 있다.

```java
@Override
public UserEntity process(ExternalUser item) {

    return UserEntity.builder()
            .userId(item.getUserId())
            .userName(item.getUserName())
            .build();
}
```

Reader와 달리 Processor의 `null`은 데이터의 끝을 의미하지 않는다.

### Processor의 `null`은 현재 Item 필터링이다

Processor가 `null`을 반환하면 Spring Batch는 해당 Item을 Writer에 전달하지 않는다.

```java
@Override
public UserEntity process(ExternalUser item) {

    if (!item.isActive()) {
        return null;
    }

    return convert(item);
}
```

처리 흐름은 다음과 같다.

```text
Reader → User1
Processor → UserEntity1
Writer 전달 대상에 추가

Reader → User2
Processor → null
Writer 전달 대상에서 제외

Reader → User3
Processor → UserEntity3
Writer 전달 대상에 추가
```

Writer가 받는 결과는 다음과 같다.

```text
[UserEntity1, UserEntity3]
```

User2만 제외되며 User3 이후의 데이터는 계속 처리된다.

정리하면 Reader와 Processor의 `null`은 완전히 다른 의미다.

```text
Reader null
→ 전체 입력 데이터 종료

Processor null
→ 현재 Item만 필터링
```

### Filter와 Skip은 다르다

Processor에서 `null`을 반환하는 것은 Spring Batch에서 Filter로 처리된다.

```java
if (!item.isTarget()) {
    return null;
}
```

이 데이터는 오류 데이터가 아니라 이번 배치의 처리 대상이 아닌 정상 데이터다.

```text
readCount 증가
filterCount 증가
skipCount 증가하지 않음
```

반면 처리 중 예외가 발생했고 Fault Tolerant 설정을 통해 해당 데이터를 건너뛰는 것은 Skip이다.

```java
@Override
public UserEntity process(ExternalUser item) {

    if (item.getUserId() == null) {
        throw new InvalidUserException();
    }

    return convert(item);
}
```

```java
new StepBuilder("userStep", jobRepository)
        .<ExternalUser, UserEntity>chunk(1000, transactionManager)
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .faultTolerant()
        .skip(InvalidUserException.class)
        .skipLimit(100)
        .build();
```

두 개념의 차이는 다음과 같다.

```text
Filter
→ 정상적인 조건 판단
→ 처리 대상이 아니어서 제외
→ Processor에서 null 반환

Skip
→ 처리 중 오류 발생
→ 설정된 예외에 한해 건너뜀
→ Skip Count 증가
```

처리 대상이 아니라면 Filter를 사용하고, 잘못된 데이터나 처리 실패를 허용해야 한다면 Skip을 사용하는 것이 자연스럽다.

## Writer 동작 방식

Writer는 Processor를 통과한 결과를 Chunk 단위로 받아 최종 저장하거나 외부 시스템에 반영한다.

```java
public interface ItemWriter<T> {

    void write(Chunk<? extends T> chunk) throws Exception;
}
```

Processor는 Item 하나마다 호출되지만 Writer는 출력 Chunk 단위로 호출된다.

```text
Reader 입력 Item 1000건
        ↓
Processor 1000회 호출
        ↓
출력 Item을 Chunk로 구성
        ↓
Writer 1회 호출
```

Writer에서는 다음과 같은 작업을 수행할 수 있다.

```text
DB Insert 또는 Update
파일 쓰기
메시지 발행
외부 API 요청
메일 발송
```

구현 예시는 다음과 같다.

```java
@Override
public void write(Chunk<? extends UserEntity> chunk) {

    List<? extends UserEntity> items = chunk.getItems();

    userRepository.saveAll(items);
}
```

### Writer에는 `null` 반환이 없다

Reader와 Processor는 객체 또는 `null`을 반환한다.

하지만 Writer의 반환형은 `void`다.

```java
void write(Chunk<? extends T> chunk) throws Exception;
```

따라서 Writer에는 `null`을 반환한다는 개념 자체가 없다.

Writer의 실행 결과는 크게 두 가지다.

```text
예외 없이 write() 종료
→ 정상 처리

write()에서 예외 발생
→ 쓰기 실패
```

다음과 같이 `return;`을 작성하는 것은 가능하다.

```java
@Override
public void write(Chunk<? extends UserEntity> chunk) {

    if (chunk.isEmpty()) {
        return;
    }

    userRepository.saveAll(chunk.getItems());
}
```

하지만 `return;`은 `null`을 반환하는 것이 아니다.

단순히 현재 Writer 메서드를 예외 없이 정상 종료하는 것이다.

```java
@Override
public void write(Chunk<? extends UserEntity> chunk) {
    return;
}
```

위와 같이 아무 작업도 하지 않고 종료해도 Spring Batch는 예외가 발생하지 않았기 때문에 Writer가 정상적으로 완료됐다고 판단한다.

```text
Writer 호출
→ 아무것도 저장하지 않음
→ 예외 없음
→ 정상 완료
→ Commit
→ 다음 Chunk 진행
```

따라서 Writer의 `return;`을 특정 데이터를 필터링하는 용도로 사용해서는 안 된다.

필터링은 Processor에서 처리하는 것이 적절하다.

### Writer에서 예외가 발생하면 어떻게 될까?

일반적인 DB 트랜잭션 기반 Writer에서 예외가 발생하면 현재 Chunk가 롤백되고 Step이 실패한다.

```text
첫 번째 Chunk 1000건 → Commit
두 번째 Chunk 1000건 → Commit
세 번째 Chunk 1000건 → Writer 예외
```

결과는 다음과 같다.

```text
첫 번째 Chunk → 이미 Commit되어 유지
두 번째 Chunk → 이미 Commit되어 유지
세 번째 Chunk → Rollback
Step → FAILED
```

단, 이 설명은 DB처럼 Spring 트랜잭션에 참여하는 자원을 기준으로 한다.

Writer에서 외부 API를 호출하거나 메시지를 발행했다면 DB 트랜잭션이 롤백돼도 이미 전송된 요청까지 자동으로 취소되지는 않는다.

따라서 외부 시스템을 호출하는 Writer에서는 다음과 같은 부분을 추가로 고려해야 한다.

```text
중복 요청 방지
멱등성 키 사용
처리 여부 저장
실패 재시도
보상 처리
```

## Processor가 전부 `null`을 반환하면?

Chunk Size가 1000이고 Processor가 1000건 모두 `null`을 반환할 수도 있다.

```text
Reader에서 읽은 데이터: 1000건
Processor 필터링: 1000건
Writer 전달 대상: 0건
```

이 경우 Writer가 빈 Chunk를 받을 가능성이 있으므로 Writer는 빈 Chunk도 안전하게 처리할 수 있도록 구현하는 것이 좋다.

```java
@Override
public void write(Chunk<? extends UserEntity> chunk) {

    if (chunk.isEmpty()) {
        return;
    }

    userRepository.saveAll(chunk.getItems());
}
```

## 전체 흐름으로 살펴보기

Chunk Size가 5이고 Reader가 다음 데이터를 반환한다고 가정하자.

```text
User1
User2
User3
User4
User5
```

이 중 User2와 User4는 Processor에서 제외한다.

```text
read() → User1
read() → User2
read() → User3
read() → User4
read() → User5
```

Processor는 각 Item을 처리한다.

```text
User1 → UserEntity1
User2 → null
User3 → UserEntity3
User4 → null
User5 → UserEntity5
```

Writer가 받는 Chunk는 다음과 같다.

```text
[
    UserEntity1,
    UserEntity3,
    UserEntity5
]
```

처리 결과는 다음과 같다.

```text
readCount:   5
filterCount: 2
writeCount:  3
commitCount: 1
```

Chunk Size가 5라고 해서 Writer가 반드시 5건을 받는 것은 아니다.

## Reader는 `read()`마다 DB를 다시 조회할까?

Reader가 `read()`마다 Item 하나를 반환한다고 하면 다음과 같은 의문이 생긴다.

> Chunk Size가 1000이면 DB 쿼리도 1000번 실행되는 것일까?

그렇지는 않다.

실제 DB 조회 방식은 사용하는 Reader 구현체에 따라 달라진다.

대표적으로 Cursor 방식과 Paging 방식이 있다.


### Cursor Reader

Cursor Reader는 일반적으로 SELECT 쿼리를 실행한 뒤 DB Cursor와 JDBC ResultSet을 열어 둔다.

```text
SELECT 실행
    ↓
DB Cursor 및 ResultSet 유지
    ↓
read() → 다음 행
read() → 다음 행
read() → 다음 행
...
```

`read()`가 호출될 때마다 SELECT 쿼리를 다시 실행하는 것이 아니라 열린 ResultSet의 다음 행으로 이동한다.

```text
read()
→ resultSet.next()
→ 현재 행을 Item으로 변환
→ 반환
```

총 데이터가 10000건이고 Chunk Size가 1000이라면 개념적으로 다음과 같이 동작한다.

```text
SELECT 실행: 일반적으로 1회

1000건 읽기 → Writer → Commit
1000건 읽기 → Writer → Commit
...
1000건 읽기 → Writer → Commit
```

Writer와 Commit은 10번 실행되지만 조회 Cursor는 Reader가 종료될 때까지 유지된다.

다만 SELECT가 한 번 실행된다는 것이 DB와 네트워크 통신도 반드시 한 번만 발생한다는 의미는 아니다.

JDBC 드라이버는 Fetch Size 등에 따라 ResultSet 데이터를 여러 묶음으로 가져올 수 있다.

```text
DB → JDBC 버퍼 100건
read() 100회

DB → JDBC 버퍼 다음 100건
read() 100회
```

### Paging Reader

Paging Reader는 데이터를 일정 페이지 단위로 조회한다.

```text
첫 번째 페이지 SELECT
→ Reader 내부에 페이지 결과 저장
→ read()로 한 건씩 반환

두 번째 페이지 SELECT
→ Reader 내부 결과 교체
→ read()로 한 건씩 반환
```

총 데이터가 10000건이고 Page Size가 1000이라면 데이터를 가져오기 위한 페이지 조회는 약 10번 발생한다.

```text
1페이지 → 1~1000
2페이지 → 1001~2000
...
10페이지 → 9001~10000
```

데이터가 Page Size의 정확한 배수라면 데이터가 끝났는지 확인하기 위한 빈 페이지 조회가 한 번 더 발생할 수도 있다.

여기서도 Chunk Size와 Page Size는 서로 다른 설정이다.

```text
Chunk Size
→ 몇 건을 읽고 처리한 뒤 Commit할지 결정

Page Size
→ DB에서 한 번에 몇 건씩 조회할지 결정
```

보통 Chunk Size와 Page Size를 같게 설정하면 이해하기 쉽지만, 반드시 같은 값이어야 하는 것은 아니다.

## Reader 버퍼와 Chunk는 다르다

Reader가 데이터를 미리 읽어 내부에 저장하는 경우가 있다.

Paging Reader를 예로 들면 다음과 같다.

```text
DB
 ↓ 페이지 단위 조회
Reader 내부 results
 ↓ read()로 한 건 반환
Spring Batch 입력 Chunk
 ↓ Processor 처리
출력 Chunk
 ↓
Writer
```

여기에는 서로 다른 데이터 저장 공간이 존재한다.

```text
Reader 내부 버퍼
→ Reader가 조회한 페이지 데이터를 임시 저장

Spring Batch Chunk
→ Reader가 반환한 Item을 트랜잭션 단위로 저장
```

Cursor Reader는 Paging Reader처럼 명확한 페이지 List를 보관하지 않지만, JDBC 드라이버가 내부 Fetch 버퍼를 사용할 수 있다.

즉, 다음 두 가지는 같은 개념이 아니다.

```text
DB 또는 Reader가 사용하는 조회 버퍼
Spring Batch가 관리하는 Chunk
```

## Iterator를 사용하는 이유

외부 API나 직접 작성한 DB 조회 메서드에서는 조회 결과가 List 형태로 반환되는 경우가 많다.

```java
List<Item> resultList = getNextPage();
```

하지만 Spring Batch Reader는 `read()` 한 번에 Item 하나를 반환해야 한다.

```java
Item read();
```

그래서 조회한 List를 Iterator로 변환한 뒤 `read()`가 호출될 때마다 하나씩 꺼내 반환할 수 있다.

```java
private Iterator<Item> currentIterator = Collections.emptyIterator();

@Override
public Item read() {

    if (currentIterator.hasNext()) {
        return currentIterator.next();
    }

    List<Item> resultList = getNextPage();

    if (resultList == null || resultList.isEmpty()) {
        return null;
    }

    currentIterator = resultList.iterator();

    return currentIterator.next();
}
```

처리 흐름은 다음과 같다.

```text
getNextPage() 호출
→ List<Item> 조회

List를 Iterator로 변환
→ currentIterator에 저장

read() 호출
→ currentIterator.next()
→ Item 한 건 반환
```

즉, Iterator는 한 번 조회한 List 데이터를 Spring Batch에 한 건씩 전달하기 위한 장치다.

## Iterator Reader에서 주의할 점

위 코드에서 `getNextPage()`는 호출될 때마다 반드시 다음 데이터를 조회해야 한다.

예를 들어 현재 페이지 번호를 증가시키지 않고 항상 같은 데이터를 조회하면 다음과 같은 문제가 발생한다.

```text
첫 번째 List 조회
→ 모든 데이터 반환

Iterator 소진
→ 같은 List 다시 조회

같은 데이터 다시 반환
→ 무한 반복
```

따라서 페이지 기반 API를 호출한다면 페이지 상태를 관리해야 한다.

```java
private int page = 0;
private Iterator<Item> currentIterator = Collections.emptyIterator();

@Override
public Item read() {

    if (currentIterator.hasNext()) {
        return currentIterator.next();
    }

    List<Item> resultList = getPage(page);

    if (resultList == null || resultList.isEmpty()) {
        return null;
    }

    page++;
    currentIterator = resultList.iterator();

    return currentIterator.next();
}
```

하지만 이것만으로는 Job 재시작까지 지원되지 않는다.

`page` 값은 애플리케이션 메모리에만 있기 때문에 Step이 실패한 후 재시작하면 다시 0부터 시작할 수 있다.

재시작을 지원해야 한다면 다음과 같은 방법이 필요하다.

```text
ItemStream 구현
ExecutionContext에 현재 페이지 저장
ExecutionContext에서 페이지 복원
기존 PagingItemReader 사용
```

가능하다면 직접 Reader를 구현하기보다 Spring Batch에서 제공하는 Cursor 또는 Paging Reader를 먼저 검토하는 것이 좋다.

## `return read()`와 직접 반환의 차이

다음과 같이 새로운 List를 Iterator에 저장한 뒤 `read()`를 다시 호출하는 코드도 작성할 수 있다.

```java
currentIterator = resultList.iterator();

return read();
```

이 코드도 현재 List가 비어 있지 않다면 정상적으로 동작할 수 있다.

하지만 `read()` 메서드 내부에서 다시 `read()`를 호출하는 재귀 구조가 된다.

```text
첫 번째 read() 진입
→ 데이터 조회
→ Iterator 설정
→ 다시 read() 호출
→ Iterator에서 Item 반환
```

조금 더 명확한 방식은 현재 Iterator에서 직접 Item을 꺼내는 것이다.

```java
currentIterator = resultList.iterator();

return currentIterator.next();
```

두 방식의 차이는 다음과 같다.

```text
return read()

- read() 메서드에 다시 진입한다.
- 상태 변경과 반환 흐름이 분리된다.
- 조건이 추가되면 흐름 추적이 어려워질 수 있다.
- 잘못 구현하면 재귀 호출이 반복될 수 있다.

직접 반환

- 현재 실행 흐름에서 Item을 반환한다.
- Iterator가 변경되는 위치가 명확하다.
- 디버깅이 상대적으로 쉽다.
```

현재 구조처럼 조회 결과가 비어 있으면 종료하고, 데이터가 있으면 첫 번째 Item을 반환하는 경우라면 직접 `next()`를 호출하는 편이 더 단순하다.

## Chunk를 사용하면 메모리 사용량이 항상 일정할까?

Chunk를 사용하면 Spring Batch가 한 번의 트랜잭션에서 관리하는 데이터의 양을 제한할 수 있다.

하지만 Chunk를 사용한다는 이유만으로 전체 메모리 사용량이 항상 제한되는 것은 아니다.

예를 들어 Reader 내부에서 다음과 같이 전체 데이터를 한 번에 조회하면,

```java
List<User> users = userRepository.findAll();
```

이미 모든 데이터가 메모리에 올라간 상태다.

그 이후 Iterator를 통해 한 건씩 반환하더라도 전체 List는 메모리에 남아 있다.

```text
DB 전체 데이터 조회
→ 100만 건 List 메모리 적재
→ Iterator로 한 건씩 반환
→ Spring Batch는 1000건씩 Chunk 처리
```

이 경우 Chunk 메모리는 1000건 단위로 관리되지만 Reader가 가진 List에는 100만 건이 들어 있을 수 있다.

대량 데이터를 안정적으로 처리하려면 Chunk Size뿐만 아니라 Reader의 조회 방식도 함께 확인해야 한다.

```text
Cursor Reader
Paging Reader
Streaming 방식
API 페이지 조회
파일 스트리밍
```

## Writer가 Chunk를 받으면 Batch Insert가 자동으로 적용될까?

Writer는 Chunk 전체를 전달받는다.

```java
@Override
public void write(Chunk<? extends User> chunk) {
    userRepository.saveAll(chunk.getItems());
}
```

하지만 Chunk를 전달받는다는 이유만으로 실제 SQL이 자동으로 하나의 Batch Insert로 실행되는 것은 아니다.

실제 Batch Insert 여부는 사용하는 Writer와 DB 접근 기술의 설정에 따라 달라진다.

```text
JdbcBatchItemWriter 사용 여부
JDBC Batch API 사용 여부
Hibernate jdbc.batch_size 설정
ID 생성 전략
MyBatis Batch Executor 설정
Driver의 Batch 지원 여부
```

따라서 다음 두 문장은 구분해야 한다.

```text
Writer는 Chunk 단위로 호출된다.
→ Spring Batch의 기본 동작

DB Batch Insert가 실행된다.
→ Writer 구현과 DB 설정에 따라 결정
```

## 자주 헷갈리는 내용 정리

### Chunk Size가 1000이면 DB를 1000건씩 조회한다?

아니다.

```text
Chunk Size
→ 트랜잭션 및 처리 단위

DB 조회 단위
→ Reader 구현과 Fetch Size, Page Size 등에 따라 결정
```

### Reader가 `null`을 반환하면 현재 Item만 제외된다?

아니다.

```text
Reader null
→ 전체 입력 데이터 종료
```

### Processor가 `null`을 반환하면 Step이 종료된다?

아니다.

```text
Processor null
→ 현재 Item만 Writer 대상에서 제외
```

### Writer가 `null`을 반환하면 현재 Chunk를 건너뛴다?

Writer는 `void`이므로 `null`을 반환할 수 없다.

```text
예외 없이 종료
→ 성공

예외 발생
→ 실패 또는 Fault Tolerant 설정에 따른 처리
```

### Chunk를 사용하면 메모리 문제가 무조건 해결된다?

아니다.

Reader가 전체 데이터를 먼저 메모리에 올리면 Chunk를 사용해도 전체 데이터는 메모리에 남아 있을 수 있다.

### Writer가 Chunk를 받으면 DB Batch Insert가 자동 적용된다?

아니다.

실제 Batch Insert 여부는 Writer 구현과 DB 접근 설정에 따라 달라진다.

## 정리

Spring Batch Chunk 구조의 핵심은 다음과 같다.

- Reader는 `read()` 호출마다 `Item` 하나를 반환한다.
- Reader의 `null`은 더 이상 읽을 데이터가 없다는 의미다.
- Processor는 `Item` 하나씩 가공하거나 검증한다.
- Processor의 `null`은 현재 `Item`을 Writer 대상에서 제외한다는 의미다.
- Writer는 출력 데이터를 `Chunk` 단위로 받는다.
- Writer는 `void`이므로 `null`을 반환하는 개념이 없다.
- `Chunk Size`는 Writer에 반드시 전달되는 출력 건수가 아니라 입력 데이터를 몇 건 단위로 처리하고 Commit할지 정하는 값이다.
- `Cursor`, `Paging`, `Iterator` 등 Reader 구현 방식에 따라 실제 데이터 조회 방식은 달라진다.
- Chunk가 관리하는 버퍼와 Reader 또는 JDBC가 관리하는 버퍼는 서로 다르다.

결국 Chunk 기반 처리는 다음 흐름으로 정리할 수 있다.

```text
Reader에서 입력 Item 수집
        ↓
Processor에서 Item 단위 처리 및 필터링
        ↓
출력 Chunk 생성
        ↓
Writer 호출
        ↓
트랜잭션 Commit
        ↓
다음 Chunk 반복
```

Spring Batch를 제대로 사용하려면 Chunk Size만 설정하는 것이 아니라 Reader가 데이터를 어떻게 가져오는지, Processor에서 어떤 데이터를 필터링하는지, Writer가 실제로 어떤 방식으로 저장하는지까지 함께 이해해야 한다.
