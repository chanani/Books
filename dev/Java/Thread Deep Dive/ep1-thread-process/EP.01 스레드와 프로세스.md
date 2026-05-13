---
title: "EP.01 스레드와 프로세스"
date: "2026-05-08"
episode: 1
tags: ["Thread", "Java", "Concurrency", "Process"]
---

이번에는 스레드를 제대로 파헤쳐보기로 했습니다.🙂

이 글은 Thread Deep Dive 시리즈의 첫 번째 편입니다.

## 왜 스레드인가?

지난 JVM 시리즈에서 우리는 JVM이 메모리를 어떻게 나누는지 살펴봤습니다.
그중에서 **JVM Stack, PC Register, Native Method Stack** 은 "스레드 전용" 영역이었죠.
즉, JVM의 메모리 구조 자체가 이미 **여러 스레드가 동시에 돌아간다는 전제** 위에서 설계되어 있습니다.

현대 서버 애플리케이션은 거의 예외 없이 멀티스레드입니다.

```
한 명의 사용자 요청만 처리하는 웹 서버는 없다.
→ Tomcat은 기본 200개의 워커 스레드로 요청을 동시에 처리한다.
→ Spring의 @Async는 별도 스레드 풀에서 비동기 작업을 돌린다.
→ GC, JIT 컴파일러, Finalizer도 모두 별도의 스레드에서 동작한다.
```

스레드를 모르면 동시성 버그도, 성능 튜닝도, 데드락도 제대로 잡을 수 없습니다.
시리즈의 출발점으로 **프로세스와 스레드의 차이**, 그리고 **스레드의 생명주기**를 살펴보겠습니다.

---

## 프로세스 vs 스레드

OS 관점에서 둘의 차이는 **메모리 공간을 공유하는가**로 갈립니다.

### 프로세스 (Process)

운영체제로부터 자원을 할당받은 **실행 중인 프로그램**입니다. 각 프로세스는 자신만의 독립된 메모리 공간을 가집니다.

```
┌─────────────────────────┐  ┌─────────────────────────┐
│      프로세스 A           │  │      프로세스 B           │
│                         │  │                         │
│  ┌───────────────────┐  │  │  ┌───────────────────┐  │
│  │     Code(Text)    │  │  │  │     Code(Text)    │  │
│  │     Data          │  │  │  │     Data          │  │
│  │     Heap          │  │  │  │     Heap          │  │
│  │     Stack         │  │  │  │     Stack         │  │
│  └───────────────────┘  │  │  └───────────────────┘  │
│                         │  │                         │
│  ┌─────┐                │  │  ┌─────┐                │
│  │ PCB │ (PID, 상태 등)   │  │  │ PCB │                │
│  └─────┘                │  │  └─────┘                │
└─────────────────────────┘  └─────────────────────────┘
                                                       
        ↑ 서로 독립된 메모리 공간 (격리됨)
```

프로세스끼리 데이터를 주고받으려면 **IPC(Inter-Process Communication)** 가 필요합니다. 파이프, 소켓, 공유 메모리 등을 거쳐야 하므로 비용이 큽니다.

### 스레드 (Thread)

프로세스 내부에서 실제로 **CPU를 점유하며 코드를 실행하는 흐름**입니다. 한 프로세스 안의 여러 스레드는 **Code, Data, Heap을 공유**하고, **Stack과 PC Register만 각자** 가집니다.

```
┌─────────────────────────────────────────────────┐
│                  프로세스 A                       │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │    Code (Text)  /  Data  /  Heap          │  │  ← 모든 스레드가 공유
│  └───────────────────────────────────────────┘  │
│                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐     │
│  │ Thread 1 │   │ Thread 2 │   │ Thread 3 │     │
│  │  Stack   │   │  Stack   │   │  Stack   │     │  ← 스레드 전용
│  │  PC      │   │  PC      │   │  PC      │     │
│  │  Reg     │   │  Reg     │   │  Reg     │     │
│  └──────────┘   └──────────┘   └──────────┘     │
└─────────────────────────────────────────────────┘
```

JVM 시리즈 EP.02에서 봤던 "스레드 공유 영역(Heap, Method Area)"과 "스레드 전용 영역(Stack, PC Register)" 구분이 바로 이 모델을 그대로 따르고 있습니다.

### 한눈에 비교

| 구분 | 프로세스 | 스레드 |
|---|---|---|
| **메모리** | 독립된 공간 | 같은 프로세스 내 공유 |
| **생성 비용** | 높음 (메모리 할당, PCB 생성) | 낮음 (Stack, PC만 추가) |
| **컨텍스트 스위칭 비용** | 높음 (메모리 맵 전환) | 낮음 (메모리 맵 공유) |
| **통신** | IPC 필요 | 메모리 공유로 직접 |
| **장애 영향** | 한 프로세스 죽어도 다른 프로세스 영향 X | 한 스레드 죽으면 프로세스 전체 영향 가능 |

---

## 동시성(Concurrency) vs 병렬성(Parallelism)

스레드 얘기를 할 때 자주 헷갈리는 두 단어입니다.

```
동시성 (Concurrency)          병렬성 (Parallelism)

      CPU 1개                   CPU 2개 이상
  ┌──────────────┐           ┌──────────────┐
  │  T1  T2  T1  │           │ T1 T1 T1 T1  │
  │     T2  T1   │           │ T2 T2 T2 T2  │
  └──────────────┘           └──────────────┘
     시간 분할로                   실제로 동시에
   "동시처럼 보이게"                  실행됨
```

| 개념 | 설명 |
|---|---|
| **동시성** | 여러 작업을 **논리적으로 동시에** 처리. CPU가 1개여도 가능 |
| **병렬성** | 여러 작업을 **물리적으로 동시에** 처리. CPU(코어)가 여러 개 필요 |

> 💡 단일 코어 CPU에서도 멀티스레드 프로그램은 의미가 있습니다. I/O 대기 시간 동안 다른 스레드가 CPU를 쓸 수 있기 때문입니다.

---

## Java에서 스레드를 만드는 5가지 방법

### 1. `Thread` 클래스 상속

가장 원시적이지만 권장되지 않는 방식입니다. Java는 단일 상속만 지원하므로 다른 클래스를 상속받을 수 없게 됩니다.

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Running on " + Thread.currentThread().getName());
    }
}

new MyThread().start();
```

### 2. `Runnable` 구현

가장 일반적이고 권장되는 방식입니다. 작업(Runnable)과 실행 주체(Thread)를 분리합니다.

```java
Runnable task = () -> System.out.println("Hello from " + Thread.currentThread().getName());
new Thread(task).start();
```

### 3. `Callable` + `Future`

`Runnable`은 반환값이 없고 예외를 throws할 수 없지만, `Callable`은 둘 다 가능합니다.

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};

Future<Integer> future = executor.submit(task);
Integer result = future.get();   // 블로킹: 작업이 끝날 때까지 대기
executor.shutdown();
```

### 4. `ExecutorService` (스레드 풀)

실무에서 가장 많이 쓰입니다. 스레드를 매번 만들고 버리는 대신 풀에서 재사용합니다.

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

for (int i = 0; i < 100; i++) {
    final int taskId = i;
    executor.submit(() -> System.out.println("Task " + taskId));
}

executor.shutdown();
```

### 5. Virtual Thread (Java 21+)

JDK 21에서 정식 도입된 가벼운 스레드입니다. OS 스레드 1:1이 아니라 JVM이 스케줄링합니다.

```java
Thread.startVirtualThread(() -> {
    System.out.println("Virtual thread: " + Thread.currentThread());
});

// 또는 ExecutorService로
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> doSomething());
}
```

> 💡 Virtual Thread는 자체로도 큰 주제라 이번 시리즈에서는 EP.03에서 컨셉만 소개하고 깊은 내용은 별도로 다룹니다.

### 정리

| 방식 | 장점 | 단점 |
|---|---|---|
| **Thread 상속** | 단순 | 단일 상속 제약, 재사용 불가 |
| **Runnable** | 작업과 실행 분리 | 반환값/예외 처리 불가 |
| **Callable + Future** | 반환값, 예외 처리 가능 | `get()`이 블로킹 |
| **ExecutorService** | 스레드 재사용, 풀 관리 | 풀 설정에 신경 써야 함 |
| **Virtual Thread** | 매우 가볍고 많이 만들 수 있음 | 학습 비용, JDK 21+ |

---

## 스레드 생명주기 (Thread Lifecycle)

`Thread.State` enum에 정의된 6가지 상태가 있습니다.

```
                    ┌─────┐
                    │ NEW │   ← Thread 객체 생성, 아직 start() 호출 X
                    └──┬──┘
                       │ start()
                       ▼
                  ┌──────────┐
                  │ RUNNABLE │   ← 실행 중이거나 실행 준비 완료
                  └────┬─────┘   (OS 스케줄러가 CPU 할당하면 실행)
                       │
        ┌──────────────┼────────────────────┐
        │              │                    │
        ▼              ▼                    ▼
   ┌─────────┐   ┌──────────┐      ┌──────────────┐
   │ BLOCKED │   │ WAITING  │      │TIMED_WAITING │
   │         │   │          │      │              │
   │ 모니터 락 │   │ 무기한 대기 │      │  시간 제한 대기 │
   │ 획득 대기 │   │ wait() / │      │   sleep(t),  │
   │         │   │ join()   │      │   wait(t)    │
   └────┬────┘   └────┬─────┘      └──────┬───────┘
        │             │                   │
        └─────────────┴───────────────────┘
                      │
                      ▼
                 ┌──────────┐
                 │TERMINATED│   ← run() 종료 또는 예외
                 └──────────┘
```

| 상태 | 의미 |
|---|---|
| **NEW** | `Thread` 객체 생성됐지만 `start()` 호출되지 않음 |
| **RUNNABLE** | 실행 중이거나 실행 준비 완료 (OS의 스케줄링 대기) |
| **BLOCKED** | `synchronized` 블록의 모니터 락 획득을 위해 대기 |
| **WAITING** | `wait()`, `join()` 등으로 무기한 대기 |
| **TIMED_WAITING** | `sleep(ms)`, `wait(ms)` 등 시간 제한 대기 |
| **TERMINATED** | `run()` 메서드 정상 종료 또는 예외로 종료 |

### 코드로 확인하기

```java
public class ThreadStateDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            try {
                Thread.sleep(1000);   // → TIMED_WAITING
            } catch (InterruptedException ignored) {}
        });

        System.out.println(t.getState());   // NEW

        t.start();
        System.out.println(t.getState());   // RUNNABLE

        Thread.sleep(100);
        System.out.println(t.getState());   // TIMED_WAITING

        t.join();
        System.out.println(t.getState());   // TERMINATED
    }
}
```

### `BLOCKED`와 `WAITING`의 차이

자주 헷갈리는 부분입니다.

```
BLOCKED:
  synchronized 락을 다른 스레드가 쥐고 있어서
  락을 얻으려고 줄 서서 대기.
  → "내가 들어가야 하는데 문이 잠겨 있다"

WAITING:
  스스로 대기 상태로 들어감 (wait(), join(), park()).
  다른 스레드가 깨워줘야(notify, unpark) 다시 RUNNABLE이 됨.
  → "누가 신호 줄 때까지 자고 있겠다"
```

스레드 덤프를 분석할 때 이 차이를 알아야 데드락인지 단순 대기인지 구분할 수 있습니다.

---

## 데몬 스레드(Daemon Thread)

JVM은 **모든 사용자 스레드(non-daemon)가 종료되면 자동으로 종료**됩니다. 데몬 스레드는 이 종료 조건에서 무시됩니다.

```java
Thread daemon = new Thread(() -> {
    while (true) {
        System.out.println("백그라운드 작업 중...");
        try { Thread.sleep(1000); } catch (InterruptedException e) { return; }
    }
});
daemon.setDaemon(true);   // 반드시 start() 전에 호출
daemon.start();

// main 스레드가 끝나면 daemon은 강제 종료됨
```

### 활용 예

- **GC 스레드**: 가장 대표적인 데몬 스레드
- **JIT 컴파일러 스레드**
- **모니터링/로깅 백그라운드 작업**

> ⚠️ 데몬 스레드는 강제 종료될 수 있으므로 **중요한 정리 작업(파일 닫기, 트랜잭션 커밋)을 절대 맡기면 안 됩니다.**

---

## main 스레드와 JVM 종료

JVM이 `public static void main(...)`을 호출할 때, 이 메서드는 **main이라는 이름의 사용자 스레드** 위에서 실행됩니다.

```
JVM 시작
  │
  ├─ main 스레드 생성 → main() 실행
  │
  ├─ GC 스레드 (daemon)
  ├─ JIT Compiler 스레드 (daemon)
  ├─ Reference Handler (daemon)
  └─ Finalizer (daemon)

main()이 종료되어도 → 사용자 스레드가 남아있으면 JVM 계속 실행
                  → 사용자 스레드가 모두 종료되면 JVM 종료
                    (남은 데몬 스레드는 강제 종료됨)
```

이 동작 때문에 다음과 같은 코드는 **즉시 종료되지 않습니다**.

```java
public static void main(String[] args) {
    new Thread(() -> {
        try { Thread.sleep(10000); } catch (InterruptedException e) {}
    }).start();
    // main()은 여기서 끝나지만 JVM은 위 스레드가 끝날 때까지 살아있음
}
```

`setDaemon(true)`로 바꾸면 main이 끝나는 순간 같이 종료됩니다.

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 주제 | 핵심 내용 |
|---|---|
| 프로세스 vs 스레드 | 메모리 공유 여부가 핵심 차이 |
| 동시성 vs 병렬성 | 논리적 동시 실행 vs 물리적 동시 실행 |
| 스레드 생성 5가지 | Thread, Runnable, Callable, ExecutorService, Virtual Thread |
| 스레드 생명주기 | NEW → RUNNABLE → (BLOCKED / WAITING / TIMED_WAITING) → TERMINATED |
| BLOCKED vs WAITING | 락 대기 vs 자발적 대기 |
| 데몬 스레드 | 사용자 스레드 모두 끝나면 강제 종료 |

이번 편에서는 "스레드가 무엇이고 어떻게 만드는가"라는 정적인 그림을 봤습니다.
그런데 한 가지 궁금증이 남습니다.

> CPU 코어 수보다 훨씬 많은 스레드를 만들어도 동시에 돌아가는 것처럼 보이는데, 이게 어떻게 가능한 거지?

이 질문에 답하려면 OS가 스레드를 어떻게 전환하는지를 알아야 합니다.
다음 편에서는 **Context Switching** — 운영체제가 한 CPU에서 수천 개의 스레드를 굴리는 마법을 살펴보겠습니다.