---
title: "EP.05 CAS와 Atomic"
date: "2026-05-17"
episode: 5
tags: ["CAS", "Atomic", "Java", "Concurrency", "Lock-free"]
---

지난 편에서 우리는 `synchronized`와 `volatile`을 통해 **남이 못 들어오게 막는** 방식의 동시성 제어를 배웠습니다.
이번 편에서는 정반대 철학을 살펴봅니다 — **충돌이 나면 다시 시도하는** 방식, 즉 **락 없는 동시성(Lock-free Concurrency)** 입니다.🙂

이 글은 Thread Deep Dive 시리즈의 다섯 번째이자 마지막 편입니다.

## 락의 비용, 그리고 다른 길

`synchronized`는 강력하지만 본질적으로 **블로킹**입니다.

```
스레드 A가 락 획득 → 임계영역 실행 (10µs)
스레드 B는 대기 → BLOCKED 상태 → Context Switch 발생
스레드 A가 락 해제 → 스레드 B를 깨움 → Context Switch 다시 발생
```

이 과정에서 발생하는 비용:

```
락 자체의 비용
  ├─ 모니터 진입/퇴출
  ├─ 경합 시 OS 모니터로 승격 (heavyweight)
  ├─ Context Switching
  └─ 깨우기/큐잉

추가 위험
  ├─ 데드락 (락 획득 순서 어긋나면)
  ├─ 라이브락 (서로 양보만 하다 진행 안 됨)
  └─ 우선순위 역전 (낮은 우선순위 스레드가 락을 쥐고 있어서
                  높은 우선순위 스레드가 못 진행)
```

> "단순히 카운터 하나 1 증가시키는 작업에 이 모든 게 필요한가?"

이 질문에서 출발한 게 **CAS** 입니다.

---

## CAS(Compare-And-Swap)란

CAS는 다음 세 가지 인자를 받는 **CPU 수준의 원자적 명령어**입니다.

```
CAS(memory, expected, new)

  의미: "memory 위치의 현재 값이 expected와 같다면,
        new로 바꿔라. 아니면 아무것도 안 한 채 실패를 알려라.
        이 모든 작업을 원자적으로 수행한다."
```

의사 코드로 표현하면:

```java
// CAS의 의미를 Java처럼 표현하면 (실제로는 CPU 명령 한 줄)
boolean cas(int[] memory, int expected, int newValue) {
    // 이 블록 전체가 원자적으로 실행됨 (CPU가 보장)
    if (memory[0] == expected) {
        memory[0] = newValue;
        return true;
    }
    return false;
}
```

### 왜 락이 필요 없는가

```
일반 read-modify-write:
  1. read   (값 읽기)
  2. modify (계산)        ← 이 사이에 다른 스레드가 끼어들 수 있음
  3. write  (저장)

CAS:
  1. CAS(addr, expected, new)  ← 이 한 단계가 원자적
                                   "내가 본 값이 그대로 있으면 바꿔라"
                                   다른 스레드가 끼어들었으면 실패
```

CAS는 "값이 안 바뀌었음을 확인 + 새 값 저장"을 **CPU 명령어 단 하나**로 처리합니다. 락도, OS도 거치지 않습니다.

---

## CPU 명령어 수준의 지원

CAS는 운영체제의 락처럼 소프트웨어로 구현된 것이 아니라 **CPU 자체가 제공하는 기본 명령어**입니다.

| 아키텍처 | 명령어 |
|---|---|
| **x86 / x86_64** | `CMPXCHG` (Compare and Exchange), 멀티프로세서에서는 `LOCK CMPXCHG` |
| **ARM (구형)** | `LDREX` / `STREX` (Load-Linked / Store-Conditional 페어) |
| **ARMv8.1+** | `CAS`, `CASA` (네이티브 CAS 명령어) |

x86의 `LOCK CMPXCHG`는 다음을 보장합니다:

```
LOCK 접두사 효과
  - 명령어 실행 중 메모리 버스를 잠금 (또는 캐시 라인 잠금)
  - 다른 코어가 이 메모리 위치에 동시 접근 불가
  - 결과적으로 원자성 보장
```

이 하드웨어 지원 덕분에 CAS는 **수 나노초** 수준에서 끝납니다. OS 락(수 µs ~ 수십 µs)과는 비교가 안 되는 속도입니다.

---

## Java에서 CAS — `AtomicInteger`의 내부

Java는 CAS를 직접 다룰 수 있는 도구로 `java.util.concurrent.atomic` 패키지를 제공합니다. 가장 대표적인 `AtomicInteger`를 봅시다.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();   // ++count와 동일하지만 thread-safe
```

### `incrementAndGet()`의 구현 (개념)

```java
public final int incrementAndGet() {
    int current, next;
    do {
        current = get();           // 현재 값 읽기
        next = current + 1;        // 새 값 계산
    } while (!compareAndSet(current, next));  // CAS 시도, 실패하면 재시도
    return next;
}
```

핵심은 **while 루프**입니다.

```
1. 현재 값을 읽는다              (예: 5)
2. +1 한 새 값을 계산한다        (6)
3. CAS(5 → 6) 시도
   ├─ 성공: 종료, 6 반환
   └─ 실패: 다른 스레드가 먼저 바꿨음 → 1번부터 다시
```

### 실제 OpenJDK 코드 (단순화)

내부적으로는 `Unsafe` 또는 (Java 9+) `VarHandle`을 통해 CPU의 CAS 명령어를 호출합니다.

```java
public class AtomicInteger {
    private volatile int value;
    private static final VarHandle VALUE;
    static {
        try {
            VALUE = MethodHandles.lookup()
                .findVarHandle(AtomicInteger.class, "value", int.class);
        } catch (Exception e) { throw new Error(e); }
    }

    public final int incrementAndGet() {
        return (int) VALUE.getAndAdd(this, 1) + 1;
        // VALUE.getAndAdd → 내부적으로 CPU CAS 명령어 사용
    }

    public final boolean compareAndSet(int expected, int newValue) {
        return VALUE.compareAndSet(this, expected, newValue);
    }
}
```

> 💡 `value` 필드가 `volatile`인 점에 주목하세요. 가시성이 보장되어야 다른 스레드의 변경을 정확히 읽을 수 있기 때문입니다. **CAS 알고리즘은 volatile 위에 세워진다**고 보면 됩니다.

---

## CAS 패턴: 직접 구현해보기

`AtomicInteger`로 충분한 경우가 대부분이지만, 더 복잡한 동시 자료구조는 CAS 패턴을 직접 짜기도 합니다.

```java
// 예시: 두 변수를 동시에 일관되게 변경하기 (잔액 + 거래 횟수)
class Account {
    private static class State {
        final long balance;
        final long txCount;
        State(long b, long c) { this.balance = b; this.txCount = c; }
    }

    private final AtomicReference<State> state = 
        new AtomicReference<>(new State(0, 0));

    public void deposit(long amount) {
        State current, next;
        do {
            current = state.get();
            next = new State(current.balance + amount, current.txCount + 1);
        } while (!state.compareAndSet(current, next));
        //         ^ 다른 스레드가 그 사이에 변경했으면 false → 재시도
    }
}
```

핵심 패턴:

```
do {
    1. 현재 상태 스냅샷 읽기
    2. 새 상태 계산 (immutable 객체로)
    3. CAS로 교체 시도
} while (실패하면 재시도);
```

이 패턴을 **CAS 루프** 또는 **optimistic locking** 이라고 합니다.

---

## ABA 문제

CAS의 가장 유명한 함정입니다.

### 시나리오

```
공유 변수 X = A

스레드 1: X 읽음 → A
스레드 1: ... 잠시 다른 일 ...

스레드 2: X 를 A → B 로 변경
스레드 2: X 를 B → A 로 변경 (다시 A로 돌아옴)

스레드 1: CAS(X, A, C)  ← 성공함! 
                          (X가 여전히 A이므로 CAS는 통과)
                          
하지만 그 사이 X가 B를 거쳤다는 사실이 무시됨
→ 어떤 알고리즘에서는 이게 치명적인 버그
```

### 실제로 문제가 되는 예 — Lock-free Stack

```
초기 스택: A → B → C
top = A

스레드 1: pop()
  - top 읽음 → A
  - A.next 읽음 → B
  - CAS(top, A → B) 준비

스레드 2: pop() pop() push(A)
  - A pop, B pop, C가 top
  - A를 다시 push
  - 이제 스택: A → C
  - top = A

스레드 1: CAS(top, A → B) 실행
  - 성공! (top이 여전히 A니까)
  - 그런데 B는 이미 스택에 없음!
  - top이 사라진 노드 B를 가리킴 → 메모리 깨짐
```

### 해결: 버전 번호 추가

값과 함께 **버전(stamp)** 을 묶어 CAS합니다. 같은 값으로 돌아왔어도 버전이 다르면 실패합니다.

```java
AtomicStampedReference<String> ref = 
    new AtomicStampedReference<>("A", 0);

// 읽기
int[] stampHolder = new int[1];
String value = ref.get(stampHolder);
int stamp = stampHolder[0];

// CAS — 값과 stamp 모두 일치해야 성공
boolean success = ref.compareAndSet(value, "C", stamp, stamp + 1);
```

| 클래스 | 용도 |
|---|---|
| `AtomicStampedReference` | 정수 stamp를 함께 비교 (버전 카운터) |
| `AtomicMarkableReference` | boolean mark를 함께 비교 (delete 표시 등) |

---

## CAS의 장단점

| 장점 | 단점 |
|---|---|
| 락 없음 → 데드락 없음 | 경합이 심하면 재시도가 늘어 CPU 낭비 |
| Context Switching 없음 | ABA 문제 |
| 매우 빠름 (수 ns) | 단일 변수 단위 — 복합 연산은 어려움 |
| 우선순위 역전 없음 | 코드 작성이 까다로움 (lock-free 알고리즘은 정말 어렵다) |

### 경합이 심하면 어떻게 되는가

```
경합이 약할 때 (CAS 거의 한 번에 성공)
  → CAS 압승 (locks: 수 µs vs CAS: 수 ns)

경합이 심할 때 (CAS가 자주 실패해 재시도 폭증)
  → 재시도 루프가 CPU를 갉아먹음
  → synchronized보다 오히려 느려질 수 있음
```

이 문제를 완화하기 위해 Java는 `LongAdder` 같은 클래스를 추가했습니다.

---

## Atomic 클래스 패밀리 한눈에 보기

| 클래스 | 용도 |
|---|---|
| `AtomicBoolean` | 원자적 boolean |
| `AtomicInteger` / `AtomicLong` | 원자적 정수 |
| `AtomicReference<V>` | 원자적 객체 참조 |
| `AtomicStampedReference<V>` | ABA 방지용 (stamp) |
| `AtomicMarkableReference<V>` | ABA 방지용 (mark) |
| `AtomicIntegerArray` 등 | 배열의 각 원소를 원자적으로 |
| `AtomicIntegerFieldUpdater` 등 | 기존 필드를 원자적으로 |
| **`LongAdder` / `LongAccumulator`** | **고경합 환경용 카운터** |

### `LongAdder` — 고경합 환경의 해결책

```java
// 일반 카운터 (경합 시 CAS 재시도 폭증)
AtomicLong counter1 = new AtomicLong();

// LongAdder (경합을 분산)
LongAdder counter2 = new LongAdder();

// 사용법은 거의 동일
counter1.incrementAndGet();
counter2.increment();

// 합계 읽을 때
long sum1 = counter1.get();
long sum2 = counter2.sum();   // 내부 셀들의 합 계산
```

`LongAdder`는 내부적으로 **여러 개의 셀(Cell)** 을 두고 스레드별로 다른 셀에 더합니다. 합계를 읽을 때만 모든 셀을 더하므로 쓰기 경합이 분산됩니다.

```
AtomicLong:
  모든 스레드  ──→  하나의 long 변수  (CAS 경합 폭증)

LongAdder:
  Thread1 ──→  Cell[0]
  Thread2 ──→  Cell[1]            (각자 다른 셀에 더함, 경합 분산)
  Thread3 ──→  Cell[2]
                  │
            sum() 호출 시
                  ▼
            Cell들을 모두 더함
```

| 시나리오 | 권장 |
|---|---|
| 읽기가 매우 빈번, 쓰기 적음 | `AtomicLong` |
| 쓰기 경합이 심함 (멀티코어 카운터) | `LongAdder` |

---

## synchronized vs Atomic — 선택 기준

EP.04와 이번 편의 도구를 종합한 의사결정 가이드입니다.

```
무엇을 보호해야 하는가?
  │
  ├─ 단일 변수의 단순 연산 (++ -- set/get)
  │     → AtomicInteger / AtomicLong / AtomicReference
  │
  ├─ 단일 변수, 그러나 매우 고경합
  │     → LongAdder / LongAccumulator
  │
  ├─ 여러 변수를 함께 일관되게 변경
  │     → synchronized 또는 ReentrantLock
  │     → 또는 AtomicReference로 immutable 객체 통째로 교체
  │
  ├─ 단순한 가시성 보장만 필요
  │     → volatile
  │
  └─ 복잡한 임계영역 / 조건 변수가 필요
        → ReentrantLock + Condition
```

### 성능 직관

```
저경합 환경:
  Atomic ≈ synchronized (JVM 최적화 덕분)

고경합 + 단순 연산:
  Atomic > synchronized

고경합 + 복합 연산:
  synchronized > Atomic (CAS 재시도 폭증)

매우 고경합 + 단순 카운터:
  LongAdder > AtomicLong > synchronized
```

> 💡 **항상 측정하세요.** "Atomic이 빠르다"는 직관은 저경합/단순 연산에서만 성립합니다. JMH(Java Microbenchmark Harness)로 실제 워크로드를 측정하는 게 안전합니다.

---

## 락 vs 락 없음 — 두 철학의 비교

| 항목 | 락 기반 (synchronized, ReentrantLock) | 락 없음 (CAS, Atomic) |
|---|---|---|
| **방식** | 진입을 막음 (Blocking) | 충돌 시 재시도 (Optimistic) |
| **상태 변화 시 다른 스레드** | BLOCKED 상태로 대기 | RUNNABLE 유지, 재시도 |
| **데드락** | 가능 | 불가능 |
| **단일 연산 성능** | 좋음 (저경합 시) | 매우 좋음 |
| **복합 연산** | 자연스러움 | 까다로움 (CAS 루프 + immutable) |
| **공정성** | `ReentrantLock(true)`로 가능 | 보장 어려움 |

두 방식은 **경쟁 관계가 아니라 보완 관계**입니다. 실무 코드에서는 거의 항상 둘 다 등장합니다.

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 주제 | 핵심 내용 |
|---|---|
| 락의 비용 | 블로킹, Context Switch, 데드락 위험 |
| CAS | "값이 안 바뀌었으면 바꿔라"를 원자적으로 — CPU 명령어 |
| AtomicInteger 내부 | `volatile + CAS 루프` |
| CAS 루프 패턴 | `do { read; compute; } while (!cas(...))` |
| ABA 문제 | 같은 값이지만 거쳐온 경로가 다름 → 잘못된 성공 |
| 해결책 | `AtomicStampedReference` 등으로 버전 추가 |
| Atomic 패밀리 | Boolean / Integer / Reference / Stamped / **LongAdder** |
| LongAdder | 고경합 환경에서 셀 분산으로 쓰기 경합 완화 |
| 선택 기준 | 단순 → Atomic, 복합 → synchronized, 고경합 카운터 → LongAdder |

---

## 시리즈 전체 회고

**Thread Deep Dive** 5편을 통해 우리는 다음을 함께 살펴봤습니다.

| 편 | 주제 | 핵심 |
|---|---|---|
| EP.01 | 스레드와 프로세스 | 메모리 공유 여부, 생성 방법, 생명주기 6단계 |
| EP.02 | Context Switching | OS가 스레드를 전환하는 비용과 시점 |
| EP.03 | JVM의 스레드 모델 | 1:1 매핑, Virtual Thread의 M:N 회귀 |
| EP.04 | JMM과 락 기반 동기화 | volatile, synchronized, happens-before, JEP 491 |
| EP.05 | CAS와 Atomic | 락 없는 동시성, ABA, LongAdder |

JVM 시리즈에서 만났던 "스레드 전용 영역(JVM Stack, PC Register, Native Stack)"이 EP.03에서 OS 스레드와 1:1로 연결되었고, "스레드 공유 영역(Heap, Method Area)"이 EP.04에서 동기화 문제의 출발점이 되었습니다. 두 시리즈가 한 그림으로 연결되는 셈입니다.

```
JVM 메모리 구조 (Series 1)
        │
        │  스레드 전용 / 공유 영역 구분
        ▼
스레드 모델 (Series 2)
        │
        │  공유 영역에서 발생하는 문제
        ▼
동기화 (volatile, synchronized, CAS)
        │
        │  ?
        ▼
???
```

마지막 화살표 끝에는 무엇이 있을까요? 다음 시리즈로 자연스럽게 이어진다면 후보는 다음과 같습니다.

- **ThreadPool & ExecutorService Deep Dive** — `ThreadPoolExecutor` 내부, Spring `@Async`, `RejectedExecutionHandler`
- **Java Concurrent Collections** — `ConcurrentHashMap`, `BlockingQueue` 내부 동작
- **CompletableFuture & 비동기 프로그래밍**
- **Virtual Thread Deep Dive** — Carrier, Continuation, Structured Concurrency
- **Reactive Programming** — WebFlux, Project Reactor

읽어주신 모든 분께 감사드립니다. 다음 시리즈에서 또 만나요.👋