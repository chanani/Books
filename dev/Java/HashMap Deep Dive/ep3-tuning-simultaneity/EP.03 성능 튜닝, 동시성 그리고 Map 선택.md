---
title: "EP.03 성능 튜닝, 동시성 그리고 Map 선택"
date: "2026-04-15"
episode: 3
tags: [ "HashMap", "Java", "ConcurrentHashMap", "Performance" ]
---

앞선 두 편에서 HashMap의 구조와 `equals()` / `hashCode()` 계약을 살펴봤습니다.
이번 편은 실전입니다. HashMap을 더 빠르게 쓰는 법, 멀티스레드 환경에서 왜 터지는지, 그리고 상황에 따라 어떤 Map을 골라야 하는지를 다룹니다.

이 글은 HashMap Deep Dive 시리즈의 마지막 편입니다.

---

## 성능 튜닝 — resize를 피하는 것부터

EP.01에서 `threshold`를 초과하면 배열을 두 배로 확장하고 모든 노드를 재배치(rehash)한다고 했습니다. 이 `resize()`는 `O(n)`입니다.

데이터를 얼마나 넣을지 미리 알고 있다면 **처음부터 capacity를 넉넉하게** 잡아서 resize 자체를 없앨 수 있습니다.

### 초기 capacity 계산법

```
threshold = capacity × load factor (0.75)
capacity  = expectedSize / 0.75 + 1
```

```java
int expectedSize = 1000;
Map<String, String> map = new HashMap<>((int)(expectedSize / 0.75) + 1);
// capacity = 1334 → 내부에서 다음 2의 거듭제곱인 2048로 올림
// resize 없이 1000개를 안전하게 담을 수 있음
```

### 자주 쓰는 Map 메서드 — getOrDefault, computeIfAbsent, merge

실전에서 자주 쓰이지만 덜 알려진 메서드들입니다.

**getOrDefault** — 키가 없을 때 기본값 반환

```java
// 기존 방식
String val = map.containsKey(key) ? map.get(key) : "default";

// 깔끔한 방식
String val = map.getOrDefault(key, "default");
```

**computeIfAbsent** — 키가 없을 때만 값을 생성해서 저장

```java
// 그룹핑할 때 자주 쓰는 패턴
Map<String, List<String>> groups = new HashMap<>();

// 기존 방식
if (!groups.containsKey(key)) {
    groups.put(key, new ArrayList<>());
}
groups.get(key).add(value);

// 깔끔한 방식
groups.computeIfAbsent(key, k -> new ArrayList<>()).add(value);
```

**merge** — 키가 있으면 병합, 없으면 삽입

```java
// 단어 빈도 계산
Map<String, Integer> freq = new HashMap<>();

// 기존 방식
freq.put(word, freq.getOrDefault(word, 0) + 1);

// merge 방식
freq.merge(word, 1, Integer::sum);
```

---

## HashMap은 thread-unsafe — 직접 재현

`HashMap`은 멀티스레드 환경에서 사용해선 안 됩니다. 이유를 코드로 확인해보겠습니다.

```java
Map<Integer, Integer> map = new HashMap<>();

// 두 스레드가 서로 다른 키 범위를 동시에 put
Thread t1 = new Thread(() -> {
    for (int i = 0; i < 10000; i++) {
        map.put(i, i);              // 키: 0 ~ 9999
    }
});
Thread t2 = new Thread(() -> {
    for (int i = 10000; i < 20000; i++) {
        map.put(i, i);              // 키: 10000 ~ 19999
    }
});

t1.start();
t2.start();
t1.join();
t2.join();

System.out.println(map.size());   // 20000이 나와야 정상
                                  // 실제로는 19873, 19542 같은
                                  // 들쭉날쭉한 값이 출력됨
                                  // (Java 7 이하에선 무한루프 가능)
```

### 왜 터지는가?

원인은 세 가지입니다.

**① 같은 버킷에 동시 삽입 → 노드 유실**

```
Thread-1: tab[i] 확인 → null
Thread-2: tab[i] 확인 → null
Thread-1: tab[i] = newNode(K1)
Thread-2: tab[i] = newNode(K2)   ← K1 덮어씀
```

**② `++size` 의 race condition → 카운터 어긋남**

`++size`는 read → modify → write 3단계라 원자적이지 않습니다. 두 스레드가 같은 값을 읽고 같은 값을 쓰면 한 번의 증가가 사라집니다.

**③ `resize()` 도중 충돌 → 가장 치명적**

- Java 7 이하: 연결리스트가 순환 구조가 되어 `get()` 호출 시 무한루프
- Java 8 이후: 무한루프는 해결됐으나 노드 유실, 가끔 NPE 발생

  | Java 버전 | 증상 |
    |-----------|------|
  | Java 7 이하 | 무한루프 + 데이터 유실 |
  | Java 8 이후 | 데이터 유실 + size 불일치 |

Java 8 이후가 어떤 면에선 더 위험합니다. 프로세스가 죽지 않고 조용히 데이터만 깨지거든요.

멀티스레드 환경이라면 `HashMap` 대신 `ConcurrentHashMap`을 써야 합니다.
 
---

## ConcurrentHashMap — 어떻게 thread-safe를 달성하는가

`ConcurrentHashMap`은 전체 맵을 잠그는 대신 **버킷 단위로 락**을 겁니다.

```
Java 7: Segment 분할 (기본 16개 세그먼트)
Java 8: 버킷 단위 synchronized + CAS 연산
```

### Java 8의 접근법

```java
// ConcurrentHashMap.java (핵심 부분)
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 빈 버킷 → CAS(Compare-And-Swap)로 락 없이 삽입
    if (casTabAt(tab, i, null, new Node<>(hash, key, value)))
        break;

    // 이미 노드 있음 → 해당 버킷 헤드 노드만 synchronized
    synchronized (f) {
        // 연결리스트 또는 트리 탐색
    }
}
```

```
빈 버킷 삽입  → CAS 연산  (락 없음, 빠름)
충돌 발생     → 버킷 단위 synchronized (좁은 범위)
읽기(get)    → 락 없음 (volatile 읽기)
```

덕분에 여러 스레드가 서로 다른 버킷에 동시에 쓸 수 있어 `Hashtable`(전체 락)보다 훨씬 빠릅니다.

### 주의사항 — 복합 연산

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// ❌ thread-unsafe — get과 put 사이에 다른 스레드가 끼어들 수 있음
if (!map.containsKey(key)) {
    map.put(key, 1);
}

// ✅ thread-safe — 원자적으로 처리
map.putIfAbsent(key, 1);
map.computeIfAbsent(key, k -> 1);
map.merge(key, 1, Integer::sum);
```

개별 `put()` / `get()`은 thread-safe하지만, 여러 연산을 조합하면 그 사이에 다른 스레드가 끼어들 수 있습니다. 복합 연산은 반드시 `computeIfAbsent`, `merge` 같은 원자적
메서드를 써야 합니다.

---

## 불변 Map — Map.of()와 unmodifiableMap의 차이

읽기 전용 Map이 필요할 때 두 가지 선택지가 있습니다.

```java
// 방법 1: Collections.unmodifiableMap
Map<String, Integer> base = new HashMap<>();
base.put("a", 1);
Map<String, Integer> unmod = Collections.unmodifiableMap(base);

// 방법 2: Map.of() (Java 9+)
Map<String, Integer> immutable = Map.of("a", 1, "b", 2);
```

### 결정적 차이

```java
Map<String, Integer> base = new HashMap<>();
base.put("a", 1);
Map<String, Integer> unmod = Collections.unmodifiableMap(base);

base.put("b", 2);                    // 원본 수정
System.out.println(unmod.get("b")); // 2 ← 같이 바뀜!
```

`unmodifiableMap`은 **원본의 뷰(view)** 입니다. 원본이 바뀌면 같이 바뀝니다.
`Map.of()`는 진짜 불변입니다. 원본이 없고, 수정 시도 시 `UnsupportedOperationException`을 던집니다.

```
Collections.unmodifiableMap  →  원본 뷰, 원본 변경에 영향받음
Map.of()                     →  진짜 불변, null 키/값 불허, 순서 미보장
Map.copyOf()                 →  기존 맵을 복사해 진짜 불변으로 만듦
```

---

## Map 선택 가이드 — 상황별 정리

실전에서 어떤 Map을 골라야 할지 한눈에 정리합니다.

### Map 종류와 특성

| Map                 | 순서    | null 허용   | thread-safe | 시간복잡도    |
|---------------------|-------|-----------|-------------|----------|
| `HashMap`           | 없음    | 키/값 모두    | ❌           | O(1) 평균  |
| `LinkedHashMap`     | 삽입 순서 | 키/값 모두    | ❌           | O(1) 평균  |
| `TreeMap`           | 정렬 순서 | 값만        | ❌           | O(log n) |
| `ConcurrentHashMap` | 없음    | 키/값 모두 불허 | ✅           | O(1) 평균  |
| `Map.of()`          | 없음    | 불허        | ✅ (읽기)      | O(1) 평균  |

### 상황별 선택

```
일반적인 키-값 저장
  └─ HashMap

삽입 순서를 유지해야 함 (예: LRU 캐시)
  └─ LinkedHashMap

키를 정렬된 순서로 순회해야 함
  └─ TreeMap

멀티스레드 환경에서 읽기/쓰기가 모두 빈번함
  └─ ConcurrentHashMap

초기화 이후 변경 없는 설정값, 상수 맵
  └─ Map.of() / Map.copyOf()
```

### LinkedHashMap으로 LRU 캐시 만들기

```java
// accessOrder = true로 설정하면 접근 순서로 정렬됨
Map<String, String> lruCache = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
        return size() > 100;   // 100개 초과 시 가장 오래된 항목 제거
    }
};
```

---

## Java 21 SequencedMap — 새로운 선택지

Java 21에서 `SequencedMap` 인터페이스가 추가됐습니다.

```java
// SequencedMap 인터페이스
public interface SequencedMap<K, V> extends Map<K, V> {
    Map.Entry<K, V> firstEntry();
    Map.Entry<K, V> lastEntry();
    Map.Entry<K, V> pollFirstEntry();
    Map.Entry<K, V> pollLastEntry();
    SequencedMap<K, V> reversed();   // 역순 뷰
}
```

`LinkedHashMap`과 `TreeMap`이 이 인터페이스를 구현합니다.

```java
LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
map.put("a", 1);
map.put("b", 2);
map.put("c", 3);

System.out.println(map.firstEntry());  // a=1
System.out.println(map.lastEntry());   // c=3

map.reversed().forEach((k, v) ->
    System.out.println(k + "=" + v));  // c=3, b=2, a=1
```

기존에는 `LinkedHashMap`에서 첫 번째/마지막 항목을 꺼내려면 iterator를 직접 다뤄야 했는데, `SequencedMap` 덕분에 훨씬 직관적이 됐습니다.

---

## 💭 마치며

3편에 걸쳐 HashMap을 처음부터 끝까지 파헤쳐봤습니다.

| 주제                          | 핵심 내용                                              |
|-----------------------------|----------------------------------------------------|
| 초기 capacity                 | `expectedSize / 0.75 + 1` — resize 비용 선제 차단        |
| computeIfAbsent / merge     | put + get 조합을 원자적 연산 하나로 대체                        |
| HashMap thread-unsafe       | 동시 접근 시 데이터 유실 · 무한루프 위험                           |
| ConcurrentHashMap           | 버킷 단위 락 + CAS — 높은 동시성 처리                          |
| 복합 연산 주의                    | containsKey + put 대신 putIfAbsent · computeIfAbsent |
| Map.of() vs unmodifiableMap | 진짜 불변 vs 원본 뷰                                      |
| Map 선택 기준                   | 순서 · 동시성 · 불변 여부에 따라 다름                            |
| Java 21 SequencedMap        | firstEntry · lastEntry · reversed() 지원             |

세 편을 다 읽고 나면 HashMap을 "그냥 쓰는 것"에서 "이해하고 쓰는 것"으로 한 단계 넘어설 수 있다고 생각합니다.
다음 시리즈에서는 HashMap과 뗄 수 없는 주제인 **Java 메모리 모델과 GC**를 다뤄볼 예정입니다.