---
title: "EP.04 Execution Engine과 JVM 스레드"
date: "2025-04-07"
episode: 4
tags: [ "JVM", "Java", "JIT", "Execution Engine", "Thread", "Virtual Thread" ]
---

지난 편에서 GC가 Heap의 쓰레기를 어떻게 수거하는지 살펴봤습니다.
그렇다면 ClassLoader가 메모리에 올린 ByteCode는 실제로 어떻게 실행될까요?
그리고 JVM은 여러 스레드를 어떻게 관리할까요?

이번 편에서는 **Execution Engine**과 **JVM 스레드**의 내부 동작을 파헤쳐보겠습니다.🙂

이 글은 JVM Deep Dive 시리즈의 네 번째 편입니다.

## Execution Engine이란?

Execution Engine은 **Runtime Data Area에 적재된 ByteCode를 실제로 실행하는 엔진**입니다.

JVM 전체 구조에서 보면 마지막 단계에 해당합니다.

```
┌──────────────────────────────────────────────────────┐
│                       JVM                            │
│                                                      │
│   ┌──────────────────────────────────────────────┐   │
│   │            Class Loader Subsystem            │   │
│   └──────────────────────────────────────────────┘   │
│                         │                            │
│                         ▼                            │
│   ┌──────────────────────────────────────────────┐   │
│   │              Runtime Data Area               │   │
│   │          Method Area · Heap · Stack          │   │
│   └──────────────────────────────────────────────┘   │
│                         │                            │
│                         ▼                            │
│   ┌──────────────────────────────────────────────┐   │
│   │             ★ Execution Engine ★             │   │
│   │        Interpreter + JIT Compiler + GC       │   │
│   └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

Execution Engine의 핵심 구성요소는 두 가지입니다.

```
Execution Engine
  ├─ Interpreter   : ByteCode를 한 줄씩 해석해서 실행
  └─ JIT Compiler  : 자주 실행되는 코드를 네이티브 코드로 미리 컴파일
```

이 두 가지는 각각의 장단점이 있고, JVM은 둘을 협력시켜 성능을 극대화합니다.

---

## Interpreter — ByteCode를 한 줄씩 읽는다

Interpreter는 가장 단순한 실행 방식입니다. ByteCode 명령어(opcode)를 **한 줄씩 읽고, 해석하고, 실행**합니다.

```
ByteCode (Hello.class)
  0: getstatic     #7
  3: ldc           #13
  5: invokevirtual #15
  8: return

Interpreter 동작:
  Step 1: getstatic 읽기 → "System.out 가져와"  → 실행
  Step 2: ldc 읽기       → "Hello, JVM! 로드"  → 실행
  Step 3: invokevirtual  → "println 호출"     → 실행
  Step 4: return         → "메서드 종료"        → 실행
```

### Interpreter의 장단점

```
장점 :
  ✅ JVM 시작 즉시 실행 가능 (별도 컴파일 시간 없음)
  ✅ 메모리 사용량 적음

단점 :
  ❌ 같은 코드를 반복 실행할 때마다 매번 해석
  ❌ 반복 횟수가 많아질수록 성능 저하
```

```java
// 이 루프에서 Interpreter는 매 반복마다 ByteCode를 다시 해석한다
for (int i = 0; i < 100_000; i++) {
    calculate(i);   // 10만 번 호출 → 10만 번 해석
}
```

같은 코드를 매번 해석하는 건 비효율적입니다. 여기서 JIT Compiler가 등장합니다.

---

## JIT Compiler — 자주 쓰는 코드는 미리 번역한다

**JIT(Just-In-Time) Compiler**는 자주 실행되는 코드(핫스팟, HotSpot)를 감지해서 **네이티브 코드(기계어)로 미리 컴파일**해두는 방식입니다.

```
처음 실행 시 (Interpreter) :
  코드 실행 → ByteCode 해석 → 실행 (느림)
  코드 실행 → ByteCode 해석 → 실행 (느림)
  코드 실행 → ByteCode 해석 → 실행 (느림)
  ...반복 횟수 임계값 도달...

JIT 컴파일 시작 :
  ByteCode → [JIT Compiler] → 네이티브 코드 (기계어)

이후 실행 (네이티브 코드):
  코드 실행 → 네이티브 코드 직접 실행 (빠름!)
  코드 실행 → 네이티브 코드 직접 실행 (빠름!)
```

### HotSpot JVM의 두 가지 JIT 컴파일러

HotSpot JVM(Oracle JDK, OpenJDK)에는 두 종류의 JIT 컴파일러가 있습니다.

```
C1 Compiler (Client Compiler)
  ├─ 빠르게 컴파일
  ├─ 최적화 수준 : 낮음
  └─ 목적: 빠른 시작 시간

C2 Compiler (Server Compiler)
  ├─ 느리게 컴파일 (더 많이 분석)
  ├─ 최적화 수준 : 높음
  └─ 목적: 최고 실행 성능
```

### JIT 컴파일러의 최적화 기법

JIT는 단순히 ByteCode를 기계어로 바꾸는 것뿐 아니라 다양한 최적화를 적용합니다.

**Inlining (인라이닝)**
자주 호출되는 짧은 메서드를 호출 지점에 직접 삽입합니다.

```java
// 원래 코드
int result = add(a, b);

private int add(int x, int y) { return x + y; }

// JIT 인라이닝 후 (메서드 호출 오버헤드 제거)
int result = a + b;
```

**Dead Code Elimination (죽은 코드 제거)**

```java
// 실행될 수 없는 코드는 제거
if (false) {
    expensiveOperation();  // JIT가 이 코드를 아예 제거
}
```

**Loop Unrolling (루프 언롤링)**

```java
// 원래 코드
for (int i = 0; i < 4; i++) {
    process(i);
}

// JIT 최적화 후 (루프 오버헤드 제거)
process(0);
process(1);
process(2);
process(3);
```

---

## Tiered Compilation — Interpreter와 JIT의 협력

Java 8부터 기본으로 적용된 **Tiered Compilation**은 Interpreter, C1, C2를 단계적으로 활용합니다.

```
Level 0 : Interpreter
  → 처음 실행. ByteCode 해석

Level 1~3 : C1 Compiler
  → 호출 횟수가 늘어나면 C1으로 컴파일
  → 빠른 컴파일, 기본 최적화

Level 4 : C2 Compiler
  → 충분히 많이 호출된 핫스팟은 C2로 재컴파일
  → 느린 컴파일, 강력한 최적화
```

```
실행 횟수
      │
  많음 │                              ┌─── Level 4 (C2)
      │                         ┌────┘    최고 성능
      │                   ┌─────┘
      │            ┌──────┘ Level 1~3 (C1)
      │      ──────┘
  적음 │ Level 0 (Interpreter)
      └─────────────────────────────────────────→ 시간
        시작         초반          이후
```

덕분에 JVM은 **시작 초반에는 빠르게 실행**하고, **시간이 지날수록 점점 더 최적화된 성능**을 제공합니다.

```bash
# Tiered Compilation 비활성화 (성능 테스트 시)
$ java -XX:-TieredCompilation MyApp

# JIT 컴파일 로그 확인
$ java -XX:+PrintCompilation MyApp
```

---

## Code Cache와 Deoptimization

### Code Cache

JIT가 컴파일한 네이티브 코드는 **Code Cache**라는 별도 메모리 영역에 저장됩니다.

```
Runtime Data Area 외부
┌────────────────────────────┐
│         Code Cache         │
│   ┌────────────────────┐   │
│   │ calculate() 네이티브 │   │
│   ├────────────────────┤   │
│   │ process() 네이티브   │   │
│   ├────────────────────┤   │
│   │ ...                │   │
│   └────────────────────┘   │
└────────────────────────────┘
```

Code Cache가 가득 차면 JIT 컴파일이 중단되어 성능이 급락할 수 있습니다.

```bash
# Code Cache 크기 설정
$ java -XX:ReservedCodeCacheSize=256m MyApp
```

### Deoptimization (역최적화)

JIT는 실행 중 수집한 정보를 바탕으로 **가정(Assumption)** 을 세워 최적화합니다. 그런데 이 가정이 깨지면 다시 Interpreter로 돌아가야 합니다.
이것을 **Deoptimization**이라고 합니다.

```java
// JIT가 "animal은 항상 Dog 타입"이라고 가정하고 최적화
Animal animal = new Dog();
animal.sound();   // → JIT : "Dog.sound()를 직접 호출"로 최적화

// 나중에 Cat이 들어오면 가정이 깨짐
animal = new Cat();
animal.sound();   // → Deoptimization 발생 → Interpreter로 복귀
```

```bash
# Deoptimization 발생 확인
$ java -XX:+PrintDeoptimizationDetails MyApp
```

---

## JVM 스레드 — 생애주기

Java의 스레드는 OS 스레드와 1:1로 매핑됩니다 (Virtual Thread 제외, 뒤에서 설명).

```
Java Thread ──────────────────────────────→ OS Thread
(java.lang.Thread)                         (커널 스레드)
```

### 스레드 상태 (Thread State)

Java 스레드는 총 6가지 상태를 가집니다.

```
          start()
NEW ──────────────→ RUNNABLE
                        │
          synchronized  │  Object.wait()
          lock 획득 실패  │  Thread.sleep()
                 ↓      │  join() 등
            BLOCKED     ↓
                WAITING / TIMED_WAITING
                        │
                notify() / notifyAll()
                   sleep 시간 만료
                     join 완료
                        │
                        ↓
                     RUNNABLE
                        │
                    run() 종료
                        │
                        ↓
                    TERMINATED
```

| 상태                | 설명                                        |
|-------------------|-------------------------------------------|
| **NEW**           | `Thread` 객체 생성, 아직 `start()` 호출 전         |
| **RUNNABLE**      | 실행 중 또는 실행 대기 중 (CPU 할당 기다리는 중 포함)        |
| **BLOCKED**       | `synchronized` 블록 진입을 위해 lock 대기 중        |
| **WAITING**       | `wait()`, `join()` 호출로 무기한 대기 중           |
| **TIMED_WAITING** | `sleep()`, `wait(timeout)` 등으로 일정 시간 대기 중 |
| **TERMINATED**    | `run()` 메서드 종료                            |

```java
Thread thread = new Thread(() -> {
    System.out.println("실행 중: " + Thread.currentThread().getState());
});

System.out.println("시작 전: " + thread.getState());  // NEW
thread.start();
thread.join();
System.out.println("종료 후: " + thread.getState());  // TERMINATED
```

### Thread Dump로 스레드 상태 확인

실제 운영 환경에서 스레드 상태를 진단할 때는 Thread Dump를 활용합니다.

```bash
# Thread Dump 생성
$ jstack <PID> > thread_dump.txt

# 또는 kill 시그널로
$ kill -3 <PID>
```

```
Thread Dump 예시:
"http-nio-8080-exec-1" #25 daemon prio=5 os_prio=0 tid=0x... nid=0x... waiting on condition
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
```

BLOCKED 상태의 스레드가 많다면 **데드락 또는 lock 경쟁**이 의심됩니다.

---

## Monitor와 synchronized 내부 구조

Java의 모든 객체는 내부적으로 **Monitor**를 가지고 있습니다. `synchronized` 키워드는 이 Monitor를 통해 동기화를 구현합니다.

```java
public synchronized void increment() {  // 메서드 전체에 Monitor lock
    count++;
}

public void process() {
    synchronized (this) {               // 특정 블록에만 Monitor lock
        count++;
    }
}
```

### Monitor 동작 방식

```
객체 헤더 (Object Header)
┌─────────────────────────────────────┐
│          Mark Word (64bit)          │
│     lock 상태, hash code, GC 정보 등   │
├─────────────────────────────────────┤
│            Klass Pointer            │
│         (클래스 메타데이터 주소)          │
└─────────────────────────────────────┘

Monitor 구조:
┌─────────────────────────────────────┐
│              Monitor                │
│  ┌─────────────────────────────┐    │
│  │  Owner (현재 lock 보유 스레드)  │    │
│  ├─────────────────────────────┤    │
│  │  Entry Set (lock 대기 스레드)  │    │
│  ├─────────────────────────────┤    │
│  │  Wait Set (wait() 대기 스레드) │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

```
Thread A가 synchronized 블록 진입 시도:
  1. Monitor의 Owner가 비어있음 → Thread A가 Owner 등록
  2. Thread A가 임계 영역 실행

Thread B가 같은 synchronized 블록 진입 시도:
  1. Monitor의 Owner가 Thread A → Thread B는 Entry Set에서 대기 (BLOCKED)

Thread A가 임계 영역 완료:
  1. Monitor의 Owner 반환
  2. Entry Set의 Thread B에게 기회 부여 → Thread B가 Owner 등록
```

---

## Lock의 3단계 — Biased / Lightweight / Heavyweight

JVM은 성능을 위해 lock을 상황에 따라 세 단계로 처리합니다. 객체의 Mark Word를 변경하는 방식으로 구현됩니다.

### Biased Lock (편향 잠금)

```
가정: 대부분의 경우 lock은 항상 같은 스레드가 획득한다.

동작:
  최초 lock 획득 → Mark Word에 Thread ID 기록
  이후 같은 스레드 → Mark Word만 확인, CAS 연산 없이 바로 진입

비용: 거의 없음 (Mark Word 확인만)
```

```
Mark Word:
┌──────────────────────────────────┐
│  Thread ID  │  epoch  │  1(편향)  │
└──────────────────────────────────┘
```

### Lightweight Lock (경량 잠금)

```
가정: 스레드 경합이 있지만 금방 끝난다.

동작:
  lock 경합 발생 → Biased Lock 해제
  CAS(Compare-And-Swap) 연산으로 lock 획득 시도
  스핀(spin)하며 반복 시도 → 일정 횟수 내 성공하면 OK

비용: CAS 연산 (OS 개입 없음)
```

### Heavyweight Lock (중량 잠금)

```
가정: lock 경합이 심하고 오래 걸린다.

동작:
  스핀 시도 실패 → OS의 Mutex 사용
  lock 획득 실패한 스레드 → OS가 직접 BLOCKED 상태로 전환
  lock 해제 시 → OS가 대기 스레드 깨움

비용: OS 컨텍스트 스위치 (가장 비쌈)
```

```
lock 경합 강도에 따른 자동 전환:

경합 없음          경합 있음           경합 심함
   │                │                │
   ▼                ▼                ▼
Biased Lock → Lightweight Lock → Heavyweight Lock
(가장 빠름)         (중간)           (가장 느림)
```

JVM이 상황에 따라 자동으로 lock 단계를 전환하기 때문에, 개발자는 `synchronized`만 쓰면 JVM이 알아서 최적의 방식을 선택합니다.

---

## Virtual Thread (Project Loom)

Java 21에서 정식 도입된 **Virtual Thread**는 기존 스레드 모델의 한계를 극복하기 위해 만들어졌습니다.

### 기존 스레드의 한계

```
기존 (Platform Thread):
  Java Thread : OS Thread = 1 : 1

  Java Thread 1 ──→ OS Thread 1
  Java Thread 2 ──→ OS Thread 2
  Java Thread 3 ──→ OS Thread 3
  ...
  Java Thread N ──→ OS Thread N

문제:
  - OS Thread 생성 비용이 큼 (약 1MB 스택 메모리)
  - 스레드 수가 많아지면 컨텍스트 스위치 비용 증가
  - I/O 대기 중에도 OS Thread를 점유
```

수천 개의 동시 요청을 처리하는 서버에서는 스레드 수가 병목이 됩니다.

### Virtual Thread의 해결 방식

```
Virtual Thread:
  Virtual Thread : OS Thread = N : 1 (또는 N : M)

  Virtual Thread 1 ─┐
  Virtual Thread 2 ─┤─→ OS Thread 1 (Carrier Thread)
  Virtual Thread 3 ─┘
  Virtual Thread 4 ─┐
  Virtual Thread 5 ─┤─→ OS Thread 2 (Carrier Thread)
  ...               ┘

  I/O 대기 시:
    Virtual Thread → OS Thread에서 분리 (unmount)
    다른 Virtual Thread → 해당 OS Thread 사용
    I/O 완료 시 → Virtual Thread 재연결 (mount)
```

### 사용 방법

```java
// 기존 Platform Thread
Thread thread = new Thread(() -> {
    System.out.println("Platform Thread");
});
thread.start();

// Virtual Thread (Java 21+)
Thread vThread = Thread.ofVirtual().start(() -> {
    System.out.println("Virtual Thread");
});

// ExecutorService로 사용
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> handleRequest());  // 10만 개도 가볍게!
    }
}
```

### Virtual Thread 특징 정리

| 구분           | Platform Thread | Virtual Thread |
|--------------|-----------------|----------------|
| OS Thread 매핑 | 1:1             | N:1            |
| 생성 비용        | 높음 (약 1MB)      | 낮음 (약 수 KB)    |
| 최대 동시 수      | 수천 개            | 수백만 개          |
| I/O 대기       | OS Thread 점유    | OS Thread 반환   |
| 적합한 작업       | CPU 집약적         | I/O 집약적        |

```
주의:
  Virtual Thread는 I/O 집약적 작업에 효과적입니다.
  CPU 집약적인 작업(복잡한 계산 등)은 여전히 Platform Thread가 더 적합합니다.
```

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 주제                     | 핵심 내용                                               |
|------------------------|-----------------------------------------------------|
| Interpreter            | ByteCode를 한 줄씩 해석하여 실행, 시작은 빠르나 반복 시 비효율            |
| JIT Compiler           | 핫스팟 코드를 네이티브 코드로 컴파일, C1(빠른 컴파일) + C2(강한 최적화)       |
| Tiered Compilation     | Interpreter → C1 → C2 단계적 전환으로 시작 속도와 최적화 모두 확보     |
| Code Cache             | JIT 컴파일된 네이티브 코드 저장 공간                              |
| Deoptimization         | JIT 가정이 깨지면 Interpreter로 복귀                         |
| 스레드 생애주기               | NEW → RUNNABLE → BLOCKED/WAITING → TERMINATED       |
| Monitor & synchronized | 모든 객체가 가진 Monitor를 통해 상호 배제 구현                      |
| Lock 3단계               | Biased → Lightweight → Heavyweight, 경합 강도에 따라 자동 전환 |
| Virtual Thread         | N:1 매핑으로 수백만 개의 경량 스레드 처리 (Java 21+)                |

드디어 마지막 편만 남았습니다.
다음 편에서는 지금까지 배운 JVM 지식을 바탕으로 **실제로 JVM을 모니터링하고 성능을 튜닝하는 방법**,
그리고 GraalVM과 Virtual Thread로 대표되는 **JVM의 미래**를 살펴보겠습니다.

