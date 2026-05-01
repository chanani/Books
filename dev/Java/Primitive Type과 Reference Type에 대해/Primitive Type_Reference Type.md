---
title: "Primitive Type과 Reference Type에 대해"
date: "2026-05-01"
tags: [ "Primitive Type", "Reference Type", "Java", "Stack", "Heap" ]
description: "Java의 Primitive Type과 Reference Type의 차이, 메모리 구조, 그리고 실무에서 마주치는 함정들"
---

Primitive Type과 Reference Type 두 타입의 차이를 제대로 이해하지 못하면
NullPointerException, 의도치 않은 값 변경, 컬렉션에서의 이상한 동작까지 사소한 버그에 시간을 한참 쏟게 됩니다.

## ⭐️ Primitive Type과 Reference Type이란?

### Primitive Type (기본 타입)

Primitive Type은 Java에서 미리 정의된 **8가지 기본 타입**을 의미합니다.
값 그 자체를 변수에 저장하며, JVM의 **Stack 영역**에 직접 할당됩니다.

| 분류  | 타입                             |
|-----|--------------------------------|
| 정수형 | `byte`, `short`, `int`, `long` |
| 실수형 | `float`, `double`              |
| 문자형 | `char`                         |
| 논리형 | `boolean`                      |

Primitive Type은 `null`을 가질 수 없고, 항상 기본값(`0`, `false`, `'\u0000'`)을 가집니다.
또한 메서드를 가지지 않으며, 크기가 고정되어 있어 메모리 사용이 예측 가능하다는 특징이 있습니다.

### Reference Type (참조 타입)

Reference Type은 Primitive Type을 제외한 **모든 타입**입니다.
클래스, 인터페이스, 배열, 열거형 등이 모두 여기에 해당합니다.

변수에는 객체 자체가 아니라 **객체가 위치한 Heap 영역의 주소**가 저장됩니다.
즉, 변수는 객체를 가리키는 "리모컨" 같은 역할을 합니다.

```java
int number = 10;         // Stack에 10이 직접 저장
String text = "hello";   // Stack에는 주소만, 실제 값은 Heap에
int[] arr = {1, 2, 3};   // Stack에는 주소만, 배열은 Heap에
```

## 🔎 실제 코드로 살펴보기

### 값 복사 vs 참조 복사

가장 큰 차이는 **할당과 복사가 일어날 때** 드러납니다.

```java
// Primitive Type - 값 자체가 복사된다
int a = 10;
int b = a;
b = 20;

System.out.println(a); // 10
System.out.println(b); // 20
```

```java
// Reference Type - 주소가 복사된다 (같은 객체를 가리킴)
int[] arr1 = {1, 2, 3};
int[] arr2 = arr1;
arr2[0] = 99;

System.out.println(arr1[0]); // 99
System.out.println(arr2[0]); // 99
```

`arr2`에 `arr1`을 대입했을 뿐인데 `arr1`의 값까지 바뀌어버립니다.
두 변수가 **같은 Heap 객체를 가리키고 있기 때문**입니다.
이 동작을 모르면 메서드에 객체를 넘긴 뒤 원본이 변경되는 부작용에 당황하게 됩니다.

### 메서드 인자 전달

Java는 항상 **Pass by Value(값에 의한 전달)** 입니다.
다만 Reference Type의 경우 "값"이 곧 "주소"이기 때문에 마치 참조로 넘어가는 것처럼 보일 뿐입니다.

```java
public static void modify(int x, int[] arr) {
    x = 100;
    arr[0] = 100;
}

int num = 1;
int[] nums = {1, 2, 3};
modify(num, nums);

System.out.println(num);     // 1 (변경되지 않음)
System.out.println(nums[0]); // 100 (변경됨)
```

`x`는 `num` 값을 복사한 별개의 지역 변수이지만,
`arr`는 `nums`와 같은 Heap 객체를 가리키는 또 다른 리모컨입니다.

## 🛠️ 주의해야 할 점들

### Wrapper 클래스와 Auto-boxing

Primitive Type은 객체가 아니기 때문에 컬렉션에 직접 담을 수 없습니다.
이를 위해 Java는 각 Primitive Type에 대응하는 **Wrapper 클래스**를 제공합니다.

| Primitive | Wrapper   |
|-----------|-----------|
| `int`     | `Integer` |
| `long`    | `Long`    |
| `double`  | `Double`  |
| `boolean` | `Boolean` |

```java
List<Integer> numbers = new ArrayList<>();
numbers.add(10);              // Auto-boxing: int -> Integer
int value = numbers.get(0);   // Auto-unboxing: Integer -> int
```

편리하지만 **숨겨진 비용**이 있습니다.
박싱이 일어날 때마다 새 객체가 생성되고, 언박싱 시 `null`이라면 `NullPointerException`이 터집니다.

```java
Integer maybeNull = null;
int x = maybeNull; // NullPointerException
```

### null의 존재 여부

이것이 두 타입의 가장 실질적인 차이일지도 모릅니다.

```java
int a = null;       // 컴파일 에러
Integer b = null;   // 정상
```

Reference Type은 "값이 없음"을 표현할 수 있지만,
바로 그 표현력 때문에 `NullPointerException`이라는 Java 개발자의 불편한 친구를 만들어냅니다.
그래서 `Optional`, 정적 분석 도구, `@NonNull` 같은 장치들이 등장한 것이기도 합니다.

### 비교 연산

Primitive Type은 `==`로 값을 비교하면 되지만, Reference Type은 다릅니다.

```java
int a = 100;
int b = 100;
System.out.println(a == b); // true (값 비교)

String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1 == s2);       // false (주소 비교)
System.out.println(s1.equals(s2));  // true (값 비교)
```

> Primitive는 `==`, Reference는 `equals()`를 쓴다.

이 한 줄을 잊지 않는 것만으로도 수많은 버그를 피할 수 있습니다.

## 💭 마치며

처음에는 "Primitive는 값, Reference는 주소" 정도로만 이해했었습니다.
하지만 시간이 지나면서 이 차이가 단순히 메모리 저장 방식의 문제가 아니라
**값의 의미와 객체의 생명주기를 어떻게 다룰 것인가** 라는 더 큰 질문과 연결되어 있다는 것을 알게 됐습니다.

값 타입은 복사되어 독립적으로 살아가고, 참조 타입은 공유되며 함께 변합니다.
그래서 도메인 모델을 설계할 때 "이 값은 복사되어야 하는가, 공유되어야 하는가?"를 고민하게 되고,
이 고민은 자연스럽게 불변 객체(Immutable Object)나 값 객체(Value Object)에 대한 관심으로 이어집니다.

기본 중의 기본이라고 여겼던 개념이지만, 돌아볼수록 객체지향과 메모리, 그리고 설계 철학까지 닿아 있다는 것을 느낍니다.
역시 기본기는 단순히 외워두는 것이 아니라, 계속 다시 들여다봐야 하는 것 같습니다.