---
title: "EP.02 equals()와 hashCode() 계약과 설계"
date: "2026-04-15"
episode: 2
tags: ["HashMap", "Java", "equals", "hashCode"]
---

이전 편에서 HashMap이 `put()`할 때 키의 해시값으로 버킷을 찾고, `equals()`로 같은 키인지 비교한다는 걸 봤습니다.
그런데 이 두 메서드를 잘못 구현하면 어떤 일이 일어날까요? `get()`이 `null`을 반환합니다. 값을 분명히 넣었는데도요.

이 글은 HashMap Deep Dive 시리즈의 두 번째 편입니다.

---

## 왜 equals()와 hashCode()인가?

HashMap이 키를 찾는 과정을 다시 떠올려보면:

```
1. hash(key)로 버킷 인덱스 계산    ← hashCode()에 의존
2. 버킷 안에서 key.equals()로 비교 ← equals()에 의존
```

두 메서드가 모두 올바르게 구현되어 있어야 HashMap이 제대로 동작합니다.
하나라도 어긋나면 HashMap은 "없는 키"라고 판단합니다.

이번 편에서는 두 메서드의 **계약(contract)** 을 이해하고, 위반했을 때 어떤 버그가 생기는지 직접 재현해봅니다. 그리고 올바른 구현 방법까지 살펴봅니다.

---

## Object의 기본 구현

Java의 모든 클래스는 `Object`를 상속합니다. `Object`의 기본 구현은 이렇습니다.

```java
// Object.java
public boolean equals(Object obj) {
    return (this == obj);   // 참조 주소 비교
}

public native int hashCode();   // 객체 메모리 주소 기반
```

**참조 비교**와 **메모리 주소 기반 해시**가 기본값입니다.

```java
Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);

System.out.println(p1.equals(p2));  // false — 주소가 다름
System.out.println(p1.hashCode() == p2.hashCode()); // false
```

같은 좌표를 가진 두 Point가 다른 객체로 취급됩니다. HashMap 키로 쓰려면 이 두 메서드를 **직접 오버라이드**해야 합니다.

---

## equals() 계약 — 5가지 규칙

Java 공식 스펙은 `equals()`가 반드시 지켜야 할 다섯 가지 규칙을 정의합니다.

### 1. 반사성 (Reflexivity)

```java
x.equals(x) == true   // 항상 자기 자신과 같아야 함
```

### 2. 대칭성 (Symmetry)

```java
x.equals(y) == true  →  y.equals(x) == true
```

대칭성 위반은 실전에서 가장 많이 발생하는 실수입니다.

```java
// 잘못된 예 — 대칭성 위반
public class CaseInsensitiveString {
    private final String s;

    @Override
    public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(((CaseInsensitiveString) o).s);
        if (o instanceof String)              // String과도 비교
            return s.equalsIgnoreCase((String) o);
        return false;
    }
}

CaseInsensitiveString cis = new CaseInsensitiveString("hello");
String s = "hello";

cis.equals(s);  // true
s.equals(cis);  // false ← 대칭성 위반!
```

### 3. 추이성 (Transitivity)

```java
x.equals(y) == true
y.equals(z) == true
→  x.equals(z) == true
```

### 4. 일관성 (Consistency)

```java
// 객체 상태가 바뀌지 않는 한 결과가 달라지면 안 됨
x.equals(y) == true   // 항상 같은 결과
x.equals(y) == true
x.equals(y) == true
```

### 5. null 비교

```java
x.equals(null) == false   // NullPointerException 없이 false 반환
```

**한 줄 정리:**

```
반사 · 대칭 · 추이 · 일관 · null 안전
이 다섯 가지를 지키지 않으면 HashMap뿐 아니라
Collections, Stream 등 Java 전체에서 이상 동작이 생깁니다.
```

---

## hashCode() 계약 — 핵심 조건 하나

`hashCode()`의 계약에서 가장 중요한 것은 단 하나입니다.

> **equals()가 true인 두 객체는 반드시 같은 hashCode()를 반환해야 한다.**

역방향은 성립하지 않아도 됩니다. 같은 hashCode라고 해서 equals가 true일 필요는 없습니다. (그게 충돌입니다.)

```
equals() == true  →  hashCode() 반드시 같음  (필수)
hashCode() 같음    →  equals() 같을 수도 있음 (보장 안 됨)
```

---

## 계약을 위반하면 — 직접 버그 재현

아래 `Point` 클래스를 보겠습니다. `equals()`만 오버라이드하고 `hashCode()`는 그대로 뒀습니다.

```java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }

    // hashCode() 오버라이드 안 함 — 기본 구현(메모리 주소 기반) 사용
}
```

```java
Map<Point, String> map = new HashMap<>();
Point p1 = new Point(1, 2);
map.put(p1, "서울");

Point p2 = new Point(1, 2);   // p1과 같은 값
System.out.println(p1.equals(p2));  // true  ← 같다고 나오는데
System.out.println(map.get(p2));    // null  ← 왜?
```

### 왜 null이 나오는가?

```
put(p1, "서울")
  → hash(p1) = 0x4f5d6e7  → 7번 버킷에 저장

get(p2)
  → hash(p2) = 0x1a2b3c4  ← p1과 주소가 다르므로 다른 해시값
  → 다른 버킷을 봄
  → 없음 → null 반환
```

`equals()`는 같다고 하는데, `hashCode()`가 달라서 HashMap이 아예 다른 버킷을 뒤집니다. 값이 거기 없으니 `null`을 반환하는 것입니다.

> **규칙:** `equals()`를 오버라이드하면 반드시 `hashCode()`도 함께 오버라이드해야 합니다.
> IDE(IntelliJ, Eclipse)도 이 둘을 항상 쌍으로 생성합니다.

---

## hashCode() 올바르게 구현하기

### 직접 구현 — String.hashCode()에서 배우기

Java의 `String.hashCode()`는 이렇게 구현되어 있습니다.

```java
// String.java
public int hashCode() {
    int h = 0;
    for (char c : value) {
        h = 31 * h + c;
    }
    return h;
}
```

**왜 31을 곱하는가?**

```
31 = 2^5 - 1

JVM은 31 * h를 (h << 5) - h로 최적화합니다.
곱셈 대신 비트 시프트와 뺄셈으로 처리해 빠릅니다.

또한 31은 홀수 소수입니다.
짝수를 곱하면 비트가 왼쪽으로 밀려 정보가 손실되고,
소수는 해시 충돌을 줄이는 수학적 성질을 가집니다.
```

이 방식을 `Point`에 적용하면:

```java
@Override
public int hashCode() {
    int result = 17;        // 임의의 소수로 시작
    result = 31 * result + x;
    result = 31 * result + y;
    return result;
}
```

### Objects.hash() — 편리하지만 트레이드오프 있음

```java
@Override
public int hashCode() {
    return Objects.hash(x, y);   // 내부적으로 Arrays.hashCode() 사용
}
```

코드가 간결하지만 **가변 인수(varargs)** 처리로 인해 배열이 매번 생성됩니다.
성능이 중요한 클래스라면 직접 구현하는 게 낫습니다.

```
Objects.hash()  →  간결, 약간의 성능 오버헤드
직접 구현         →  코드 많음, 성능 유리
```

---

## 완성된 Point 클래스

```java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;            // 1. 같은 참조면 바로 true
        if (!(o instanceof Point)) return false; // 2. 타입 확인
        Point p = (Point) o;
        return x == p.x && y == p.y;           // 3. 필드 비교
    }

    @Override
    public int hashCode() {
        int result = 17;
        result = 31 * result + x;
        result = 31 * result + y;
        return result;
    }
}
```

```java
Map<Point, String> map = new HashMap<>();
Point p1 = new Point(1, 2);
map.put(p1, "서울");

Point p2 = new Point(1, 2);
System.out.println(p1.equals(p2));   // true
System.out.println(map.get(p2));     // "서울" ← 이제 제대로 동작
```

---

## Lombok과 record — 자동 생성의 함정

### Lombok @EqualsAndHashCode

```java
@EqualsAndHashCode
public class Point {
    private final int x;
    private final int y;
}
```

모든 필드를 기준으로 `equals()`와 `hashCode()`를 자동 생성합니다.
상속 관계가 있다면 `@EqualsAndHashCode(callSuper = true)`를 빠뜨리지 않도록 주의해야 합니다.

### Java 16+ record

```java
public record Point(int x, y) {}
```

`record`는 선언한 컴포넌트를 기준으로 `equals()`, `hashCode()`, `toString()`을 **컴파일러가 자동 생성**합니다. 별도 코드 없이 HashMap 키로 안전하게 사용할 수 있습니다.

```java
Point p1 = new Point(1, 2);
Point p2 = new Point(1, 2);

System.out.println(p1.equals(p2));   // true
System.out.println(p1.hashCode() == p2.hashCode());  // true
```

```
불변 값 객체를 HashMap 키로 쓴다면
record가 가장 간결하고 안전한 선택입니다.
```

---

## 좋은 hashCode의 조건

구현 방식과 상관없이 좋은 `hashCode()`가 갖춰야 할 성질은 명확합니다.

| 조건 | 내용 |
|------|------|
| **일관성** | 같은 객체는 항상 같은 값 반환 |
| **equals 연동** | equals가 true면 hashCode도 반드시 같음 |
| **분산성** | 서로 다른 객체는 되도록 다른 값 반환 |
| **빠른 계산** | HashMap은 키 접근마다 hashCode를 호출함 |

분산성이 나쁜 `hashCode()`는 모든 키를 같은 버킷에 몰아넣어 `O(1)` 성능을 `O(n)` 또는 `O(log n)`으로 떨어뜨립니다.

```java
// 나쁜 예 — 항상 같은 값 반환
@Override
public int hashCode() {
    return 42;   // 합법적이지만 모든 키가 한 버킷에 몰림
}
```

이 구현은 계약을 위반하지 않습니다. 하지만 HashMap을 사실상 연결리스트로 만들어버립니다.

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 주제 | 핵심 내용 |
|------|---------|
| Object 기본 구현 | 참조 비교 & 메모리 주소 기반 해시 — HashMap 키로 쓰려면 오버라이드 필수 |
| equals() 5가지 규칙 | 반사 · 대칭 · 추이 · 일관 · null 안전 |
| hashCode() 핵심 계약 | equals가 true면 hashCode도 반드시 같아야 함 |
| 계약 위반 버그 | hashCode 없이 equals만 구현 → get()이 null 반환 |
| 31 곱셈의 이유 | 홀수 소수 + 비트 시프트 최적화 |
| Objects.hash() | 간결하지만 varargs 배열 생성 오버헤드 있음 |
| record | 컴파일러가 자동 생성 — 불변 키 객체에 가장 적합 |

다음 편에서는 실전에서 HashMap을 어떻게 튜닝하고, 멀티스레드 환경에서 왜 위험한지, 그리고 상황별로 어떤 Map을 골라야 하는지를 다루려고 합니다.