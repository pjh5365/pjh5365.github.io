---
title: 자바 동시성 문제 해결하기 -1 (synchronized)
author: pjh5365
date: 2025-05-22 20:46:00 +0900
categories: [Language, JAVA]
tags: [java]
image:
  path: /assets/img/java-concurrency.png
  alt: Java Concurrency
---
자바의 멀티 스레드 환경에서 자주 발생하는 동시성 문제를 어떻게 해결하는지 단계별로 알아보겠습니다.

## 사전 지식

### 프로그램과 프로세스

우선 **프로그램(Program)**과 **프로세스(Process)**에 대해 알아보겠습니다. 프로그램이란 컴퓨터에서 실행할 수 있는 파일을 의미합니다. 그리고 이 프로그램을 실행시켜 컴퓨터에서 작업 중인 프로그램 상태를 프로세스라고 표현합니다. 

**하지만, 이 글에서는 이해를 돕기 위해 프로그램과 프로세스를 동일한 의미로 사용하겠습니다.**

### 스레드

다음으로, **스레드(Thread)**에 대해 알아보겠습니다. 스레드란 하나의 프로세스 내부에서 실행되는 작은 작업 단위입니다. 예를 들어, 한 애플리케이션에서 사용자로부터 입력을 받거나, 외부로 출력하는 작업을 모두 스레드로 처리되는 작업이라 볼 수 있습니다.

### 단일 스레드

프로세스가 스레드를 하나만 가지고 있을 때 **단일 스레드(Single Thread)**라고 부릅니다. 

### 멀티 스레드

프로세스가 스레드를 여러 개 가지고 있을 때 **멀티 스레드(Multi Thread)**라고 부릅니다.

#### 왜 멀티 스레드가 필요할까?

하나의 스레드만 가지고도 작업이 가능한데 왜 대부분의 프로그램이 멀티 스레드를 사용할까요? 우선 단일 스레드를 사용할 때와 멀티 스레드를 사용할 때를 비교해 보겠습니다.

하나의 프로그램에서 아래와 같은 2개의 작업이 있다고 가정하겠습니다.

- **(작업 1)사용자의 입력이 들어올 때까지 대기하는 작업**
- **(작업 2)숫자를 더하는 작업**

단일 스레드 프로그램에서는 한 번에 하나의 작업만 수행되기 때문에 작업 1이 수행되면 사용자의 입력이 들어올 때까지 작업 2를 수행할 수 없습니다. 

하지만 멀티 스레드 프로그램에서는 한 번에 여러 작업을 수행할 수 있기 때문에 작업 1이 먼저 수행되더라도 다른 스레드에서 작업 2를 **동시에 수행할 수 있기에 성능상 이점**이 존재합니다.
일상생활을 예로 들어보면 식당에서 

- 한 명이 요리, 서빙, 계산까지 수행한다면 매우 느리고 효율적이지 못합니다(단일 스레드). 
- 반면, 요리사, 서빙 직원, 계산원이 각자의 일을 맡는다면 훨씬 빠르고 효율적으로 운영됩니다(멀티 스레드).

> 단, 연산을 담당하는 CPU 코어가 하나인 경우에는 여러 작업을 동시에 수행하는 것이 아니라, 여러 작업을 아주 작은 단위로 쪼개서 **각각의 작업을 번갈아 가며 수행하여 동시에 작업이 하는 것처럼 동작**합니다. 코어가 여러 개라면 각각의 코어에서 서로 다른 작업이 수행되거나 단일 코어처럼 여러 작업을 효율적으로 쪼개 번갈아 가며 수행합니다.
{: .prompt-info }

이처럼 멀티 스레드를 사용하면 훨씬 효율적인 프로그램을 개발할 수 있습니다. 단, 이 멀티 스레드를 사용할 땐 동시성 문제가 발생할 수 있으므로 주의해서 사용해야 합니다.
## 동시성 문제

**동시성 문제(Concurrency Issue)**란 여러 스레드가 동시에 동일한 데이터에 접근할 때 발생하는 문제 전반을 의미합니다. 

### 경쟁 상태

**경쟁 상태(Race condition)**란 여러 스레드가 동시에 동일한 데이터를 수정할 때 순서나 타이밍에 따라 예상한 결과와 다른 값이 나오는 문제로 동시성 문제의 대표적인 원인 중 하나입니다. 

에를 들어, 한 변수에 100이 저장되어 있을 때,

- 스레드 A가 값을 읽고 +10을 계산하여 저장하고, (100을 읽고 +10을 하거나 80을 읽고 +10을 하거나)
- 동시에 스레드 B도 값을 읽고 -20을 계산하여 저장할 때 (100을 읽고 -20을 하거나 110을 읽고 -20을 하거나)

내가 예상한 결과는 90이지만, 순서나 타이밍에 따라 80이 될수도 110이 될수도 있는 상황을 경쟁 상태라고 합니다.

> **경쟁 상태를 제외하고도 동시성 문제의 원인은 데드락(Deadlock), 라이브락(Livelock) 등 여러 가지가 존재합니다.**
{: .prompt-tip }

## synchronized

자바에서는 동시성 문제를 해결하기 위해 `synchronized` 키워드를 제공합니다. 이 키워드는 여러 스레드를 동기화하여 **한 번에 하나의 스레드만 특정 코드 블록이나 메서드에 접근하도록 제한**합니다. 즉, 여러 스레드가 동시에 접근하려 할 때, **하나의 스레드만 진입하고 나머지는 대기**하도록 합니다.

### 내부 동작

자바에서 모든 객체(인스턴스)는 내부에 자신만의 **고유한 락(Intrinsic lock)**을 가지고 있습니다. 이는 **모니터 락(Monitor Lock)**혹은 **모니터(Monitor)**라고 부르기도 합니다.

`synchronized` 를 사용하면 JVM은 해당 객체의 모니터 락을 **확보(lock 획득)한 스레드만 임계 영역을 실행**하도록 하며, **확보하지 못한 스레드는 해당 락이 풀릴 때까지 무한정 대기**합니다.

> **임계 영역(Critical Section)**이란 여러 스레드가 동시에 접근하면 데이터 불일치나 예상치 못한 동작이 발생할 수 있는 부분을 뜻합니다. 은행을 예시로 든다면 **계좌에 접근해 잔고를 조작하는 부분을 임계 영역**이라 볼 수 있습니다.
{: .prompt-info }

## 예제 코드

문제에 집중하기 위해 여러 사용자의 계좌를 관리하는 은행이 아닌 단일 사용자의 계좌를 관리하는 프로그램을 만들며 코드로 알아보겠습니다. 

[전체 코드](https://github.com/pjh5365/Java-Concurrency)

### 동시성 문제가 발생하는 상황

멀티 스레드에 대한 별도의 처리가 없는 일반적인 코드를 작성해 보겠습니다. (여기서 `Thread.sleep()` 메서드는 동시성 문제를 쉽게 관찰하기 위해 넣은 코드로, 실제로는 여러 비즈니스 로직이 수행되는 작업 대신 넣었다고 생각하시면 됩니다.)

```java
/**
 * 계좌 클래스
 */
public class AccountV1 {

    private int balance;

    public AccountV1() {
        this.balance = 0;
    }

    /**
     * 잔액 조회 메서드
     * @return 처리 결과
     */
    public int getBalance() {
        String name = Thread.currentThread().getName();
        System.out.println("[" + name + "] 잔액을 조회합니다. 잔액: " + this.balance);
        return balance;
    }

    /**
     * 입금 메서드
     * @param amount 입금할 금액
     * @return 처리 결과
     */
    public boolean deposit(int amount) {
        String name = Thread.currentThread().getName();
        System.out.println("[" + name + "] 입금을 시도합니다. 입금 전 잔액: " + this.balance);
        try {
            Thread.sleep(1000L);
            this.balance += amount;
            System.out.println("[" + name + "] 입금이 완료되었습니다. 입금 후 잔액: " + this.balance);
            return true;
        } catch (InterruptedException e) {
            System.out.println("[" + name + "] 입금에 실패했습니다");
            return false;
        }
    }

    /**
     * 출금 메서드
     * @param amount 출금할 금액
     * @return 처리 결과
     */
    public boolean withdraw(int amount) {
        String name = Thread.currentThread().getName();
        System.out.println("[" + name + "] 출금을 시도합니다. 출금 전 잔액: " + this.balance);
        try {
            Thread.sleep(2000L);
            if (this.balance < amount) {
                System.out.println("[" + name + "] 잔액보다 많은 양을 출금할 수 없습니다.");
                return false;
            }
            this.balance -= amount;
            System.out.println("[" + name + "] 출금이 완료되었습니다. 출금 후 잔액: " + this.balance);
            return true;
        } catch (InterruptedException e) {
            System.out.println("[" + name + "] 출금에 실패했습니다");
            return false;
        }
    }
}

```
그리고 위의 코드를 멀티 스레드 환경에서 실행해 보겠습니다.

![](/assets/img/2025-05-22-cuncurrency/1.png)

위의 테스트 코드를 실행하면 대부분의 경우 테스트가 통과하지만, 종종 동시성 문제 때문에 실패하는 경우가 발생합니다. 비록 100번 중 99번 성공하더라도 단 한 번이라도 실패가 발생한다면, 해당 코드는 신뢰할 수 없기에 실제 은행에서 사용할 수 없을 것입니다.

![](/assets/img/2025-05-22-cuncurrency/2.png)

### `synchronized` 를 사용한 문제 해결

이제 동기화를 사용해 이 동시성 문제를 해결해 보겠습니다. `synchronized` 키워드를 **임계 영역**인 각 메서드에 붙여 한 번에 한 스레드만 해당 메서드를 실행시키도록 수정하였습니다.

```java
/**
 * 계좌 클래스
 */
public class AccountV2 {

    private int balance;

    public AccountV2() {
        this.balance = 0;
    }

    /**
     * 잔액 조회 메서드
     * @return 처리 결과
     */
    public synchronized int getBalance() {
        String name = Thread.currentThread().getName();
        System.out.println("[" + name + "] 잔액을 조회합니다. 잔액: " + this.balance);
        return balance;
    }

    /**
     * 입금 메서드
     * @param amount 입금할 금액
     * @return 처리 결과
     */
    public synchronized boolean deposit(int amount) {
        String name = Thread.currentThread().getName();
        System.out.println("[" + name + "] 입금을 시도합니다. 입금 전 잔액: " + this.balance);
        try {
            Thread.sleep(1000L);
            this.balance += amount;
            System.out.println("[" + name + "] 입금이 완료되었습니다. 입금 후 잔액: " + this.balance);
            return true;
        } catch (InterruptedException e) {
            System.out.println("[" + name + "] 입금에 실패했습니다");
            return false;
        }
    }

    /**
     * 출금 메서드
     * @param amount 출금할 금액
     * @return 처리 결과
     */
    public synchronized boolean withdraw(int amount) {
        String name = Thread.currentThread().getName();
        System.out.println("[" + name + "] 출금을 시도합니다. 출금 전 잔액: " + this.balance);
        try {
            Thread.sleep(2000L);
            if (this.balance < amount) {
                System.out.println("[" + name + "] 잔액보다 많은 양을 출금할 수 없습니다.");
                return false;
            }
            this.balance -= amount;
            System.out.println("[" + name + "] 출금이 완료되었습니다. 출금 후 잔액: " + this.balance);
            return true;
        } catch (InterruptedException e) {
            System.out.println("[" + name + "] 출금에 실패했습니다");
            return false;
        }
    }
}
```

이제 앞에서 실행했던 테스트 코드를 복사하여 클래스만 변경하여 실행해 보겠습니다.

![](/assets/img/2025-05-22-cuncurrency/3.png)

![](/assets/img/2025-05-22-cuncurrency/4.png)

`synchronized` 키워드를 통해 각 메서드를 한 번에 한 스레드에서만 실행되게 하여 동시성 문제를 해결하였습니다. 반복하여 실행해도 항상 테스트에 성공합니다.

### 한계

이처럼 간단하게 동시성 문제를 막기 위해 `synchronized` 키워드를 사용할 수 있지만, 이 키워드에는 단점이 존재합니다. 

#### 무한 대기

`synchronized` 는 락을 확보하지 못한 스레드는 락이 풀릴 때까지 무한 대기에 들어갑니다. 이때 대기 중인 스레드는 `notify()`, `notifiyAll()` 과 같은 메서드로도 깨울 수 없으며, 오직 락을 확보할 수 있을 때만 실행됩니다.

#### 기아 (Starvation) 현상

또한 대기하고 있는 여러 스레드 중 어떤 스레드가 다음으로 실행될지 제어할 수 없습니다. 따라서 특정 스레드가 지속적으로 락을 확보하지 못하고 기다리는 기아 상태가 발생할 수 있습니다.

위와 같은 한계를 막기 위해 자바는 1.5 버전부터 `ReentrantLock` 을 제공합니다. 다음 게시글에선 `ReentrantLock` 을 사용해서 이러한 한계를 해결해 보겠습니다.
