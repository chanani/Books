---
title: "EP.04 JMM과 락 기반 동기화"
date: "2026-05-29"
episode: 4
tags: [ "JMM", "synchronized", "volatile", "Java", "Concurrency", "Lock" ]
---

지난 세 편을 통해 우리는 스레드가 무엇이고, OS와 JVM에서 어떻게 동작하는지 살펴봤습니다.
이제 본격적인 멀티스레드 프로그래밍의 위험 지대로 들어갈 차례입니다.

여러 스레드가 같은 메모리를 공유한다는 사실이 어떤 문제를 만드는지, Java는 이를 어떻게 해결하는지 알아보겠습니다.🙂

이 글은 Thread Deep Dive 시리즈의 네 번째 편입니다.

## 멀티스레드에서 깨지는 세 가지

다음 코드를 두 개의 스레드가 동시에 실행한다고 해봅시다.

```java
public class Counter {
    private int count = 0;

    public void increment() {
        count++;   // 단순해 보이지만...
    }
}
```

`count++`는 한 줄짜리 코드지만, ByteCode 수준에서는 세 단계입니다.

```
1. count 값을 읽는다             (load)
2. 1을 더한다                     (add)
3. 결과를 다시 저장한다            (store)
```

두 스레드가 동시에 이 세 단계를 실행하면:

```
초기값 count = 0

스레드 A                  스레드 B
─────────                ─────────
1. load 0
                         1. load 0          ← A가 저장하기 전에 읽음
2. add  → 1
                         2. add  → 1
3. store 1
                         3. store 1         ← 덮어씀

결과: count = 1 (기대값은 2)
```

이런 현상을 **race condition(경쟁 상태)** 이라고 합니다. 멀티스레드에서 깨질 수 있는 것은 크게 세 가지입니다.

```
┌─────────────────────────────────────────────────────┐
│               멀티스레드에서 깨지는 세 가지                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ① Atomicity (원자성)                                │
│     하나의 작업이 중간에 끼어들 수 없어야 함                 │
│     → count++ 가 깨지는 이유                           │
│                                                     │
│  ② Visibility (가시성)                               │
│     한 스레드의 변경이 다른 스레드에 즉시 보여야 함            │
│     → CPU 캐시 때문에 안 보이는 경우 있음                  │
│                                                     │
│  ③ Ordering (순서)                                   │
│     코드의 명령어가 작성된 순서대로 실행되어야 함              │
│     → 컴파일러/CPU가 최적화로 순서 바꿈                    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

각각을 살펴보겠습니다.

---

## 가시성(Visibility) 문제 — CPU 캐시

현대 CPU는 메인 메모리 접근이 너무 느려서, 각 코어마다 캐시(L1, L2)를 가지고 있습니다.

```
┌──────────────────────────────────────────────────────┐
│                   메인 메모리 (RAM)                     │
│                  count = 0                           │
└──────────────────────────────────────────────────────┘
        ▲                                ▲
        │                                │
   ┌────┴─────┐                     ┌────┴─────┐
   │  L1/L2   │                     │  L1/L2   │
   │  Cache   │                     │  Cache   │
   │ count=0  │                     │ count=0  │
   └────┬─────┘                     └────┬─────┘
        │                                │
   ┌────┴─────┐                     ┌────┴─────┐
   │  Core 1  │                     │  Core 2  │
   │(Thread A)│                     │(Thread B)│
   └──────────┘                     └──────────┘
```

스레드 A가 `count = 1`로 바꿔도, 그 변경이 **자기 코어의 캐시에만 반영**되고 메인 메모리나 다른 코어에는 즉시 전파되지 않을 수 있습니다.

```java
class StopFlag {
    private boolean stopped = false;

    public void run() {
        while (!stopped) {     // 스레드 A가 무한 반복
            // ...
        }
    }

    public void stop() {       // 스레드 B가 호출
        stopped = true;
    }
}
```

이 코드에서 스레드 B가 `stopped = true`로 바꿔도, 스레드 A는 자기 캐시의 옛날 값(false)을 계속 보면서 영원히 루프를 돕니다. **가시성 보장이 없기 때문**입니다.

---

## 순서(Ordering) 문제 — 명령어 재배치

컴파일러와 CPU는 성능을 위해 코드의 실행 순서를 임의로 바꿉니다.

```java
int a = 1;          // ①
int b = 2;          // ②
int c = a + b;      // ③ — ①과 ②에 의존
```

이 코드에서 ①과 ②는 서로 독립적이라 **순서가 바뀌어 실행되어도 결과가 같습니다**. 단일 스레드에서는 문제가 없지만, 멀티스레드에서는 충격적인 일이 벌어집니다.

```java
// 초기값 x = 0, y = 0, a = 0, b = 0

스레드 A           스레드 B
──────────       ──────────
x = 1             y = 1
a = y             b = x

// 직관적으로 가능한 결과: (a=0, b=1), (a=1, b=0), (a=1, b=1)
// 그런데 (a=0, b=0) 도 실제로 발생할 수 있음!
//   → 컴파일러가 'x=1과 a=y의 순서를 바꿔도 된다'고 판단할 수 있기 때문
```

이런 재배치는 **메모리 배리어(Memory Barrier)** 라는 장치로 막을 수 있고, `volatile`과 `synchronized`가 내부적으로 이 배리어를 사용합니다.

---

## Java Memory Model (JMM)

JMM은 **어떤 조건에서 한 스레드의 변경이 다른 스레드에 보이는가** 를 정의하는 명세입니다. JMM 없이는 위에서 본 가시성/순서 문제를 언어 차원에서 다룰 수가 없습니다.

```
┌─────────────────────────────────────────────────────┐
│                   메인 메모리                          │
│      (Heap, Method Area — 모든 스레드 공유)             │
└─────────────────────────────────────────────────────┘
       ▲                ▲                ▲
       │                │                │
   ┌───┴────┐       ┌───┴────┐       ┌───┴────┐
   │ 작업    │       │ 작업    │       │ 작업    │
   │ 메모리  │        │ 메모리  │       │ 메모리   │
   │(레지스터/│       │(레지스터/│       │(레지스터/ │
   │ 캐시)   │       │ 캐시)   │       │ 캐시)   │
   └───┬────┘       └───┬────┘       └───┬────┘
       │                │                │
   ┌───┴────┐       ┌───┴────┐       ┌───┴────┐
   │ Thread1│       │ Thread2│       │ Thread3│
   └────────┘       └────────┘       └────────┘
```

JMM은 이 추상 모델 위에서 **언제 작업 메모리의 값을 메인 메모리로 동기화해야 하는가**를 규정합니다.

---

## happens-before 관계

JMM의 핵심 개념입니다. **A happens-before B**라고 하면, A의 효과가 B에서 반드시 보임을 보장합니다.

```
A happens-before B
  → A에서 일어난 모든 메모리 변경이 B 시점에 보장됨
  → A와 B의 순서가 재배치되지 않음
```

JMM이 보장하는 주요 happens-before 관계는 다음과 같습니다.

| 관계                | 설명                                     |
|-------------------|----------------------------------------|
| **Program Order** | 같은 스레드 안에서 앞 줄이 뒷줄보다 먼저                |
| **Monitor Lock**  | `synchronized` 블록 종료가 다음 진입보다 먼저       |
| **Volatile**      | volatile 쓰기가 이후의 volatile 읽기보다 먼저      |
| **Thread Start**  | `Thread.start()` 이전 작업이 그 스레드의 시작보다 먼저 |
| **Thread Join**   | 스레드 안의 모든 작업이 `join()` 이후보다 먼저         |
| **Transitivity**  | A → B, B → C 면 A → C                   |

이 관계가 성립하지 않는 두 작업 사이에서는 **JMM이 가시성을 보장하지 않으므로 race condition이 발생할 수 있습니다.**

---

## volatile — 가시성과 순서를 보장하는 가벼운 도구

`volatile` 키워드가 보장하는 것은 명확합니다.

```
volatile이 보장하는 것
  ✓ 가시성(Visibility): 쓰기 즉시 메인 메모리에 반영, 읽을 때 메인 메모리에서 가져옴
  ✓ 순서(Ordering): 메모리 배리어로 재배치 방지

volatile이 보장하지 않는 것
  ✗ 원자성(Atomicity): count++ 같은 복합 연산은 여전히 깨짐
```

### 가시성 문제 해결

앞서 본 `StopFlag` 예시에 `volatile`을 붙이면 즉시 해결됩니다.

```java
class StopFlag {
    private volatile boolean stopped = false;   // volatile 추가

    public void run() {
        while (!stopped) {
            // ...
        }
    }

    public void stop() {
        stopped = true;
    }
}
```

### volatile은 동작 원리

```
일반 변수 쓰기:
  CPU 레지스터 → L1 캐시 → (언젠가) → 메인 메모리

volatile 쓰기:
  CPU 레지스터 → L1 캐시 → 메모리 배리어(StoreStore + StoreLoad)
                        → 메인 메모리에 즉시 flush
                        → 다른 코어의 캐시 무효화

volatile 읽기:
  → 메모리 배리어(LoadLoad + LoadStore)
  → 메인 메모리에서 직접 읽기
```

### volatile만으로 부족한 경우

```java
class Counter {
    private volatile int count = 0;   // volatile만으론 부족!

    public void increment() {
        count++;   // load → add → store, 여전히 race condition 발생
    }
}
```

`volatile`은 **읽기와 쓰기 각각의 원자성**은 보장하지만, **읽기-수정-쓰기(read-modify-write)** 의 원자성은 보장하지 못합니다. 이 경우엔 `synchronized`
나 `AtomicInteger`(EP.05)가 필요합니다.

### volatile이 적합한 사례

| 사례                    | 예                          |
|-----------------------|----------------------------|
| 단순 플래그                | `volatile boolean stopped` |
| 1회성 발행(publication)   | 객체 참조를 한 번 쓰고 여러 번 읽기      |
| 더블 체크 락(DCL)의 인스턴스 필드 | 싱글톤 구현                     |

---

## synchronized — 가장 기본이 되는 락

`synchronized`는 **모니터 락(Monitor Lock)** 을 사용해 임계 영역(critical section)을 한 번에 하나의 스레드만 들어가도록 제한합니다.

### 두 가지 사용법

```java
// 1. 동기화 메서드 — 인스턴스의 모니터 락 사용
public synchronized void increment() {
    count++;
}

// static 메서드 — Class 객체의 모니터 락 사용
public static synchronized void doSomething() { ... }

// 2. 동기화 블록 — 임의 객체의 모니터 락 사용
public void increment() {
    synchronized (lock) {
        count++;
    }
}
```

### 모니터 락의 동작

모든 Java 객체는 **모니터(Monitor)** 라는 락 정보를 가질 수 있습니다.

```
객체 헤더(Object Header) 안의 Mark Word

  ┌────────────────────────────────────────────────────┐
  │ HashCode | Age | Lock State | Lock Pointer | ...   │
  └────────────────────────────────────────────────────┘
                                    ▲
                                    │
                            락 상태에 따라
                            여기에 락 정보가 저장됨
```

스레드가 `synchronized` 블록에 진입하려면 해당 객체의 모니터 락을 획득해야 하고, 이미 다른 스레드가 쥐고 있으면 **BLOCKED 상태**(EP.01)로 대기합니다.

```
T1이 lock 객체 락 획득 → critical section 실행
T2가 진입 시도 → BLOCKED 상태로 대기 (모니터 큐에 들어감)
T1이 락 해제 → T2가 깨어나 락 획득 → critical section 실행
```

### 가시성과 happens-before

`synchronized`는 원자성뿐 아니라 가시성과 순서까지 모두 보장합니다.

```
synchronized가 보장하는 것
  ✓ Atomicity: 한 번에 한 스레드만 진입
  ✓ Visibility: 락을 풀 때 변경사항을 메인 메모리에 flush
  ✓ Ordering: 락 진입/해제가 메모리 배리어 역할
```

happens-before 관점에서:

```
synchronized (lock) { 
    /* 작업 A */ 
}                    

 → A happens-before B
                          
synchronized (lock) {     
    /* 작업 B */         
}                    
```

같은 락에 대한 unlock → lock 사이에 happens-before가 성립하므로, A의 모든 변경이 B 시점에 보입니다.

### JVM의 synchronized 최적화

`synchronized`는 단순히 OS 락을 거는 것보다 훨씬 정교합니다. JVM은 다음과 같은 최적화를 적용합니다.

| 최적화                     | 설명                                                                      |
|-------------------------|-------------------------------------------------------------------------|
| **Lock Elision**        | 락이 실제로 필요 없는 경우 제거 (escape analysis 기반)                                 |
| **Lock Coarsening**     | 인접한 작은 락들을 하나의 큰 락으로 병합                                                 |
| **Biased Locking**      | 단일 스레드가 반복 진입 시 락 획득 비용 거의 0 *(JDK 15에서 deprecated, JDK 18에서 disabled)* |
| **Lightweight Locking** | 경합 없을 때 CAS로 가볍게 처리                                                     |
| **Heavyweight Locking** | 경합이 심해지면 OS 모니터로 승격                                                     |

이런 최적화들 덕분에 **단일 스레드 / 저경합 환경에서 `synchronized`는 거의 공짜**에 가깝습니다.

---

## synchronized와 Virtual Thread — JEP 491 이야기

EP.03에서 짧게 언급했던 핀닝(Pinning) 문제를 여기서 본격적으로 봅니다.

### Java 21~23: 핀닝 문제

Virtual Thread가 `synchronized` 블록 안에서 블로킹 I/O를 만나면 캐리어 스레드에 **핀(pin)** 됩니다. 캐리어를 놓아주지 못하므로 다른 Virtual Thread가 그 캐리어를 쓸 수
없게 됩니다.

```
Virtual Thread VT1이 synchronized 진입
   │
   ├─ Carrier C1에 마운트
   │
   ├─ DB 쿼리 실행 (블로킹 I/O)
   │
   ▼
   ※ 일반 상황이라면 VT1은 C1에서 언마운트되고
     C1은 다른 VT를 받아야 함
   
   그런데 synchronized 안에서는?
   → 모니터가 캐리어 C1을 기준으로 락을 추적하므로
   → VT1이 언마운트되면 락 추적이 깨짐
   → JVM이 안전을 위해 VT1을 C1에 "고정(pin)"
   → C1은 VT1의 I/O 대기 동안 아무것도 못 함
```

이 문제 때문에 한동안 **"synchronized를 ReentrantLock으로 바꿔라"** 라는 가이드가 강했습니다.

### Java 24: JEP 491이 해결

Java 24의 JEP 491은 모니터를 **캐리어가 아닌 Virtual Thread 자체에 연결**하도록 JVM을 다시 구현했습니다.

```
Java 24+
  Virtual Thread VT1이 synchronized 진입
     │
     ├─ Carrier C1에 마운트
     ├─ 모니터 락은 VT1과 연결됨 (C1이 아니라)
     │
     ├─ DB 쿼리 → 블로킹
     ├─ VT1 언마운트, 락은 VT1이 그대로 보유
     ├─ C1은 자유로워져 다른 VT 처리 가능
     │
     ├─ I/O 완료
     ├─ VT1이 다시 마운트(다른 Carrier일 수도 있음)
     └─ synchronized 블록 계속 실행
```

JEP 491 저자의 직접적인 입장:

> "synchronized를 ReentrantLock으로 바꾸는 것을 더 이상 권장하지 않는다.
> 어떤 락이 문제에 더 잘 맞는지를 기준으로 선택하면 된다."

**정리: 2026년 현재(Java 24+ 기준), `synchronized`는 여전히 1급 시민이며 안심하고 사용해도 됩니다.**

---

## 데드락(Deadlock)

`synchronized`를 쓸 때 가장 큰 함정입니다.

```java
// 위험한 코드
class Account {
    synchronized void transfer(Account other, int amount) {
        synchronized (other) {              // 다른 락도 획득
            this.balance -= amount;
            other.balance += amount;
        }
    }
}

// 실행:
//   T1: a.transfer(b, 100)   → a 락 획득, b 락 대기
//   T2: b.transfer(a, 100)   → b 락 획득, a 락 대기
//   → 서로 영원히 대기 → 데드락
```

```
   ┌───────┐                    ┌───────┐
   │  T1   │  ── 가지고 있음 ──→   │ a 락  │
   │       │                    └──┬────┘
   │       │                       │
   │       │                       │ T2가 가지려고 대기
   │       │                       │
   │       │                   ┌───▼───┐
   │       │ ←── 가지려고 대기 ── │  T2   │
   │       │                   │       │
   │       │                   │       │
   └───┬───┘                   └───┬───┘
       │                           │
       │ 가지려고 대기                  │ 가지고 있음
       │                           │
   ┌───▼───┐                   ┌───▼───┐
   │ b 락  │ ←───────────────── │       │
   └───────┘                   └───────┘
```

### 데드락이 발생하는 4가지 조건 (Coffman 조건)

```
1. 상호 배제(Mutual Exclusion): 자원을 한 번에 한 스레드만 점유
2. 점유와 대기(Hold and Wait): 자원을 쥔 채 다른 자원 대기
3. 비선점(No Preemption): 다른 스레드가 강제로 뺏을 수 없음
4. 순환 대기(Circular Wait): T1→T2→T3→T1처럼 순환 형태
```

이 중 하나라도 깨면 데드락이 발생하지 않습니다. 가장 실용적인 해결책은 **순환 대기를 깨는 것** — 락의 획득 순서를 정해두는 방법입니다.

```java
void transfer(Account other, int amount) {
    Account first  = this.id < other.id ? this : other;
    Account second = this.id < other.id ? other : this;
    
    synchronized (first) {
        synchronized (second) {
            // ...
        }
    }
}
```

스레드들이 항상 같은 순서로 락을 획득하므로 순환이 만들어지지 않습니다.

`jstack`은 데드락을 자동 감지해서 다음과 같이 알려줍니다.

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x... (object 0x..., a Account),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x... (object 0x..., a Account),
  which is held by "Thread-1"
```

---

## volatile vs synchronized — 언제 무엇을?

| 상황                                           | 권장                                 |
|----------------------------------------------|------------------------------------|
| 단순 플래그 (`boolean stopped`)                   | `volatile`                         |
| 객체 참조 발행 (write once, read many)             | `volatile`                         |
| read-modify-write (`count++`, `if-then-set`) | `synchronized` 또는 `Atomic` (EP.05) |
| 복합 연산 (여러 변수를 함께 일관되게 변경)                    | `synchronized`                     |
| 코드 블록 단위의 임계영역                               | `synchronized`                     |

`volatile`은 **가시성/순서만 필요한 가장 가벼운 도구**, `synchronized`는 **그 위에 원자성과 임계영역까지 보장하는 종합 도구**라고 기억하면 됩니다.

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 주제                | 핵심 내용                                              |
|-------------------|----------------------------------------------------|
| 멀티스레드의 위험         | 원자성, 가시성, 순서 — 세 가지 깨짐                             |
| 가시성 문제            | CPU 캐시 때문에 변경이 다른 스레드에 안 보일 수 있음                   |
| 순서 문제             | 컴파일러/CPU의 명령어 재배치                                  |
| Java Memory Model | 메인 메모리와 작업 메모리의 동기화 규칙                             |
| happens-before    | A의 효과가 B에서 보장된다는 관계                                |
| volatile          | 가시성과 순서 보장, 원자성은 X                                 |
| synchronized      | 가시성, 순서, 원자성 모두 보장 + 임계영역                          |
| JVM 최적화           | Lock Elision/Coarsening, Lightweight → Heavyweight |
| JEP 491           | Java 24에서 synchronized의 Virtual Thread 핀닝 해결       |
| 데드락               | 4가지 조건, 락 획득 순서 통일로 해결                             |

이번 편의 모든 도구는 **"남이 못 들어오게 막는다(Blocking)"** 라는 철학 위에 세워져 있습니다. 락을 쥔 스레드가 일을 끝낼 때까지 다른 스레드는 기다려야 하죠.

그런데 이런 의문이 듭니다.

> 매번 막고 기다리는 게 정말 최선일까?
> "충돌이 나면 다시 시도하자"는 방식은 안 될까?

이게 바로 **락 없는 동시성(Lock-free Concurrency)** 의 출발점입니다.

다음 편(시리즈 마지막)에서는 **CAS(Compare-And-Swap)와 Atomic 클래스** 를 다룹니다. CPU 명령어 한 줄로 동시성을 푸는
방법, `AtomicInteger.incrementAndGet()`의 내부, 그리고 ABA 문제 같은 함정까지 살펴보겠습니다.