---
title: "EP.02 Runtime Data Area — JVM 메모리 구조"
date: "2025-04-06"
episode: 2
tags: [ "JVM", "Java", "Memory", "Heap", "Stack" ]
---

지난 편에서 JVM의 전체 구조와 ClassLoader가 클래스를 메모리에 올리는 과정을 살펴봤습니다.
그렇다면 올라간 클래스는 메모리 어디에, 어떤 형태로 저장될까요?

이번 편에서는 JVM이 메모리를 어떻게 나누어 관리하는지 살펴보겠습니다.🙂

이 글은 JVM Deep Dive 시리즈의 두 번째 편입니다.

## Runtime Data Area란?

JVM이 프로그램을 실행하기 위해 OS로부터 할당받은 **메모리 공간** 전체를 Runtime Data Area라고 합니다.

JVM 전체 구조에서 보면 ClassLoader가 클래스를 로딩한 뒤 데이터가 저장되는 곳입니다.

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
│   │          ★ Runtime Data Area ★               │   │
│   │   Method Area · Heap · Stack · PC · Native   │   │
│   └──────────────────────────────────────────────┘   │
│                         │                            │
│                         ▼                            │
│   ┌──────────────────────────────────────────────┐   │
│   │             Execution Engine                 │   │
│   └──────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

Runtime Data Area는 아래와 같이 총 5개의 영역으로 나뉩니다.

```
┌─────────────────────────────────────────────────┐
│               Runtime Data Area                 │
│                                                 │
│  ┌──────────────────┐  ┌─────────────────────┐  │
│  │   Method Area    │  │        Heap         │  │
│  │   (Metaspace)    │  │  (Young + Old Gen)  │  │
│  │    [스레드 공유]    │  │    [스레드 공유]       │  │
│  └──────────────────┘  └─────────────────────┘  │
│                                                 │
│  ┌────────────┐  ┌──────────┐  ┌─────────────┐  │
│  │ JVM Stack  │  │    PC    │  │   Native    │  │
│  │            │  │ Register │  │   Method    │  │
│  │ [스레드 전용] │  │[스레드전용]│   │   Stack     │  │
│  └────────────┘  └──────────┘  └─────────────┘  │
└─────────────────────────────────────────────────┘
```

각 영역을 하나씩 살펴보겠습니다.

---

## Method Area (Metaspace)

**클래스 자체에 대한 정보**가 저장되는 공간입니다. ClassLoader가 클래스를 로딩하면 그 메타데이터가 여기에 쌓입니다.

### 저장되는 데이터

```
Method Area (Metaspace)
  ├─ 클래스 이름, 부모 클래스 이름
  ├─ 클래스의 타입 정보 (class / interface / enum)
  ├─ static 변수
  ├─ 메서드 정보 (이름, 리턴 타입, 파라미터)
  └─ 메서드 ByteCode
```

```java
public class Counter {
    static int count = 0;       // → Method Area에 저장
    String name;                // → 인스턴스 생성 시 Heap에 저장

    public void increment() {   // → 메서드 정보 Method Area에 저장
        count++;
    }
}
```

---

## Heap

Heap은 JVM에서 가장 중요한 메모리 영역입니다. **`new` 키워드로 생성된 모든 객체와 배열**이 저장됩니다.

```java
String name = new String("JVM");   // → Heap에 저장
int[] arr = new int[10];           // → Heap에 저장
Counter counter = new Counter();   // → Heap에 저장
```

### Heap의 내부 구조

Heap은 GC(Garbage Collector)가 효율적으로 동작하도록 내부적으로 영역이 나뉩니다.

```
┌──────────────────────────────────────────────────────┐
│                        Heap                          │
│                                                      │
│  ┌────────────────────────────┐  ┌─────────────────┐ │
│  │     Young Generation       │  │  Old Generation │ │
│  │                            │  │                 │ │
│  │  ┌────────┐  ┌──────────┐  │  │  오래 살아남은     │ │
│  │  │  Eden  │  │ Survivor │  │  │  객체들이         │ │
│  │  │        │  │  S0 / S1 │  │  │  이동해오는 곳     │ │
│  │  │ 새 객체  │  │          │  │  │                │ │
│  │  │ 생성 공간│  │          │  │  │                 │ │
│  │  └────────┘  └──────────┘  │  │                 │ │
│  └────────────────────────────┘  └─────────────────┘ │
└──────────────────────────────────────────────────────┘
```

| 영역                     | 설명                                       |
|------------------------|------------------------------------------|
| **Eden**               | 새로 생성된 객체가 처음 할당되는 곳                     |
| **Survivor (S0 / S1)** | Eden에서 살아남은 객체가 이동하는 곳                   |
| **Old Generation**     | Survivor에서 오래 살아남은 객체가 승격(Promotion)되는 곳 |

> 💡 이 구조는 GC와 밀접하게 연결됩니다. Heap의 Young Generation에서 발생하는 GC를 **Minor GC**, 
> Old Generation을 포함한 전체 GC를 **Major GC (Full GC)** 라고 합니다. 자세한 내용은 다음 편 GC에서 다룹니다.

---

## JVM Stack

**메서드 호출과 지역 변수**를 관리하는 공간입니다. 스레드마다 하나씩 생성됩니다.

### Stack Frame

메서드가 호출될 때마다 Stack에 **Stack Frame**이 하나씩 쌓이고, 메서드가 종료되면 Frame이 제거됩니다.

```java
public class Example {
    public static void main(String[] args) {
        int result = add(3, 5);         // add() 호출
        System.out.println(result);
    }

    public static int add(int a, int b) {
        return a + b;                   // add() 종료
    }
}
```

```
main() 호출 시:
┌──────────────────┐
│   add() Frame    │  ← 가장 최근 호출 (top)
│    a=3, b=5      │
├──────────────────┤
│   main() Frame   │
│  result, args    │
└──────────────────┘

add() 종료 후:
┌──────────────────┐
│   main() Frame   │  ← top
│  result=8, args  │
└──────────────────┘
```

### Stack Frame의 구성

각 Stack Frame은 세 가지로 구성됩니다.

```
Stack Frame
  ├─ Local Variable Array  : 지역 변수, 파라미터 저장
  ├─ Operand Stack         : 연산 중간 결과 저장 (계산기 역할)
  └─ Frame Data            : 상수 풀 참조, 이전 Frame 정보 등
```

```java
public static int add(int a, int b) {
    int result = a + b;
    // Local Variable Array: a=3, b=5, result
    // Operand Stack: a와 b를 꺼내 더한 뒤 result에 저장
    return result;
}
```

---

## PC Register

**Program Counter Register**의 줄임말입니다. 현재 스레드가 **실행 중인 JVM 명령어(opcode)의 주소**를 저장합니다.

```
스레드 A                 스레드 B
┌─────────────┐         ┌─────────────┐
│ PC Register │         │ PC Register │
│  addr: 0x04 │         │  addr: 0x11 │
└─────────────┘         └─────────────┘
        │                       │
        ▼                       ▼
  invokevirtual             getstatic
  (현재 실행 중인 명령어)    (현재 실행 중인 명령어)
```

CPU가 여러 스레드를 번갈아 실행할 때, 각 스레드는 자신의 PC Register를 보고 어디서부터 다시 실행할지 기억합니다. 스레드마다 독립적으로 존재하기 때문에 다른 스레드의 실행 흐름에 영향을 받지 않습니다.

---

## Native Method Stack

Java가 아닌 **C/C++ 등 네이티브 언어로 작성된 메서드**가 실행될 때 사용되는 스택입니다.

```java
// Java에서 native 메서드 선언
public class System {
    public static native void arraycopy(...);  // JNI를 통해 C로 구현됨
}
```

`native` 키워드가 붙은 메서드는 JVM Stack이 아닌 **Native Method Stack**에서 실행됩니다. JNI(Java Native Interface)를 통해 OS 수준의 기능을 직접 호출할 때
사용됩니다.

---

## 스레드 공유 vs 스레드 전용

5개 영역의 가장 중요한 구분 기준은 **스레드 간 공유 여부**입니다.

```
┌─────────────────────────────────────────────────────┐
│                  [스레드 공유 영역]                     │
│                                                     │
│  ┌──────────────────┐    ┌────────────────────────┐ │
│  │   Method Area    │    │          Heap          │ │
│  │   (Metaspace)    │    │   (Young + Old Gen)    │ │
│  └──────────────────┘    └────────────────────────┘ │
└─────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                    [스레드 전용 영역]                        │
│                                                          │
│  Thread 1              Thread 2              Thread 3    │
│  ┌────────────┐        ┌────────────┐        ┌────────┐  │
│  │ JVM Stack  │        │ JVM Stack  │        │  ...   │  │
│  │ PC Register│        │ PC Register│        │        │  │
│  │ Native     │        │ Native     │        │        │  │
│  │ Stack      │        │ Stack      │        │        │  │
│  └────────────┘        └────────────┘        └────────┘  │
└──────────────────────────────────────────────────────────┘
```

| 구분         | 영역                                   | 특징                              |
|------------|--------------------------------------|---------------------------------|
| **스레드 공유** | Method Area, Heap                    | 모든 스레드가 같은 데이터에 접근 → **동기화 필요** |
| **스레드 전용** | JVM Stack, PC Register, Native Stack | 스레드마다 독립적으로 존재 → **동기화 불필요**    |

스레드 공유 영역은 여러 스레드가 동시에 접근하기 때문에 멀티스레드 환경에서 **동기화 문제**가 발생할 수 있습니다. `synchronized`, `volatile` 같은 키워드가 필요한 이유가 여기에 있습니다.

---

## 실제 에러 사례 — OOM과 StackOverflow

Runtime Data Area를 이해하면 자주 마주치는 JVM 에러의 원인을 정확히 파악할 수 있습니다.

### OutOfMemoryError (OOM)

**Heap 또는 Metaspace가 가득 찼을 때** 발생합니다.

```java
// Heap OOM 유발 예시
List<byte[]> list = new ArrayList<>();
while (true) {
    list.add(new byte[1024 * 1024]); // 1MB씩 계속 추가
}
```

```
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

```java
// Metaspace OOM 유발 예시 (동적으로 클래스를 무한 생성)
while (true) {
    new ByteBuddy().subclass(Object.class).make()
        .load(ClassLoader.getSystemClassLoader());
}
```

```
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
```

에러 메시지에 어느 영역이 문제인지 명시되기 때문에, 메시지를 잘 읽으면 원인을 빠르게 파악할 수 있습니다.

```
java.lang.OutOfMemoryError: Java heap space   → Heap 부족
java.lang.OutOfMemoryError: Metaspace         → Metaspace 부족
java.lang.OutOfMemoryError: GC overhead limit → GC가 너무 자주 발생
```

### StackOverflowError

**JVM Stack이 가득 찼을 때** 발생합니다. 주로 재귀 호출이 종료 조건 없이 계속될 때 나타납니다.

```java
// StackOverflow 유발 예시
public static void recurse() {
    recurse(); // 종료 조건 없는 재귀 호출
}
```

```
Exception in thread "main" java.lang.StackOverflowError
```

```
recurse() Frame
recurse() Frame
recurse() Frame   ← Stack이 꽉 찰 때까지 Frame이 쌓임
recurse() Frame
...
→ StackOverflowError 발생!
```

JVM Stack의 크기는 `-Xss` 옵션으로 조절할 수 있습니다.

```bash
$ java -Xss512k MyApp   # Stack 크기를 512KB로 설정
```

### 메모리 옵션 정리

```bash
-Xms512m               # Heap 초기 크기 (512MB)
-Xmx2g                 # Heap 최대 크기 (2GB)
-Xss512k               # 스레드별 Stack 크기 (512KB)
-XX:MetaspaceSize=128m # Metaspace 초기 크기
-XX:MaxMetaspaceSize=256m  # Metaspace 최대 크기
```

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 영역                          | 저장 데이터                             | 스레드 공유 |
|-----------------------------|------------------------------------|--------|
| **Method Area (Metaspace)** | 클래스 메타데이터, static 변수, 메서드 ByteCode | ✅ 공유   |
| **Heap**                    | new로 생성된 객체와 배열                    | ✅ 공유   |
| **JVM Stack**               | Stack Frame (지역변수, 연산 결과)          | ❌ 전용   |
| **PC Register**             | 현재 실행 중인 명령어 주소                    | ❌ 전용   |
| **Native Method Stack**     | 네이티브 메서드 실행 정보                     | ❌ 전용   |

Heap은 다음 편에서 다룰 GC와 직결됩니다. Heap 구조를 이해했다면 GC가 왜 Young Generation을 먼저 청소하는지, 왜 Old Generation에서 Full GC가 발생하면 성능에 영향을 주는지
자연스럽게 이해할 수 있을 것입니다.

다음 편에서는 **GC가 Heap을 어떻게 청소하는지**, 그리고 G1 GC와 ZGC 같은 최신 GC는 기존 방식과 무엇이 다른지 살펴보겠습니다.

