---
title: "EP.01 JVM이란 무엇인가?"
date: "2025-04-06"
episode: 1
tags: ["JVM", "Java"]
---

Java 개발자로 업무하고 있지만 정작 JVM 내부가 어떻게 동작하는지 설명하려면 선뜻 답이 나오지 않습니다.
그래서 JVM을 제대로 파헤쳐보기로 했습니다🙂

이 글은 JVM Deep Dive 시리즈의 첫 번째 편입니다.

## 왜 JVM인가?

1995년, Java가 처음 세상에 나왔을 때 가장 강조했던 슬로건은 **"Write Once, Run Anywhere"** 였습니다.

당시 개발자들은 프로그램을 배포할 때 운영체제마다 다른 버전을 따로 컴파일해야 했습니다.
Windows용, Linux용, macOS용 등과 같은 코드를 여러 번 빌드하고, 환경마다 다르게 동작하는 버그를 따로 잡아야했죠.

Java는 이 문제를 **JVM(Java Virtual Machine)** 이라는 가상 머신을 두는 방식으로 해결했습니다.
Java 코드는 JVM이 이해하는 중간 언어(ByteCode)로 한 번만 컴파일되고, 실제 실행은 각 운영체제 위에 올라간 JVM이 담당합니다.
개발자는 OS를 신경 쓸 필요 없이 JVM만 믿으면 됩니다.

```
Java 소스코드 (.java)
        ↓ javac (java compiler) 컴파일
    ByteCode (.class)
        ↓
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │  JVM on Win  │   │ JVM on Linux │   │ JVM on macOS │
   └──────────────┘   └──────────────┘   └──────────────┘
```

---

## JDK, JRE, JVM — 이 세 가지가 뭐가 다를까?

Java 생태계를 처음 접하면 JDK, JRE, JVM이 헷갈릴 수 있는 데 이 세 가지는 포함 관계로 설명할 수 있습니다.

```
┌─────────────────────────────────────┐
│               JDK                   │
│       (Java Development Kit)        │
│                                     │
│  ┌───────────────────────────────┐  │
│  │             JRE               │  │
│  │  (Java Runtime Environment)   │  │
│  │                               │  │
│  │  ┌─────────────────────────┐  │  │
│  │  │          JVM            │  │  │
│  │  │  (Java Virtual Machine) │  │  │
│  │  └─────────────────────────┘  │  │
│  │       + Java 표준 라이브러리      │  │
│  └───────────────────────────────┘  │
│    + javac, javadoc, jdb, jps ...   │
└─────────────────────────────────────┘
```

| 구분      | 역할                                               | 대상                      |
|---------|--------------------------------------------------|-------------------------|
| **JVM** | ByteCode를 읽어 실제 OS에서 실행하는 가상 머신                  | 실행 엔진 자체                |
| **JRE** | JVM + Java 표준 라이브러리 (java.lang, java.util 등)     | Java 프로그램 실행에 필요한 최소 환경 |
| **JDK** | JRE + 개발 도구 (javac, javadoc, jdb, jps, jstack 등) | Java 개발자를 위한 전체 키트      |

**한 줄 정리 :**

- Java 프로그램을 **실행만** 하려면 → JRE
- Java 프로그램을 **개발**하려면 → JDK (JRE 포함)
- Java의 **핵심**은 → JVM

---

## Java 코드는 어떻게 실행될까? — 컴파일부터 실행까지

Java의 실행 과정은 크게 두 단계로 나뉩니다.

### 1단계: 컴파일 (javac)

```java
// Hello.java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, JVM!");
    }
}
```

```bash
$ javac Hello.java
→ Hello.class 파일 생성 
```

`javac`(Java Compiler)가 `.java` 소스파일을 읽어 `.class` 파일(ByteCode)을 생성합니다. 이 과정에서 문법 오류를 잡아냅니다.

### 2단계: 실행 (JVM)

```bash
$ java Hello
Hello, JVM!
```

`java` 명령어가 JVM을 띄우고, `Hello.class` 파일을 로드하여 실행합니다.

### 전체 흐름 정리

```
Hello.java
    │
    │  javac (컴파일러)
    ▼
Hello.class (ByteCode)
    │
    │  ClassLoader (클래스 로딩)
    ▼
Runtime Data Area (메모리에 적재)
    │
    │  Execution Engine (실행)
    ▼
결과 출력
```

---

## ByteCode란 무엇인가?

ByteCode는 Java 소스코드와 기계어의 **중간 어딘가**에 있는 언어입니다. 특정 CPU의 명령어 셋이 아닌, JVM이라는 가상 머신을 위한 명령어 셋입니다.

직접 ByteCode를 눈으로 확인할 수 있습니다.

```bash
$ javac Hello.java
$ javap -c Hello.class
```

```
Compiled from "Hello.java"
public class Hello {
  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #7  // Field java/lang/System.out
       3: ldc           #13 // String Hello, JVM!
       5: invokevirtual #15 // Method java/io/PrintStream.println
       8: return
}
```

`getstatic`, `ldc`, `invokevirtual` 같은 JVM 명령어(opcode)들이 보입니다. JVM은 이 명령어들을 하나씩 해석하여 실행합니다.

### ByteCode의 장점

| 특성          | 설명                                                |
|-------------|---------------------------------------------------|
| **플랫폼 독립성** | OS나 CPU에 관계없이 JVM만 있으면 실행 가능                      |
| **보안**      | 소스코드 없이 배포 가능, 역컴파일 어려움                           |
| **다언어 지원**  | Kotlin, Scala, Groovy 등도 ByteCode로 컴파일되어 JVM에서 실행 |

---

## JVM 전체 구조 한눈에 보기

JVM은 크게 세 개의 주요 서브시스템으로 구성됩니다.

```
┌──────────────────────────────────────────────────────┐
│                       JVM                            │
│                                                      │
│   ┌──────────────────────────────────────────────┐   │
│   │            Class Loader Subsystem            │   │
│   │     Loading → Linking → Initialization       │   │
│   └──────────────────────────────────────────────┘   │
│                         │                            │
│                         ▼                            │
│   ┌──────────────────────────────────────────────┐   │
│   │             Runtime Data Area                │   │
│   │  ┌───────────┐ ┌──────┐ ┌──────────────────┐ │   │
│   │  │   Method  │ │ Heap │ │     JVM Stack    │ │   │
│   │  │    Area   │ │      │ │    PC Register   │ │   │
│   │  │(Metaspace)│ │      │ │   Native Stack   │ │   │
│   │  └───────────┘ └──────┘ └──────────────────┘ │   │
│   └──────────────────────────────────────────────┘   │
│                         │                            │
│                         ▼                            │
│  ┌──────────────────────────────────────────────┐    │
│  │               Execution Engine               │    │
│  │        Interpreter + JIT Compiler + GC       │    │
│  └──────────────────────────────────────────────┘    │
│                         │                            │
│                         ▼                            │
│            Native Method Interface (JNI)             │
└──────────────────────────────────────────────────────┘
```

각 구성요소는 앞으로 하나씩 깊이 파헤쳐보겠습니다. 이번 편에서는 **Class Loader Subsystem**를 알아보겠습니다.

---

## ClassLoader 시스템 — 클래스는 어떻게 JVM에 올라올까?

JVM이 `Hello.class`를 실행하려면 먼저 그 파일을 메모리에 올려야 합니다. 이 역할을 하는 것이 **ClassLoader**입니다.

ClassLoader는 단 하나가 아니라 **계층 구조**로 이루어진 세 종류의 ClassLoader가 협력합니다.

### ClassLoader 3계층

```
┌────────────────────────────────────────┐
│         Bootstrap ClassLoader          │  ← C/C++ 구현, JVM 내장
│     loads: java.lang, java.util ...    │
└───────────────────┬────────────────────┘
                    │ 부모
┌───────────────────▼────────────────────┐
│          Platform ClassLoader          │  ← Java 구현
│       loads: javax.*, java.sql ...     │
└───────────────────┬────────────────────┘
                    │ 부모
┌───────────────────▼────────────────────┐
│        Application ClassLoader         │  ← Java 구현
│       loads: classpath (user code)     │
└────────────────────────────────────────┘
```

| ClassLoader              | 담당 범위                                   | 구현             |
|--------------------------|-----------------------------------------|----------------|
| **Bootstrap**            | `java.lang`, `java.util` 등 JDK 핵심 라이브러리 | C/C++ (JVM 내장) |
| **Extension (Platform)** | `javax.*`, `java.sql.*` 등 확장 라이브러리      | Java           |
| **Application (System)** | 작성한 코드, 외부 라이브러리 (classpath)            | Java           |

---

## 클래스 로딩의 3단계 : Loading · Linking · Initialization

ClassLoader가 클래스를 JVM에 올리는 과정은 세 단계로 나뉩니다.

### 1단계: Loading (로딩)

`.class` 파일을 찾아서 읽고, JVM 메모리(Method Area)에 올리는 단계입니다.

- ClassLoader가 classpath를 뒤져 `.class` 파일을 찾습니다.
- 파일을 읽어 **바이너리 데이터**로 메모리에 적재합니다.
- 해당 클래스를 표현하는 `java.lang.Class` 객체를 힙에 생성합니다.

```java
// 명시적으로 클래스를 로딩하는 예시
Class<?> clazz = Class.forName("com.example.MyClass");
```

### 2단계: Linking (링킹)

Linking은 다시 세 단계로 나뉩니다.

```
Linking
  ├─ Verification  (검증)
  ├─ Preparation   (준비)
  └─ Resolution    (분석)
```

**Verification (검증)**

- ByteCode가 JVM 스펙에 맞는지 검사합니다.
- 악의적으로 조작된 `.class` 파일 실행을 방지합니다.
- `java.lang.VerifyError`가 여기서 발생합니다.

**Preparation (준비)**

- 클래스의 `static` 필드를 위한 메모리를 할당합니다.
- 이 시점에는 **기본값(default value)** 으로 초기화됩니다. (`int` → `0`, `boolean` → `false`, 객체 → `null`)

**Resolution (분석)**

- 심볼릭 참조(symbolic reference)를 실제 메모리 주소(direct reference)로 변환합니다.
- 예: `"java/lang/System"` 같은 문자열 참조 → 실제 `System` 클래스의 메모리 주소

### 3단계: Initialization (초기화)

드디어 우리가 작성한 코드가 실행되는 첫 단계입니다.

- `static` 블록과 `static` 필드의 **실제 값**으로 초기화됩니다.
- JVM은 이 단계를 **스레드 안전(thread-safe)** 하게 처리합니다.

```java
public class Config {
    // Preparation 단계: count = 0 (기본값)
    // Initialization 단계: count = 10 (실제 값)
    static int count = 10;

    // Initialization 단계에 실행됨
    static {
        System.out.println("Config 클래스 초기화!");
        count = 100;
    }
}
```

```
클래스 최초 사용 시 초기화 트리거 조건 :
  - new 키워드로 인스턴스 생성
  - static 필드 접근 (final 상수 제외)
  - static 메서드 호출
  - Class.forName() 호출
  - 자식 클래스 초기화 시 부모 클래스 먼저 초기화
```

---

## 부모 위임 모델 (Delegation Model)

ClassLoader 계층 구조에는 중요한 원칙이 있습니다.
**클래스 로딩 요청이 들어오면 자신이 직접 로드하기 전에 반드시 부모에게 먼저 위임합니다.**

### 동작 방식

```
Application ClassLoader가 "com.example.Hello" 로딩 요청 받음
         │
         │ 부모에게 위임
         ▼
Extension ClassLoader에게 위임
         │
         │ 부모에게 위임
         ▼
Bootstrap ClassLoader에게 위임
         │
         │ 못 찾음 (핵심 라이브러리가 아님)
         ▼
Extension ClassLoader가 직접 시도
         │
         │ 못 찾음 (확장 라이브러리가 아님)
         ▼
Application ClassLoader가 직접 시도
         │
         │ classpath에서 발견!
         ▼
        로딩 완료
```

```java
// ClassLoader의 loadClass() 의사 코드
protected Class<?> loadClass(String name) throws ClassNotFoundException {
    // 1. 이미 로딩된 클래스인지 캐시 확인
    Class<?> c = findLoadedClass(name);
    if (c != null) return c;

    // 2. 부모에게 먼저 위임
    if (parent != null) {
        try {
            c = parent.loadClass(name); // 재귀적으로 위로 올라감
            if (c != null) return c;
        } catch (ClassNotFoundException ignored) {}
    }

    // 3. 부모가 못 찾으면 자신이 직접 탐색
    return findClass(name);
}
```

### 왜 이런 방식을 쓸까?

**보안과 일관성** 때문입니다.

누군가 악의적으로 `java.lang.String`을 위조한 클래스를 만든다고 상상해보세요. 부모 위임 모델이 없다면 Application ClassLoader가 가짜 `String` 클래스를 로드할 수 있습니다.
하지만 이 모델 덕분에 `java.lang.String` 같은 핵심 클래스는 항상 Bootstrap ClassLoader가 로드한 **진짜 클래스**만 사용됩니다.

```
→ 핵심 클래스 위조 방지 (보안)
→ 동일 클래스가 여러 번 로드되는 것 방지 (일관성)
→ 클래스 간 타입 호환성 보장
```

### ClassLoader와 클래스 동일성

같은 `.class` 파일이라도 **서로 다른 ClassLoader가 로드하면 JVM은 다른 클래스로 인식**합니다.

```java
ClassLoader cl1 = new MyClassLoader();
ClassLoader cl2 = new MyClassLoader();

Class<?> c1 = cl1.loadClass("com.example.Hello");
Class<?> c2 = cl2.loadClass("com.example.Hello");

System.out.println(c1 == c2); // false!
```

이 특성은 WAS(Tomcat, JBoss 등)에서 여러 웹 애플리케이션을 격리하는 데 활용됩니다. 각 애플리케이션마다 별도의 ClassLoader를 두어 서로의 클래스가 충돌하지 않도록 합니다.

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 주제              | 핵심 내용                                              |
|-----------------|----------------------------------------------------|
| JVM의 존재 이유      | Write Once, Run Anywhere — 플랫폼 독립성                 |
| JDK / JRE / JVM | 포함 관계, 각각의 역할                                      |
| 컴파일과 실행         | `.java` → ByteCode → JVM 실행                        |
| ByteCode        | JVM이 이해하는 중간 언어                                    |
| JVM 전체 구조       | ClassLoader + Runtime Data Area + Execution Engine |
| ClassLoader 3계층 | Bootstrap → Extension → Application                |
| 클래스 로딩 3단계      | Loading → Linking → Initialization                 |
| 부모 위임 모델        | 보안과 일관성을 위한 위임 원칙                                  |

다음 편에서는 JVM이 메모리를 어떻게 나누어 관리하는지를 다루려고합니다.
