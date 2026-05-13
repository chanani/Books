---
title: "EP.03 JVM의 스레드 모델"
date: "2026-05-13"
episode: 3
tags: [ "JVM", "Thread", "Java", "Virtual Thread" ]
---

지난 편에서 OS가 스레드를 어떻게 전환하는지 살펴봤습니다.
그렇다면 Java 코드에서 `new Thread().start()`를 호출하면 실제로 OS와 어떤 일이 벌어질까요? JVM 스레드와 OS 스레드는 어떤 관계일까요?🙂

이 글은 Thread Deep Dive 시리즈의 세 번째 편입니다.

## 스레드 매핑 모델 — 3가지 방식

가상 머신 또는 런타임이 자신의 스레드를 OS 스레드와 연결하는 방법은 크게 세 가지가 있습니다.

```
┌───────────────────────────────────────────────────────────┐
│  ① 1:1 (Native Thread / Kernel-Level)                     │
│                                                           │
│   Runtime Thread  ──→  OS Thread  (1:1)                   │
│                                                           │
│   장점: OS 스케줄러 활용, 멀티코어 활용 쉬움                       │
│   단점: 스레드 생성 비용 큼, 개수 제한                            │
│   채택: HotSpot JVM (현재까지 기본 모델)                        │
├───────────────────────────────────────────────────────────┤
│  ② N:1 (Green Thread / User-Level)                        │
│                                                           │
│   Runtime Thread × N  ──→  OS Thread × 1                  │
│                                                           │
│   장점: 매우 가벼움                                           │
│   단점: 멀티코어 못 씀, I/O 블로킹이 전체를 막음                   │
│   채택: 초기 Java(JDK 1.1까지), 일부 코루틴 라이브러리             │
├───────────────────────────────────────────────────────────┤
│  ③ M:N (Hybrid)                                           │
│                                                           │
│   Runtime Thread × M  ──→  OS Thread × N  (M ≫ N)         │
│                                                           │
│   장점: 가벼움 + 멀티코어 활용                                  │
│   단점: 구현이 복잡                                           │
│   채택: Go의 goroutine, Java 21+ Virtual Thread             │
└───────────────────────────────────────────────────────────┘
```

| 모델      | Runtime ↔ OS | 멀티코어 | 가벼움 | 대표 예                           |
|---------|--------------|------|-----|--------------------------------|
| **1:1** | 1:1          | ✅    | ❌   | HotSpot Platform Thread        |
| **N:1** | N:1          | ❌    | ✅   | Green Thread (옛날 Java), 일부 코루틴 |
| **M:N** | M:N          | ✅    | ✅   | Goroutine, Virtual Thread      |

---

## HotSpot JVM의 1:1 매핑

우리가 흔히 쓰는 OpenJDK / Oracle JDK의 HotSpot JVM은 **1:1 매핑**을 사용합니다.

```
Java 애플리케이션
  │
  │  new Thread(runnable).start()
  ▼
┌──────────────────┐
│ java.lang.Thread │  (Java 객체)
└────────┬─────────┘
         │
         │  JNI 통해 네이티브 호출
         ▼
┌──────────────────┐
│    JavaThread    │  (JVM 내부 C++ 객체)
└────────┬─────────┘
         │
         │  OS API 호출
         │   - Linux: pthread_create()
         │   - Windows: CreateThread()
         │   - macOS: pthread_create()
         ▼
┌──────────────────┐
│     OS Thread    │  (커널이 관리하는 진짜 스레드)
└──────────────────┘
```

`Thread.start()`를 호출할 때마다 **OS에 시스템 콜이 들어가 새로운 OS 스레드가 만들어집니다**.

- Java의 스레드 스케줄링 = OS 스케줄링 그 자체
- Java 스레드 우선순위(`setPriority`) = OS에 힌트로 전달 (반드시 지켜지진 않음)
- Java 스레드 컨텍스트 스위칭 = OS 컨텍스트 스위칭

EP.02에서 본 Context Switching 비용이 그대로 Java 스레드에 적용된다는 뜻입니다.

---

## JVM 안에서 돌아가는 스레드들

Java 애플리케이션을 실행하면 우리가 만든 스레드 외에도 JVM이 자체적으로 여러 스레드를 운영합니다.

```
JVM 프로세스 내부

┌────────────────────────────────────────────────────┐
│                    사용자 스레드                      │
│  - main                                            │
│  - 우리가 만든 스레드들 (워커 풀, @Async 등)              │
└────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│                JVM 내부 시스템 스레드 (대부분 daemon)     │
│  - VM Thread          : VMOperation 처리 (GC 트리거 등)│
│  - GC Threads         : 가비지 컬렉션 수행              │
│  - Compiler Threads   : JIT 컴파일 (C1, C2)          │
│  - Reference Handler  : Reference 객체 처리           │
│  - Finalizer          : finalize() 메서드 호출         │
│  - Signal Dispatcher  : OS 시그널 처리                 │
│  - Attach Listener    : jstack, jmap 등 도구 응답      │
└─────────────────────────────────────────────────────┘
```

`jcmd <pid> Thread.print` 또는 `jstack <pid>` 명령으로 실시간 확인이 가능합니다.

```bash
$ jstack 12345 | head -30

"main" #1 prio=5 os_prio=0 cpu=12.34ms ...
   java.lang.Thread.State: RUNNABLE
   ...

"Reference Handler" #2 daemon prio=10 os_prio=0 ...
   java.lang.Thread.State: RUNNABLE

"Finalizer" #3 daemon prio=8 os_prio=0 ...
   java.lang.Thread.State: WAITING

"GC Thread#0" os_prio=0 ...
"C2 CompilerThread0" #5 daemon prio=9 ...
```

`#` 다음 번호는 JVM이 매기는 스레드 ID, `os_prio`는 OS 우선순위, `prio`는 Java 우선순위입니다.

---

## JVM Stack과 OS 스레드의 관계

JVM 시리즈 EP.02에서 다룬 메모리 구조와 이번 주제를 연결할 수 있습니다.

```
새 스레드 생성 시 일어나는 일

  new Thread(runnable).start()
        │
        ▼
  ┌───────────────────────────────────────────┐
  │  ① OS에 스레드 생성 요청 (pthread_create)     │
  │     - OS가 새 스레드용 Native Stack 할당       │
  │                                           │
  │  ② JVM이 새 스레드를 위한 영역 초기화            │
  │     - JVM Stack 할당 (-Xss로 크기 지정)       │
  │     - PC Register 할당                     │
  │     - Native Method Stack 할당 (Native와    │
  │       동일 영역을 공유하는 구현이 많음)           │
  │                                           │
  │  ③ Method Area, Heap은 모든 스레드가 공유      │
  └───────────────────────────────────────────┘
```

EP.02에서 **JVM Stack, PC Register, Native Method Stack이 "스레드 전용"** 이었던 이유가 여기서 명확해집니다. 스레드 자체가 OS 수준의 실행 단위이고, 각 스레드가
자기만의 호출 스택을 가져야 하기 때문입니다.

### `-Xss` 옵션이 의미하는 것

```bash
$ java -Xss512k MyApp
```

이 옵션은 **각 스레드의 JVM Stack 크기**를 지정합니다. 1000개의 스레드를 만들면 단순 계산으로 `512KB × 1000 = 500MB`가 스택용으로만 필요합니다. 이게 1:1 모델에서 스레드를 무한정
만들 수 없는 가장 직접적인 이유 중 하나입니다.

| `-Xss` 기본값 | 환경                   |
|------------|----------------------|
| 512KB      | 32-bit Linux/Windows |
| 1MB        | 64-bit Linux/macOS   |
| 1MB        | 64-bit Windows       |

---

## 1:1 모델의 한계

1:1 모델은 단순하고 OS의 모든 기능을 그대로 활용할 수 있다는 장점이 있지만, 현대 웹 서비스의 요구사항에는 한계가 있습니다.

```
한계 1: 스레드 생성 비용
  - 시스템 콜로 OS 스레드 생성 → 수 µs ~ 수십 µs
  - 그래서 Thread Pool로 재사용해야 함

한계 2: 메모리 비용
  - 스레드당 Stack 1MB → 1만 개 만들면 10GB
  - 실제로는 OS 한계로 수천 개 정도가 한계

한계 3: Context Switching 비용
  - 스레드 수가 많아지면 스케줄러 오버헤드 폭증
  - 어느 지점부터는 처리량이 오히려 감소

한계 4: Thread-per-Request 모델의 비효율
  - 웹 요청 1개마다 스레드 1개 할당하는 모델
  - I/O 대기 시간 동안 스레드는 그냥 놀고 있음
  - 스레드는 비싼데, 대부분 시간을 대기로 보냄
```

이 4번 한계가 특히 큽니다. Spring MVC의 일반적인 요청은 다음과 같은 흐름인데:

```
요청 처리 시간 (총 100ms)
  │
  ├─ 비즈니스 로직 (CPU 작업): 5ms
  ├─ DB 쿼리 대기:            70ms   ← 이 동안 스레드는 놀고 있음
  ├─ 외부 API 대기:           20ms   ← 이 동안에도 놀고 있음
  └─ 응답 직렬화:              5ms

→ 100ms 중 95ms를 그냥 대기로 보냄
→ 비싼 OS 스레드가 95% 시간을 놀고 있는 셈
```

이 비효율을 해결하기 위해 두 가지 접근이 등장했습니다.

| 접근                        | 방식                       | 단점                 |
|---------------------------|--------------------------|--------------------|
| **리액티브** (WebFlux 등)      | 콜백/이벤트 루프, 적은 수의 스레드로 처리 | 코드 복잡도 폭증, 디버깅 어려움 |
| **가상 스레드** (Project Loom) | 스레드는 그대로 쓰되 가볍게 만듦       | JDK 21+ 필요         |

---

## Virtual Thread — M:N 모델의 부활

Java 21에서 정식 도입된 Virtual Thread는 다시 **M:N 매핑**을 채택합니다.

```
일반 스레드 (Platform Thread)

  Java Thread ────── OS Thread     ← 1:1, 비쌈
  
  
Virtual Thread

  Virtual Thread (수만 개)
       │
       │  스케줄링 (JVM 사용자 모드에서)
       ▼
  Carrier Thread (수~수십 개)       ← 보통 CPU 코어 수
       │
       │  1:1
       ▼
  OS Thread
```

핵심 개념:

- **Virtual Thread:** JVM이 관리하는 가벼운 스레드. Stack은 Heap에 할당됨.
- **Carrier Thread:** Virtual Thread를 실제로 실행해주는 Platform Thread. 캐리어 풀의 기본 크기는 CPU 코어 수.
- **마운트(Mount) / 언마운트(Unmount):** Virtual Thread가 Carrier에 올라타고 내려오는 과정.

### 왜 가벼운가?

```
Virtual Thread는 "OS 스레드가 아니다"
  → Stack을 Heap에 가변 크기로 저장 (몇 KB부터 시작)
  → 수십만 개를 만들어도 메모리 문제 없음

Virtual Thread 간 전환은 JVM이 사용자 모드에서 처리
  → Mode Switch 없음
  → 메모리 맵 전환 없음
  → Context Switching 비용이 OS 스레드보다 훨씬 저렴
```

EP.02에서 본 Context Switching의 진짜 비용(Mode Switch + 캐시/TLB 무효화)이 Virtual Thread 간 전환에서는 거의 발생하지 않습니다.

### 사용 예

```java
// 단발성
Thread.startVirtualThread(() -> doWork());

// Executor로
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i ->
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        })
    );
}   // try-with-resources로 자동 shutdown
```

위 코드는 1만 개의 Virtual Thread를 생성합니다. Platform Thread로 동일하게 시도하면 OS 한계에 부딪히지만, Virtual Thread는 정상 동작합니다.

### Virtual Thread의 핀닝(Pinning) 문제와 JEP 491

Java 21~23에서는 한 가지 큰 문제가 있었습니다.

> Virtual Thread가 `synchronized` 블록 안에서 블로킹 I/O를 만나면 캐리어 스레드에 **핀(pin)** 되어버려, 그 캐리어가 다른 Virtual Thread를 받지 못함.

이 문제는 Java 24의 **JEP 491(Synchronize Virtual Threads without Pinning)** 로 해결되었습니다. 모니터를 캐리어가 아닌 Virtual Thread 자체에 연결하도록
JVM을 수정한 것입니다.

이 부분은 EP.04에서 `synchronized`를 다룰 때 다시 짚어볼 예정입니다.

### 언제 Virtual Thread를 써야 하나?

| 적합                      | 부적합                        |
|-------------------------|----------------------------|
| Thread-per-Request 웹 서버 | CPU 집약 작업 (이건 코어 수만큼만 필요)  |
| I/O 대기가 많은 작업           | 스레드 풀로 동시 실행 수를 제한해야 하는 경우 |
| 외부 API 호출 fan-out       | 짧고 단순한 작업 (오버헤드 대비 이득 없음)  |

> ⚠️ **Virtual Thread는 풀링하지 않습니다.** 가볍기 때문에 매번 생성/폐기하는 게 정상입니다. 동시성 제한이 필요하면 풀 대신 `Semaphore`를 사용합니다.

---

## 스레드 덤프(Thread Dump) 분석

운영 환경 트러블슈팅에서 가장 자주 쓰는 도구입니다. 모든 스레드의 현재 호출 스택을 한 번에 캡처합니다.

### 덤프 뜨는 방법

```bash
# 방법 1: jstack
$ jstack <pid> > thread-dump.txt

# 방법 2: jcmd
$ jcmd <pid> Thread.print > thread-dump.txt

# 방법 3: kill -3 (Unix)
$ kill -3 <pid>     # 표준출력에 덤프됨
```

### 읽는 법

```
"http-nio-8080-exec-3" #45 daemon prio=5 os_prio=0 cpu=234.56ms ...
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.Service.process(Service.java:42)
        - waiting to lock <0x000000076b8c4a90> (a java.lang.Object)
        at com.example.Controller.handle(Controller.java:18)
        ...

"http-nio-8080-exec-7" #49 daemon prio=5 os_prio=0 ...
   java.lang.Thread.State: RUNNABLE
        at com.example.Service.process(Service.java:55)
        - locked <0x000000076b8c4a90> (a java.lang.Object)   ← 이 스레드가 락을 쥐고 있음
        ...
```

핵심 포인트:

- `Thread.State`에 `BLOCKED`가 많으면 락 경합 의심
- `waiting to lock <주소>`와 같은 주소를 `locked <주소>`로 쥐고 있는 스레드 페어를 찾으면 락 보유자 식별 가능
- 두 스레드가 서로의 락을 기다리고 있으면 **데드락** (jstack은 데드락을 자동 감지해 알려주기도 함)

이 분석을 제대로 하려면 EP.04에서 다룰 모니터 락 구조를 알아야 합니다.

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 주제             | 핵심 내용                                                 |
|----------------|-------------------------------------------------------|
| 스레드 매핑 모델      | 1:1 (Native), N:1 (Green), M:N (Hybrid)               |
| HotSpot JVM    | 기본은 1:1 — Java 스레드 = OS 스레드                           |
| JVM 내부 스레드     | main 외에 GC, JIT, Reference Handler 등                  |
| `-Xss`         | 각 스레드의 JVM Stack 크기, 1:1 모델의 한계 원인                    |
| 1:1 모델의 한계     | 생성 비용, 메모리, Context Switching, Thread-per-Request 비효율 |
| Virtual Thread | M:N 모델로 회귀, Carrier Thread에 마운트/언마운트                  |
| 핀닝과 JEP 491    | Java 24에서 synchronized 핀닝 해결                          |
| 스레드 덤프         | 트러블슈팅의 핵심 도구, BLOCKED와 락 페어 분석                        |

여기까지 우리는 "스레드가 무엇이고 어떻게 동작하는가"를 OS와 JVM 관점에서 모두 살펴봤습니다.

이제 본격적인 멀티스레드 프로그래밍의 위험 지대로 들어갈 차례입니다.
여러 스레드가 같은 메모리(Heap, Method Area)를 공유한다는 사실이 어떤 문제를 만들어내는지, 그리고 Java가 이를 어떻게 풀어내는지를 봐야 합니다.

다음 편에서는 **Java Memory Model과 락 기반 동기화** 를 다룹니다. `volatile`, `synchronized`, happens-before 
이름은 들어봤지만 정확히 뭘 보장하는지는 모호한 그 키워드들을 깊이 파헤쳐보겠습니다.