---
title: 자바 동시성 문제 해결하기 -2 (ReentrantLock)
author: pjh5365
date: 2025-06-11 21:08:00 +0900
categories: [Language, JAVA]
tags: [java]
image:
  path: /assets/img/java-concurrency.png
  alt: Java Concurrency
---

[이전 게시글과 이어집니다.](https://pjh5365.github.io/posts/%EC%9E%90%EB%B0%94-%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0-1-%28synchronized%29/)

이번에는 저번 게시글에서 소개한 `synchronized` 의 한계를 보완한 `ReentrantLock` 을 사용하여 동시성 문제를 해결해 보겠습니다.

## ReentrantLock

우선 `ReentrantLock` 에 대해 알아보겠습니다. `ReentrantLock` 은 자바 1.5 버전부터 추가된 클래스이며, `synchronized` 보다 더 많은 제어권을 제공합니다.

### 특징

- 락을 **명시적으로 획득/해제**할 수 있습니다. **&rarr;** `synchronized` 키워드와 달리 코드로 직접 락을 획득하고 반납하기 때문에 명시적입니다.

- **인터럽트 대응**이 가능합니다. **&rarr;** 락을 획득하기 위해 대기 중 인터럽트가 발생하면 락 획득을 포기합니다.

- **tryLock()** 으로 락 획득 실패 로직 작성이 가능합니다. **&rarr;** 락 획득을 시도하고 실패한다면 조건문을 사용해 실패 로직을 작성할 수 있습니다.

- **공정성(Fairness)** 설정이 가능합니다. **&rarr;** 락 객체 생성 시 옵션을 주어 락을 요청한 순서대로 락을 획득하도록 설정이 가능합니다.

> 추가적으로, `ReentrantLock` 이 사용하는 락은 객체 내부에 있는 모니터 락이 아닙니다. 모니터락은 `synchronized` 에서만 사용됩니다.
{: .prompt-tip }

## 예제 코드

[전체 코드](https://github.com/pjh5365/Java-Concurrency)

저번 게시글에서 `synchronized` 를 통해 동시성을 해결했던 코드를 발전시켜 `ReentrantLock` 을 사용한 코드로 변경해 보겠습니다.

### ReentrantLock 을 사용한 문제 해결 1

```java
/**
 * 계좌 클래스 (ReentrantLock 사용)
 */
public class AccountV3 {

    private int balance;
    private ReentrantLock lock = new ReentrantLock();

    public AccountV3() {
        this.balance = 0;
    }

    /**
     * 잔액 조회 메서드
     * @return 처리 결과
     */
    public int getBalance() {
        String name = Thread.currentThread().getName();
        lock.lock();
        try {
            System.out.println("[" + name + "] 잔액을 조회합니다. 잔액: " + this.balance);
            return balance;
        } finally {
            lock.unlock();
        }
    }

    /**
     * 입금 메서드
     * @param amount 입금할 금액
     * @return 처리 결과
     */
    public boolean deposit(int amount) {
        String name = Thread.currentThread().getName();
        lock.lock();
        try {
            System.out.println("[" + name + "] 입금을 시도합니다. 입금 전 잔액: " + this.balance);
            Thread.sleep(1000L);
            this.balance += amount;
            System.out.println("[" + name + "] 입금이 완료되었습니다. 입금 후 잔액: " + this.balance);
            return true;
        } catch (InterruptedException e) {
            System.out.println("[" + name + "] 입금에 실패했습니다");
            return false;
        } finally {
            lock.unlock();
        }
    }

    /**
     * 출금 메서드
     * @param amount 출금할 금액
     * @return 처리 결과
     */
    public boolean withdraw(int amount) {
        String name = Thread.currentThread().getName();
        lock.lock();
        try {
            System.out.println("[" + name + "] 출금을 시도합니다. 출금 전 잔액: " + this.balance);
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
        } finally {
            lock.unlock();
        }
    }
}
```

> `finally` 블록에서 `unlock()` 을 **반드시 호출해야 락 해제를 보장**할 수 있습니다. 호출 누락 시 락을 반납하지 않아 **무한 대기에 빠질 수 있습니다.**
{: .prompt-info }

`lock()`, `unlock()` 메서드를 사용하여 임계 영역에 들어가기 전에 락을 획득하고, 임계 영역이 끝나면 락을 반환합니다.

이 코드를 멀티 스레드 환경에서 실행해 보겠습니다.

![](/assets/img/2025-06-11-cuncurrency/1.png)

![](/assets/img/2025-06-11-cuncurrency/2.png)

락을 이용하여 각 메서드를 한 번에 한 스레드에서만 실행되게 하여 동시성 문제를 해결하였습니다.

하지만, 이 코드 역시 `synchronized` 와 똑같이 무한 대기 문제가 발생할 가능성이 존재합니다. 

### ReentrantLock 을 사용한 문제 해결 2

이번에는 락 획득을 시도하고 실패하면 이전과 같이 대기하는 것이 아니라 바로 포기하고 다음 로직으로 넘어가도록 작성해 보겠습니다.

```java
/**
 * 계좌 클래스 (ReentrantLock 로 무한 대기 해결)
 */
public class AccountV4 {

    private int balance;
    private ReentrantLock lock = new ReentrantLock();

    public AccountV4() {
        this.balance = 0;
    }

    /**
     * 잔액 조회 메서드
     * @return 처리 결과
     */
    public int getBalance() {
        String name = Thread.currentThread().getName();
        lock.lock();
        try {
            System.out.println("[" + name + "] 잔액을 조회합니다. 잔액: " + this.balance);
            return balance;
        } finally {
            lock.unlock();
        }
    }

    /**
     * 입금 메서드
     * @param amount 입금할 금액
     * @return 처리 결과
     */
    public boolean deposit(int amount) {
        String name = Thread.currentThread().getName();
        if (!lock.tryLock()) { // 락 확보를 시도하고, 실패하면 반환한다.
            System.out.println("[" + name + "] **실패** 이미 다른 스레드에서 작업입니다... ");
            return false;
        }
        try {
            System.out.println("[" + name + "] 입금을 시도합니다. 입금 전 잔액: " + this.balance);
            Thread.sleep(1000L);
            this.balance += amount;
            System.out.println("[" + name + "] 입금이 완료되었습니다. 입금 후 잔액: " + this.balance);
            return true;
        } catch (InterruptedException e) {
            System.out.println("[" + name + "] 입금에 실패했습니다");
            return false;
        } finally {
            lock.unlock();
        }
    }

    /**
     * 출금 메서드
     * @param amount 출금할 금액
     * @return 처리 결과
     */
    public boolean withdraw(int amount) {
        String name = Thread.currentThread().getName();
        if (!lock.tryLock()) { // 락 확보를 시도하고, 실패하면 반환한다.
            System.out.println("[" + name + "] **실패** 이미 다른 스레드에서 작업입니다... ");
            return false;
        }
        try {
            System.out.println("[" + name + "] 출금을 시도합니다. 출금 전 잔액: " + this.balance);
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
        } finally {
            lock.unlock();
        }
    }
}
```

이 코드를 멀티 스레드 환경에서 실행해 보겠습니다.

![](/assets/img/2025-06-11-cuncurrency/3.png)

![](/assets/img/2025-06-11-cuncurrency/4.png)

락을 획득한 스레드에서만 입금을 시도하고, 락을 획득하지 못한 스레드는 바로 실패하여 다음 로직을 수행합니다.

## 정리

이처럼 `ReentrantLock` 은 `synchronized` 보다 유연한 락 제어를 가능하게 하며, 동시성 제어가 필요한 복잡한 시스템에서 매우 유용합니다. 하지만 단순한 경우에는 오히려 복잡성만 증가할 수 있으므로 `synchronized` 가 더 간결하고 좋은 선택일 수 있습니다.
