---
title: "EP.05 JVM 모니터링, 튜닝, 그리고 미래"
date: "2026-04-08"
episode: 5
tags: [ "JVM", "Java", "Monitoring", "Tuning", "GraalVM", "Virtual Thread", "AOT" ]
---

지난 4편에 걸쳐 JVM의 ClassLoader, 메모리 구조, GC, 실행 엔진, 스레드까지 내부를 깊이 파헤쳤습니다.
이번 마지막 편에서는 지금까지 배운 지식을 실전에서 활용하는 방법, 그리고 JVM이 앞으로 어디로 향하는지 살펴보겠습니다.🙂

이 글은 JVM Deep Dive 시리즈의 마지막 편입니다.

---

## JVM 모니터링이 필요한 순간

JVM 내부를 아무리 잘 알아도, 실제 운영 환경에서 문제가 생겼을 때 진단하는 능력이 없다면 절반만 아는 것입니다.

다음과 같은 상황에서 JVM 모니터링이 필요합니다.

```
🚨 이런 증상이 나타나면 JVM을 들여다봐야 합니다

  - 서버가 갑자기 느려졌다
  - 특정 시간대에 응답 지연이 생긴다 (→ GC 의심)
  - 메모리 사용량이 계속 증가한다 (→ 메모리 누수 의심)
  - OutOfMemoryError가 발생했다
  - CPU 사용률이 갑자기 치솟는다
  - 애플리케이션이 아무 이유 없이 멈춘다 (→ 데드락 의심)
```

문제를 진단하는 흐름은 일반적으로 이렇습니다.

```
증상 파악
    │
    ▼
JVM 지표 확인 (Heap, GC, Thread)
    │
    ▼
GC 로그 / Thread Dump / Heap Dump 수집
    │
    ▼
원인 분석
    │
    ▼
튜닝 (JVM 옵션 조정 또는 코드 수정)
    │
    ▼
 결과 검증
```

---

## JVM 핵심 지표 — 무엇을 봐야 하나?

모니터링에서 가장 먼저 봐야 할 JVM 핵심 지표입니다.

### Heap 사용량

```
정상 패턴:
Heap ▲
use  │  /\/\/\/\/\/\  ← GC 이후 일정 수준으로 회복
     │ /            \
     └──────────────────→ 시간

메모리 누수 의심 패턴:
Heap ▲
use  │         /
     │      /
     │   /             ← GC 이후에도 계속 증가
     │/
     └──────────────────→ 시간
```

### GC 지표

```
확인할 항목:
  - GC 발생 빈도      : Minor GC는 자주 발생해도 괜찮음, Full GC는 드물어야 함
  - GC STW 시간      : 응답 시간 요구사항보다 짧아야 함
  - GC 후 Heap 회수량 : GC 후에도 Heap이 줄지 않으면 메모리 누수 의심
```

### 스레드 지표

```
확인할 항목:
  - 전체 스레드 수          : 갑자기 증가하면 스레드 누수 의심
  - BLOCKED 상태 스레드 수  : 많으면 lock 경합 또는 데드락 의심
  - WAITING 상태 스레드 수  : 외부 I/O 대기 상태 파악
```

---

## 모니터링 도구 — JFR, JVisualVM, JConsole

### JFR (Java Flight Recorder)

JDK에 내장된 **가장 강력한 프로파일링 도구**입니다. 애플리케이션 성능에 거의 영향을 주지 않으면서 JVM 내부 이벤트를 기록합니다.

```bash
# 60초 동안 JFR 기록 (JDK 11+)
$ java -XX:StartFlightRecording=duration=60s,filename=recording.jfr MyApp

# 실행 중인 프로세스에 JFR 시작
$ jcmd <PID> JFR.start duration=60s filename=recording.jfr

# 기록 저장
$ jcmd <PID> JFR.dump filename=recording.jfr
```

기록된 `.jfr` 파일은 **JDK Mission Control(JMC)** 로 열어서 시각적으로 분석합니다.

```
JFR로 확인 가능한 항목:
  ✅ GC 발생 시간, 빈도, STW 시간
  ✅ CPU 사용률, 스레드 상태
  ✅ 메서드 실행 시간 (핫스팟 파악)
  ✅ 메모리 할당 패턴
  ✅ I/O 이벤트
  ✅ JIT 컴파일 현황
```

### JVisualVM

GUI 기반의 모니터링 도구로 실시간 Heap, CPU, 스레드 상태를 시각적으로 확인할 수 있습니다.

```bash
# JDK에 포함 (JDK 8까지는 기본 포함, 이후 별도 설치)
$ jvisualvm
```

```
주요 기능:
  - 실시간 Heap / CPU / 스레드 그래프
  - Heap Dump 생성 및 분석
  - Thread Dump 생성
  - 프로파일러 (메서드별 실행 시간)
  - 원격 JVM 연결 (JMX)
```

### JConsole

JDK에 기본 포함된 경량 모니터링 도구입니다. JVisualVM보다 단순하지만 빠르게 현황을 파악할 때 유용합니다.

```bash
$ jconsole
```

### jcmd — 커맨드라인 만능 도구

GUI 없이 커맨드라인만으로 JVM을 진단할 때 유용합니다.

```bash
# 실행 중인 JVM 프로세스 목록
$ jcmd

# 사용 가능한 명령어 목록
$ jcmd <PID> help

# Heap 정보 확인
$ jcmd <PID> GC.heap_info

# Thread Dump 생성
$ jcmd <PID> Thread.print

# Heap Dump 생성
$ jcmd <PID> GC.heap_dump filename=heap.hprof

# GC 강제 실행
$ jcmd <PID> GC.run
```

### jstat — GC 실시간 모니터링

```bash
# 1초마다 GC 통계 출력
$ jstat -gcutil <PID> 1000

출력 예시:
  S0      S1     E      O      M     CCS      YGC   YGCT     FGC   FGCT      GCT
  0.00  42.31  73.54  28.12  95.21  92.43     12    0.234     1    0.456    0.690

  S0/S1  : Survivor 0/1 사용률 (%)
  E      : Eden 사용률 (%)
  O      : Old Generation 사용률 (%)
  YGC    : Minor GC 횟수
  YGCT   : Minor GC 총 소요 시간 (초)
  FGC    : Full GC 횟수
  FGCT   : Full GC 총 소요 시간 (초)
```

---

## Heap Dump 분석

**Heap Dump**는 특정 시점의 Heap 전체 상태를 파일로 저장한 스냅샷입니다. OutOfMemoryError 발생 원인을 찾을 때 가장 강력한 무기입니다.

### Heap Dump 생성

```bash
# OOM 발생 시 자동 생성
$ java -XX:+HeapDumpOnOutOfMemoryError \
       -XX:HeapDumpPath=/logs/heap.hprof \
       MyApp

# 수동으로 생성
$ jcmd <PID> GC.heap_dump filename=/logs/heap.hprof

# jmap으로 생성
$ jmap -dump:format=b,file=/logs/heap.hprof <PID>
```

### Eclipse MAT (Memory Analyzer Tool)으로 분석

Heap Dump 분석에 가장 많이 쓰이는 도구입니다.

```
주요 분석 기능:

1. Leak Suspects Report (누수 의심 자동 분석)
   → MAT가 자동으로 메모리를 많이 점유한 객체를 찾아줌

2. Dominator Tree (지배자 트리)
   → 어떤 객체가 가장 많은 메모리를 보유하고 있는지 확인
   → "이 객체 하나가 사라지면 얼마나 많은 메모리가 해제되는가"

3. Retained Heap vs Shallow Heap
   Shallow Heap  : 객체 자체의 크기
   Retained Heap : 이 객체가 GC되면 함께 해제될 수 있는 전체 크기

4. OQL (Object Query Language)
   → SQL처럼 Heap에서 객체를 쿼리
   SELECT * FROM java.util.ArrayList WHERE size > 10000
```

### Heap Dump 분석 흐름

```
Heap Dump (.hprof) 생성
         │
         ▼
MAT로 Leak Suspects 보고서 확인
         │
         ▼
Dominator Tree에서 메모리 점유 TOP 객체 파악
         │
         ▼
해당 객체의 참조 경로 (GC Root까지) 추적
         │
         ▼
코드에서 해당 객체가 해제되지 않는 원인 찾기
         │
         ▼
     수정 및 검증
```

---

## GC 튜닝 옵션 총정리

### Heap 크기 설정

```bash
-Xms2g                  # Heap 초기 크기 (2GB)
-Xmx4g                  # Heap 최대 크기 (4GB)
# 팁: -Xms와 -Xmx를 같게 설정하면 동적 조정 오버헤드 제거
```

### GC 선택

```bash
-XX:+UseSerialGC         # Serial GC
-XX:+UseParallelGC       # Parallel GC
-XX:+UseG1GC             # G1 GC (Java 9+ 기본)
-XX:+UseZGC              # ZGC (Java 15+ 안정화)
-XX:+UseShenandoahGC     # Shenandoah GC
```

### G1 GC 튜닝

```bash
-XX:MaxGCPauseMillis=200     # 목표 STW 시간 (ms) — 기본 200
-XX:G1HeapRegionSize=16m     # Region 크기 (1~32MB, 2의 제곱수)
-XX:G1NewSizePercent=20      # Young Gen 최소 비율 (%)
-XX:G1MaxNewSizePercent=60   # Young Gen 최대 비율 (%)
-XX:InitiatingHeapOccupancyPercent=45  # Old Gen GC 시작 임계값 (%)
```

### Metaspace 설정

```bash
-XX:MetaspaceSize=128m       # Metaspace 초기 크기
-XX:MaxMetaspaceSize=256m    # Metaspace 최대 크기 (설정 권장)
```

### GC 로그 설정

```bash
# Java 11+
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=20m

# 옵션 설명:
#   filecount=5   : 로그 파일 최대 5개 (롤링)
#   filesize=20m  : 파일당 최대 20MB
```

### JVM 튜닝 체크리스트

```
✅ -Xms == -Xmx 로 설정 (동적 조정 오버헤드 제거)
✅ MaxMetaspaceSize 명시 (무제한 확장 방지)
✅ GC 로그 항상 활성화 (문제 발생 시 분석 가능)
✅ HeapDumpOnOutOfMemoryError 설정 (OOM 발생 시 자동 덤프)
✅ 애플리케이션 특성에 맞는 GC 선택
   - 대용량 배치 → Parallel GC
   - 일반 웹 서버 → G1 GC
   - 초저지연 요구 → ZGC
```

---

## 메모리 누수 진단 패턴

JVM의 GC가 있어도 메모리 누수는 발생할 수 있습니다. GC는 **참조가 살아있는 객체는 절대 수거하지 않기** 때문입니다.

### 흔한 메모리 누수 패턴

**1. static 컬렉션에 계속 추가하는 경우**

```java
// 위험 패턴
public class Cache {
    private static final List<Data> cache = new ArrayList<>();

    public void add(Data data) {
        cache.add(data);  // 제거 로직 없음 → 무한 증가
    }
}
```

**2. 리스너 / 콜백 등록 후 해제 누락**

```java
// 위험 패턴
eventBus.register(listener);   // 등록
// ... 사용 후
// eventBus.unregister(listener); ← 깜빡하고 빼먹으면 누수
```

**3. ThreadLocal 해제 누락**

```java
// 위험 패턴 (스레드 풀 환경)
private static ThreadLocal<Object> local = new ThreadLocal<>();

public void process() {
    local.set(new HeavyObject());
    // ... 작업 수행
    // local.remove(); ← 빠지면 스레드 재사용 시 객체가 남아있음
}
```

**4. 내부 클래스가 외부 클래스를 암묵적으로 참조**

```java
// 위험 패턴
public class Outer {
    private byte[] largeData = new byte[1024 * 1024];  // 1MB

    class Inner {  // 내부 클래스는 Outer를 암묵적으로 참조
        // Inner가 살아있으면 Outer도 GC 안 됨 → largeData도 유지
    }
}
```

### 누수 확인 방법

```bash
# Heap 사용량 추이 확인 (jstat로 지속 모니터링)
$ jstat -gcutil <PID> 5000 20
# 5초마다 20번 출력 → Old Gen(O) 사용률이 계속 오르면 누수 의심

# Heap Dump 두 번 비교
# 1차 덤프: 정상 동작 중
# 2차 덤프: 일정 시간 후
# MAT의 "Compare Snapshots" 기능으로 증가한 객체 확인
```

---

## GraalVM & Native Image (AOT)

### GraalVM이란?

GraalVM은 Oracle이 개발한 **고성능 JVM 구현체**입니다. 기존 HotSpot JVM을 대체하거나 보완할 수 있으며, 
Java 외에도 JavaScript, Python, Ruby, R 등 다양한 언어를 같은 런타임에서 실행하는 **Polyglot** 기능을 제공합니다.

```
GraalVM 주요 특징:
  ┌─────────────────────────────────────────────────┐
  │                   GraalVM                       │
  │                                                 │
  │  ┌─────────┐ ┌────────┐ ┌────────┐ ┌────────┐   │
  │  │  Java   │ │   JS   │ │ Python │ │  Ruby  │   │
  │  └────┬────┘ └───┬────┘ └───┬────┘ └───┬────┘   │
  │       └──────────┴──────────┴───────────┘       │
  │                      │                          │
  │              Truffle Framework                  │
  │           (언어 인터프리터 공통 프레임워크)             │
  │                      │                          │
  │             GraalVM JIT Compiler                │
  └─────────────────────────────────────────────────┘
```

### Native Image (AOT 컴파일)

GraalVM의 가장 주목받는 기능은 **Native Image**입니다. Java 애플리케이션을 **실행 파일(Native Binary)로 직접 컴파일**합니다.

```
기존 JIT 방식:
  .class → JVM 실행 → JIT 컴파일 → 네이티브 코드
  (실행 시점에 컴파일 → JVM 워밍업 시간 필요)

AOT (Ahead-Of-Time) 방식:
  .class → Native Image 빌드 → 실행 파일 (.exe / 바이너리)
  (빌드 시점에 모두 컴파일 → 즉시 실행)
```

```bash
# Native Image 빌드 (GraalVM 설치 필요)
$ native-image -jar myapp.jar myapp

# 실행 (JVM 없이!)
$ ./myapp
```

### JIT vs AOT 비교

| 구분     | JIT (기존 JVM)     | AOT (Native Image)  |
|--------|------------------|---------------------|
| 시작 시간  | 느림 (워밍업 필요)      | **매우 빠름** (즉시 실행)   |
| 최고 성능  | **높음** (런타임 최적화) | 제한적 (컴파일 시점 정보만 사용) |
| 메모리 사용 | 높음 (JVM 오버헤드)    | **낮음**              |
| 파일 크기  | JVM 별도 필요        | 단독 실행 파일            |
| 적합한 환경 | 장기 실행 서버         | CLI 도구, 서버리스, 컨테이너  |

```
Spring Boot + Native Image 예시:

JIT 방식:
  시작 시간: 2~5초
  메모리: 300~500MB

AOT 방식 (GraalVM Native):
  시작 시간: 0.05~0.1초 (50~100ms!)
  메모리: 50~100MB
```

Kubernetes나 서버리스 환경에서 컨테이너가 자주 생성/삭제될 때 AOT의 빠른 시작 시간이 큰 장점이 됩니다.

### 제약사항

AOT는 모든 상황에서 완벽하지 않습니다.

```
AOT (Native Image) 제약:
  ❌ Reflection 사용 시 별도 설정 필요
  ❌ 동적 클래스 로딩 제한
  ❌ 빌드 시간이 매우 오래 걸림 (수 분)
  ❌ JIT보다 최고 처리 성능은 낮을 수 있음
```

---

## Project Loom — Virtual Thread 심화

EP 04에서 Virtual Thread의 기본 개념을 살펴봤는데, 실제 활용 관점에서 더 살펴보겠습니다.

### Virtual Thread가 해결하는 문제

기존 Java 서버의 전통적인 패턴은 **Thread-Per-Request** 모델이었습니다.

```
Thread-Per-Request (기존):
  요청 1 → Thread 1 → DB 쿼리 대기 (500ms) → 응답
  요청 2 → Thread 2 → API 호출 대기 (300ms) → 응답
  요청 3 → Thread 3 → 파일 읽기 대기 (200ms) → 응답
  ...
  요청 N → Thread N

  문제: Thread 대부분이 I/O 대기 중 → OS Thread 낭비
  결과: 동시 처리 가능 요청 수 = OS Thread 수에 종속
```

Virtual Thread를 사용하면:

```
Virtual Thread (Project Loom):
  요청 1 → VThread 1 → DB 쿼리 대기 시 OS Thread 반환
                        다른 VThread가 OS Thread 사용
                        DB 응답 도착 → VThread 1 재개

  OS Thread 2개로 수천 개 요청 동시 처리 가능!
```

### Spring Boot에서 Virtual Thread 활성화 (Java 21+)

```java
// Spring Boot 3.2+ — application.properties
spring.threads.virtual.enabled=true

// 또는 직접 설정
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerCustomizer() {
    return protocolHandler -> {
        protocolHandler.setExecutor(
            Executors.newVirtualThreadPerTaskExecutor()
        );
    };
}
```

### Virtual Thread 주의사항

```java
// ❌ synchronized와 함께 사용 시 Pinning 문제
synchronized (lock) {
    // I/O 작업 수행 시 Virtual Thread가 OS Thread에 고정(Pin)됨
    // → OS Thread 반환이 안 되어 Virtual Thread의 장점이 사라짐
    Files.readAllBytes(path);
}

// ✅ ReentrantLock 사용 권장
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    Files.readAllBytes(path);  // Pinning 없이 정상 동작
} finally {
    lock.unlock();
}
```

```bash
# Pinning 발생 감지
$ java -Djdk.tracePinnedThreads=full MyApp
```

---

## Project Valhalla & Project Leyden

### Project Valhalla — Value Types

Java의 모든 객체는 Heap에 할당되고 참조(포인터)를 통해 접근합니다. 이는 캐시 미스와 GC 부담의 원인이 됩니다.

```
현재 Java (참조 타입):
  int[] arr = {1, 2, 3};
  → arr은 Heap 객체를 가리키는 포인터
  → 접근할 때마다 포인터 역참조 필요

Valhalla Value Types (예정):
  val Point p = new Point(1, 2);
  → 스택이나 배열 내부에 값 자체가 직접 저장 (C struct처럼)
  → 포인터 역참조 없음, GC 부담 없음, 캐시 친화적
```

```java
// Project Valhalla 예시 (미래 문법, 아직 정식 출시 전)
value class Point {
    int x;
    int y;
}

// 배열에 값이 직접 저장됨 (Heap 객체 참조 아님)
Point[] points = new Point[1000];
```

성능이 중요한 수치 계산, 게임, 금융 애플리케이션에서 큰 성능 향상이 기대됩니다.

### Project Leyden — 시작 시간 최적화

GraalVM의 AOT처럼 빠른 시작 시간을 **표준 JVM에서도** 제공하는 것이 목표입니다.

```
Leyden의 접근 방식:
  기존 JVM의 동적 특성을 유지하면서
  자주 쓰이는 클래스와 JIT 결과를 미리 캐시

  → GraalVM Native처럼 JVM을 버리지 않고도
    빠른 시작 시간과 낮은 메모리 사용을 달성하는 것이 목표
```

```
JVM 시작 시간 비교 (예상):
  현재 JVM (JIT):        2~5초
  GraalVM Native (AOT):  0.05~0.1초
  Project Leyden (목표):  0.3~0.5초 (JVM의 유연성 유지하면서)
```

### JVM의 미래 방향 요약

```
성능 향상:
  Project Valhalla  → Value Types으로 메모리 레이아웃 최적화

시작 시간 단축:
  GraalVM Native    → AOT로 JVM 없이 즉시 실행
  Project Leyden    → 표준 JVM에서 AOT 수준의 시작 시간

동시성 혁신:
  Project Loom      → Virtual Thread로 수백만 동시 처리

이 세 방향이 수렴하는 지점:
  "빠르게 시작하고, 적은 메모리로, 엄청난 동시성을 처리하는 Java"
```

---

## 시리즈를 마치며

5편에 걸쳐 JVM의 내부를 처음부터 끝까지 함께 파헤쳤습니다.

| 편     | 주제                    | 핵심 내용                                            |
|-------|-----------------------|--------------------------------------------------|
| EP 01 | JVM 구조와 ClassLoader   | ByteCode, 부모 위임 모델, 클래스 로딩 3단계                   |
| EP 02 | Runtime Data Area     | Heap, Stack, Metaspace, 스레드 공유 여부                |
| EP 03 | Garbage Collection    | Mark & Sweep, Generational GC, G1 vs ZGC         |
| EP 04 | Execution Engine과 스레드 | JIT, Tiered Compilation, Monitor, Virtual Thread |
| EP 05 | 모니터링, 튜닝, 미래          | JFR, Heap Dump, GraalVM AOT, Valhalla, Leyden    |

JVM은 단순히 Java 코드를 실행해주는 도구가 아닙니다. 수십 년에 걸쳐 축적된 알고리즘과 최적화가 겹겹이 쌓인 정교한 시스템입니다. 
이 시리즈를 통해 JVM을 블랙박스가 아닌 이해할 수 있는 시스템으로 바라볼 수 있게 되었으면 합니다.

### 더 공부하면 좋은 것들

---

> **[JVM Deep Dive 시리즈]**
> - EP 01 | JVM이란 무엇인가?
> - EP 02 | Runtime Data Area — JVM 메모리 구조
> - EP 03 | Garbage Collection — 원리부터 최신 GC까지
> - EP 04 | Execution Engine과 JVM 스레드
> - **EP 05** | JVM 모니터링, 튜닝, 그리고 미래 ← 현재 글