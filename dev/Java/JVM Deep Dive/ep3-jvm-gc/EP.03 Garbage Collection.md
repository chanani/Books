---
title: "EP.03 Garbage Collection — 원리부터 최신 GC까지"
date: "2026-04-07"
episode: 3
tags: [ "JVM", "Java", "GC", "Garbage Collection" ]
---

지난 편에서 JVM이 메모리를 어떻게 나누어 관리하는지, 특히 Heap의 Young Generation과 Old Generation 구조를 살펴봤습니다.
그렇다면 Heap에 쌓인 객체들은 누가, 언제, 어떻게 정리할까요?

이번 편에서는 JVM의 핵심 기능 중 하나인 **Garbage Collection(GC)** 을 깊이 파헤쳐보겠습니다.🙂

이 글은 JVM Deep Dive 시리즈의 세 번째 편입니다.

## GC란 무엇인가?

C/C++ 개발자는 메모리를 직접 할당하고 직접 해제해야 합니다.

```c
// C 언어
int* arr = (int*)malloc(sizeof(int) * 10);  // 직접 할당
free(arr);                                   // 직접 해제 (안 하면 메모리 누수)
```

Java는 다릅니다. 개발자가 `new`로 객체를 생성하기만 하면,
**더 이상 사용하지 않는 객체를 JVM이 자동으로 찾아서 메모리를 회수**해줍니다. 이 자동 메모리 관리 기능이 바로 **GC(Garbage Collection)** 입니다.

```java
// Java — 해제는 GC가 알아서 해준다
String name = new String("JVM");  // Heap에 할당
name = null;                       // 참조를 끊으면 GC 대상이 됨
// free() 같은 코드 없음
```

GC 덕분에 Java 개발자는 메모리 관리에 신경 쓰지 않아도 되지만, GC가 **언제, 어떻게 동작하는지** 모르면 성능 문제가 생겼을 때 원인을 찾기 어렵습니다.

---

## GC의 대상은 누가인가?

GC는 **더 이상 참조되지 않는 객체**를 쓰레기(Garbage)로 간주하고 회수합니다.

### Reachability (도달 가능성)

JVM은 **GC Root**에서 시작해서 참조를 따라 도달할 수 있는 객체는 살려두고, 도달할 수 없는 객체는 수거합니다.

```
GC Root (출발점)
  ├─ 현재 실행 중인 메서드의 지역 변수
  ├─ static 변수
  ├─ JNI(Java Native Inteface) 참조
  └─ 활성 스레드

GC Root → A → B → C    (A, B, C 모두 Reachable → 살아남음)
GC Root → D            (D는 Reachable → 살아남음)

X → Y                  (GC Root와 연결 없음 → Unreachable → 수거 대상)
```

```java
public void example() {
    Object a = new Object();   // GC Root(지역변수)에서 참조 → Reachable
    Object b = new Object();   // GC Root(지역변수)에서 참조 → Reachable
    b = null;                  // 참조 끊김 → Unreachable → GC 대상
}
// 메서드 종료 → a도 스택 프레임이 사라지며 GC 대상
```

---

## GC 기본 알고리즘 — Mark & Sweep & Compaction

GC의 핵심 동작은 세 단계로 이루어집니다.

### 1단계: Mark (표시)

GC Root에서 시작해 참조를 따라가며 살아있는 객체에 표시합니다.

```
Heap 상태:
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│ A  │ │ B  │ │ X  │ │ C  │ │ Y  │ │ D  │
│(참조│ │(참조│ │(미  │ │(참조│ │(미  │ │(참조│
│ 있음│ │ 있음│ │참조)│ │ 있음│ │참조)│ │ 있음│
└────┘ └────┘ └────┘ └────┘ └────┘ └────┘

Mark 후 :
  A ✅   B ✅   X ❌   C ✅   Y ❌   D ✅
```

### 2단계: Sweep (제거)

표시되지 않은 객체(Unreachable)를 메모리에서 제거합니다.

```
Sweep 후:
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│ A  │ │ B  │ │    │ │ C  │ │    │ │ D  │
│    │ │    │ │(emp│ │    │ │(emp│ │    │
│    │ │    │ │ ty)│ │    │ │ ty)│ │    │
└────┘ └────┘ └────┘ └────┘ └────┘ └────┘
```

### 3단계: Compaction (압축)

Sweep 후 메모리가 파편화(Fragmentation)되는 문제를 해결하기 위해 살아있는 객체를 한쪽으로 모읍니다.

```
Compaction 후:
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌──────────────┐
│ A  │ │ B  │ │ C  │ │ D  │ │    empty     │
│    │ │    │ │    │ │    │ │  (연속된 공간)  │
└────┘ └────┘ └────┘ └────┘ └──────────────┘
```

Compaction이 없으면 빈 공간이 여기저기 흩어져 있어 큰 객체를 할당할 때 공간이 있어도 할당에 실패하는 **메모리 단편화** 문제가 발생합니다.

---

## Generational GC — 왜 Heap을 나눴을까?

Mark & Sweep만으로도 GC는 동작하지만, 전체 Heap을 매번 스캔하는 건 너무 비효율적입니다.
이를 해결하기 위해 등장한 것이 **Generational GC** 입니다.

### 약한 세대 가설 (Weak Generational Hypothesis)

Generational GC는 다음 두 가지 경험적 사실에서 출발합니다.

```
1. 대부분의 객체는 생성 후 짧은 시간 안에 죽는다.
2. 오래된 객체가 새로운 객체를 참조하는 경우는 드물다.
```

실제로 Java 애플리케이션을 분석해보면 대부분의 객체는 임시적입니다.

```java
// 이런 객체들은 메서드가 끝나면 바로 필요 없어진다
for (int i = 0; i < 10000; i++) {
    String temp = "item_" + i;   // 루프마다 생성, 루프 끝나면 불필요
    process(temp);
}
```

### 세대별 영역 구분

이 가설을 바탕으로 Heap을 **Young Generation**과 **Old Generation**으로 나눠서 관리합니다.

```
┌───────────────────────────────────────────────────────────┐
│                          Heap                             │
│                                                           │
│   ┌──────────────────────────────────┐  ┌─────────────┐   │
│   │        Young Generation          │  │     Old     │   │
│   │                                  │  │  Generation │   │
│   │  ┌──────────┐  ┌──────┐ ┌──────┐ │  │             │   │
│   │  │  Eden    │  │  S0  │ │  S1  │ │  │  오래 살아남   │   │
│   │  │          │→→│      │↔│      │ │→→│  은 객체들    │   │
│   │  │ 새 객체    │  │      │ │      │ │  │             │   │
│   │  └──────────┘  └──────┘ └──────┘ │  │             │   │
│   └──────────────────────────────────┘  └─────────────┘   │
│         Minor GC가 자주 발생                 Major GC (드묾)  │
└───────────────────────────────────────────────────────────┘
```

---

## Minor GC vs Major GC (Full GC)

### Minor GC — Young Generation 청소

Young Generation이 가득 차면 Minor GC가 발생합니다.

```
1. 새 객체 → Eden에 할당
        ↓
2. Eden이 가득 참 → Minor GC 발생
        ↓
3. Mark: 살아있는 객체 표시
        ↓
4. 살아남은 객체 → Survivor(S0 또는 S1) 로 이동
   죽은 객체 → 제거
        ↓
5. 반복 생존 시 → age 카운터 증가
        ↓
6. age가 임계값(기본 15) 초과 → Old Generation으로 승격 (Promotion)
```

```java
// 객체마다 age 카운터를 갖는다 (JVM 내부)
// Minor GC에서 살아남을 때마다 +1
// age >= MaxTenuringThreshold(기본 15) → Old Generation 이동
```

Minor GC는 Young Generation만 대상으로 하기 때문에 **빠르게 끝납니다** (보통 수 밀리초).

### Major GC (Full GC) — Old Generation 청소

Old Generation까지 가득 차면 Major GC(Full GC)가 발생합니다.

```
Minor GC 반복
    ↓
Old Generation 점점 채워짐
    ↓
Old Generation 가득 참
    ↓
Major GC (Full GC) 발생
    ↓
전체 Heap 대상으로 Mark & Sweep & Compaction
    ↓
Stop-The-World 발생 → 애플리케이션 일시 정지!
```

Major GC는 전체 Heap을 대상으로 하기 때문에 **시간이 오래 걸립니다**
(수백 밀리초 ~ 수 초). 이 시간 동안 애플리케이션이 멈추는 것이 **Stop-The-World** 입니다.

---

## GC의 종류 — Serial부터 ZGC까지

JVM은 다양한 GC 알고리즘을 제공합니다. 각각 설계 목표가 다르므로 상황에 맞게 선택해야 합니다.

### Serial GC

```
특징 : 단일 스레드로 GC 수행
대상 : 싱글 코어 환경, 소규모 애플리케이션
단점 : GC 동안 다른 모든 작업 중단
옵션 : -XX:+UseSerialGC
```

```
GC 발생 시 :
App Thread ────────────┤  GC (단일 스레드)     ├────────────
                       └────────────────────┘
                              STW 구간
```

### Parallel GC (Java 8 기본)

```
특징 : 여러 스레드로 GC 수행 → Throughput 향상
대상 : 배치 처리, 대용량 데이터 처리
단점 : STW 시간은 여전히 존재
옵션 : -XX:+UseParallelGC
```

```
GC 발생 시 :
App Thread ────────────┤  GC (멀티 스레드)     ├────────────
                       └────────────────────┘
                         STW (Serial보다 짧음)
```

### G1 GC (Java 9+ 기본)

```
특징 : Heap을 Region 단위로 나눠 관리
     STW 시간을 예측 가능하게 제어
대상 : 대용량 Heap (4GB 이상), 응답 시간 중요한 서비스
옵션 : -XX:+UseG1GC
     -XX:MaxGCPauseMillis=200 (목표 STW 시간 설정)
```

```
G1 GC의 Heap 구조 :
┌────┬────┬────┬────┬────┬────┐
│ E  │ S  │ O  │ E  │ O  │ H  │  E: Eden
├────┼────┼────┼────┼────┼────┤  S: Survivor
│ O  │ E  │ S  │ O  │ E  │ O  │  O: Old
├────┼────┼────┼────┼────┼────┤  H: Humongous (대형 객체)
│ H  │ O  │ E  │ S  │ O  │ E  │
└────┴────┴────┴────┴────┴────┘
  Region 단위로 나눠서 필요한 곳만 GC
```

G1은 Young/Old 경계가 고정되지 않고 Region 단위로 유연하게 관리합니다.
가장 쓰레기가 많은(Garbage First!) Region을 우선 수집합니다.

### ZGC (Java 15+ 안정화)

```
특징 : STW 시간을 1~10ms 이하로 줄인 저지연 GC
      Heap 크기에 관계없이 일정한 STW 보장
대상 : 수십 GB ~ 수 TB Heap, 초저지연이 필요한 서비스
옵션 : -XX:+UseZGC
```

```
ZGC의 STW 비교 :
Serial/Parallel GC :  ████████████████ (수백ms ~ 수초)
G1 GC :               ████ (수십~수백ms)
ZGC :                 ▌ (1~10ms)
```

ZGC는 GC 작업 대부분을 애플리케이션 스레드와 **동시에(Concurrent)** 수행해서 STW를 최소화합니다.

### GC 선택 가이드

| 상황                    | 권장 GC               |
|-----------------------|---------------------|
| 소규모 앱, 단순 배치          | Serial GC           |
| 대용량 배치, Throughput 우선 | Parallel GC         |
| 일반 서버 앱, Heap 4GB+    | **G1 GC** (대부분의 상황) |
| 초저지연 필요, 대용량 Heap     | **ZGC**             |

---

## Stop-The-World란?

GC가 실행되는 동안 **모든 애플리케이션 스레드가 멈추는 현상**입니다.

```
정상 동작:
App Thread 1  ──────────────────────────────────────
App Thread 2  ──────────────────────────────────────
App Thread 3  ──────────────────────────────────────

GC 발생 (STW):
App Thread 1  ─────────┤          ├─────────────────
App Thread 2  ─────────┤   STW!!  ├─────────────────
App Thread 3  ─────────┤          ├─────────────────
GC Thread     ─────────┤██████████├─────────────────
                       └──────────┘
                이 구간 동안 사용자 요청 처리 안 됨
```

STW가 길면 사용자 입장에서 **애플리케이션이 순간적으로 멈추는 것처럼** 느껴집니다. 실시간 서비스에서 GC 튜닝이 중요한 이유입니다.

### STW를 줄이는 방법

```
1. 적절한 GC 선택 (G1, ZGC)
2. Heap 크기 조정 (-Xms, -Xmx)
3. 객체 생성 줄이기 (객체 재사용, 캐시 활용)
4. GC 로그 분석 후 튜닝
```

---

## GC 로그 읽는 법

GC 로그를 활성화하면 언제, 얼마나 오래 GC가 발생했는지 확인할 수 있습니다.

### GC 로그 활성화

```bash
# Java 9 이상
$ java -Xlog:gc*:file=gc.log:time,uptime,level,tags MyApp

# Java 8
$ java -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log MyApp
```

### G1 GC 로그 예시

```
[2025-04-06T10:00:01.234+0900][0.512s][info][gc] GC(0) Pause Young (Normal) (G1 Evacuation Pause) 128M->32M(512M) 12.345ms
[2025-04-06T10:00:05.678+0900][4.956s][info][gc] GC(1) Pause Young (Normal) (G1 Evacuation Pause) 256M->64M(512M) 8.123ms
[2025-04-06T10:00:30.123+0900][29.401s][info][gc] GC(2) Pause Full (System.gc()) 400M->100M(512M) 450.678ms
```

### 로그 읽는 법

```
[시각][경과시간][레벨][태그] GC 번호 GC 종류 Heap 변화 STW 시간

예시:
GC(0)                    → 첫 번째 GC
Pause Young (Normal)     → Minor GC (Young Generation 대상)
128M->32M(512M)          → GC 전 128MB → GC 후 32MB (전체 Heap 512MB)
12.345ms                 → STW 시간 12.345밀리초

GC(2)
Pause Full               → Full GC (전체 Heap 대상)
450.678ms                → STW 시간 450밀리초 (훨씬 길다!)
```

STW 시간이 지속적으로 길거나, Full GC가 자주 발생한다면 GC 튜닝이 필요한 신호입니다.

---

## 핵심 GC 옵션 정리

```bash
# GC 선택
-XX:+UseSerialGC          # Serial GC
-XX:+UseParallelGC        # Parallel GC
-XX:+UseG1GC              # G1 GC
-XX:+UseZGC               # ZGC

# Heap 크기
-Xms512m                  # Heap 초기 크기
-Xmx4g                    # Heap 최대 크기

# G1 GC 튜닝
-XX:MaxGCPauseMillis=200  # 목표 STW 시간 (ms)
-XX:G1HeapRegionSize=16m  # Region 크기 (1~32MB, 2의 제곱수)

# GC 로그
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# GC 통계 출력 (Java 8)
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
```

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 주제              | 핵심 내용                                      |
|-----------------|--------------------------------------------|
| GC의 역할          | 더 이상 참조되지 않는 객체를 자동으로 회수                   |
| GC 대상 판별        | GC Root에서 도달 불가능한 객체 (Unreachable)         |
| 기본 알고리즘         | Mark → Sweep → Compaction                  |
| Generational GC | 짧게 사는 객체가 많다는 가설 → Young/Old 분리            |
| Minor GC        | Young Generation 대상, 빠름 (수 ms)             |
| Major GC        | 전체 Heap 대상, 느림 (수백 ms ~ 수 초)               |
| Stop-The-World  | GC 중 모든 앱 스레드 일시 정지                        |
| GC 종류           | Serial → Parallel → G1 → ZGC (저지연 방향으로 발전) |

다음 편에서는 ByteCode가 실제로 어떻게 실행되는지,
**Interpreter와 JIT 컴파일러**가 어떻게 협력하는지, 그리고 JVM이 멀티스레드를 어떻게 관리하는지 살펴보겠습니다.
