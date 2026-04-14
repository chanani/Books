---
title: "EP.01 HashMap 전체 구조와 충돌 해결"
date: "2026-04-13"
episode: 1
tags: ["HashMap", "Java", "Data Structure"]
---

`HashMap`이 어떻게 작동하는지, 왜 그렇게 설계되었는지 궁금한 적이 있나요?
그래서 HashMap을 처음부터 끝까지 파헤쳐보기로 했습니다🙂

이 글은 HashMap Deep Dive 시리즈의 첫 번째 편입니다.

---

## 왜 HashMap인가?

`HashMap`은 Java에서 가장 많이 쓰이는 자료구조 중 하나입니다.
키-값 쌍을 저장하고, 키로 값을 `O(1)`에 꺼내올 수 있다는 점에서 강력합니다.

그런데 여기서 "왜 `O(1)`인가?"를 설명하려면 내부 구조를 알아야 합니다.
단순히 배열도 아니고, 연결리스트도 아닙니다. HashMap은 **두 자료구조를 결합**한 형태이고, Java 8부터는 상황에 따라 **트리**까지 사용합니다.

이번 편에서는 그 구조를 처음부터 끝까지 따라가 보려고합니다.

---

## HashMap의 뼈대: 버킷 배열

HashMap의 핵심은 **배열**입니다. 이 배열의 각 칸을 **버킷(bucket)** 이라고 부릅니다.

```
index:  [ 0 ]  [ 1 ]  [ 2 ]  [ 3 ]  [ 4 ]  [ 5 ]  [ 6 ]  [ 7 ]
         null   Node   null   null   Node   null   null   null
```

내부적으로는 `Node<K,V>[]` 타입의 배열인 `table`이 선언되어 있습니다.

```java
// HashMap.java (OpenJDK)
transient Node<K,V>[] table;

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;   // 연결리스트 포인터
}
```

`Node`가 단순한 키-값 쌍이 아닌 것을 주목하세요. `next` 포인터가 있어서 같은 버킷 안에서 **연결리스트**를 구성할 수 있습니다. 이게 왜 필요한지는 뒤에서 충돌을 다루면서 설명합니다.

---

## 키는 어느 버킷에 들어가는가 — hash()와 인덱스 계산

`put(key, value)`를 호출하면 HashMap은 먼저 키의 해시값을 계산합니다.

```java
// HashMap.java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

`hashCode()`를 그대로 쓰지 않고 `h ^ (h >>> 16)`을 적용합니다. 이를 **보조 해시(supplemental hash)** 라고 합니다.

배열 크기가 작을 때 상위 비트가 인덱스 계산에 반영되지 않아 충돌이 늘어나는 문제를 줄이기 위한 **비트 분산** 처리입니다.

이렇게 구한 해시값을 버킷 인덱스로 변환합니다.

```java
int index = (n - 1) & hash;
// n = table.length (항상 2의 거듭제곱)
```

> **상위 비트란?**  
> `hashCode()`는 32비트 정수를 반환합니다. 이 32비트를 절반으로 나누면 상위 16비트와 하위 16비트가 됩니다.  
> 배열 크기가 작을 때, 버킷 인덱스를 구하는 개산(hash & (n - 1))은 하위 비트만 사용합니다.

> **왜 n은 항상 2의 거듭제곱인가?**  
> `(n-1) & hash`는 `hash % n`과 결과가 같지만, 나눗셈보다 훨씬 빠릅니다.
> 이 최적화가 성립하려면 n이 반드시 2의 거듭제곱이어야 합니다.
> `new HashMap<>(10)`처럼 직접 지정해도 내부에서 다음 2의 거듭제곱(16)으로 올림 처리됩니다.

---

## put() 흐름 전체 따라가기

```java
map.put("hello", 42);
```

이 한 줄이 내부에서 거치는 단계입니다.

```
1. hash("hello") 계산         → 보조 해시 적용
2. index = (16-1) & hash      → 예: 2번 버킷
3. table[2] 확인
   ├─ null이면                → 새 Node 생성 후 저장 (끝)
   └─ 비어있지 않으면          → 아래로 계속
4. 버킷 안 순회
   ├─ 같은 키 발견             → value 덮어쓰기 (끝)
   └─ 끝까지 없음             → 연결리스트/트리 끝에 추가
5. size 증가 후 resize 필요 여부 확인
```

코드로 보면 핵심 부분은 이렇습니다.

```java
final V putVal(int hash, K key, V value, ...) {
    // 배열이 비어있으면 초기화
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    // 버킷이 비어있으면 바로 삽입
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 트리 노드면 트리에 삽입
        if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 연결리스트 순회
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 8개 이상이 되면 트리로 전환
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash && key.equals(e.key)) break;
                p = e;
            }
        }
    }
    // threshold 초과 시 배열 확장
    if (++size > threshold) resize();
}
```

`get()`은 더 단순합니다. 같은 방식으로 인덱스를 구한 뒤, 버킷 안에서 `equals()`로 키를 찾아 반환합니다.

---

## load factor와 resize — 배열은 언제 늘어나는가

데이터가 쌓일수록 충돌이 늘어납니다. 이를 막기 위해 항목이 일정 수를 넘으면 배열을 **두 배로 확장(resize)** 합니다.

**threshold = capacity × load factor**

```
기본 capacity   = 16
기본 load factor = 0.75
threshold       = 16 × 0.75 = 12
```

즉, 항목이 12개를 넘는 순간 `resize()`가 호출됩니다.

| load factor | 충돌 빈도 | 메모리 |
|-------------|---------|-------|
| 낮게 설정 | 줄어듦 | 많이 씀 |
| **0.75 (기본값)** | **균형** | **균형** |
| 높게 설정 | 늘어남 | 아낌 |

`0.75`는 공간 효율과 충돌 빈도 사이의 실험적 균형값입니다.

`resize()`는 새 배열을 만들고 기존 모든 노드를 **재배치(rehash)** 합니다. 이 과정은 `O(n)`입니다.

```
실전 팁:
넣을 데이터 개수를 미리 안다면 아래처럼 초기 capacity를 지정해
resize 비용을 없앨 수 있습니다.

new HashMap<>(expectedSize / 0.75 + 1)
```

---

## 해시 충돌 — 왜 생기고 어떻게 해결하는가

**해시 충돌**이란 서로 다른 키가 같은 버킷 인덱스를 갖는 상황입니다.

```java
// "Aa"와 "BB"는 실제로 같은 hashCode()를 가집니다.
// 'A'(65) * 31 + 'a'(97) = 'B'(66) * 31 + 'B'(66) = 2112

hash("Aa") & 15 == hash("BB") & 15   // 충돌 발생!
```

HashMap은 이 충돌을 **Separate Chaining** 방식으로 해결합니다.
같은 버킷에 들어온 노드들을 연결리스트로 이어붙이는 방식입니다.

```
table[5] → Node("Aa", 1) → Node("BB", 2) → null
```

연결리스트 탐색은 `O(n)`입니다.
충돌이 많을수록 `get()`이 느려진다는 뜻이고, 악의적으로 모든 키를 같은 버킷에 몰아넣으면 **Hash DoS** 공격도 가능합니다. Java 8이 이 문제를 어떻게 해결했는지가 다음 주제입니다.

---

## Java 8의 해결책 — treeifyBin

Java 8은 연결리스트가 일정 길이 이상이 되면 **Red-Black Tree**로 전환하는 방식을 도입했습니다.

```java
static final int TREEIFY_THRESHOLD     = 8;   // 트리 전환 임계값
static final int UNTREEIFY_THRESHOLD   = 6;   // 리스트 복귀 임계값
static final int MIN_TREEIFY_CAPACITY  = 64;  // 트리 전환 최소 배열 크기
```

### 트리 전환 조건

두 조건을 **모두** 만족해야 합니다.

```
버킷 안의 노드 수 ≥ 8
                   AND
전체 배열 크기(capacity) ≥ 64
```

capacity가 64 미만이면 treeify 대신 `resize()`를 먼저 수행합니다.
배열이 충분히 크지 않은 상태에서는 트리 전환보다 배열을 늘리는 게 더 효과적이기 때문입니다.

### 왜 하필 8인가?

버킷 안 노드 수는 **포아송 분포**를 따릅니다.
load factor 0.75 기준으로 길이가 8 이상이 될 확률은 약 **0.00000006(천만 분의 6)** 입니다.

```
길이 0 : 0.60653066
길이 1 : 0.30326533
길이 2 : 0.07581633
길이 3 : 0.01263606
길이 4 : 0.00157952
길이 5 : 0.00015795
길이 6 : 0.00001316
길이 7 : 0.00000094
길이 8 : 0.00000006  ← 여기서 트리 전환
```

정상적인 사용 환경에서는 트리가 거의 등장하지 않습니다.
반대로 충돌이 의도적으로 유발되는 상황에서는 트리 덕분에 최악의 경우에도 `O(log n)`을 보장합니다.

### 트리 전환 후의 구조

```
전환 전                          전환 후

table[5]                         table[5]
   │                                │
Node("Aa")                      TreeNode (루트)
   │                             ╱         ╲
Node("BB")                  TreeNode     TreeNode
   │                                         ╲
Node("Ca")                                TreeNode
   │
Node("Db")
   │
  ...  (8개 이상)
```

`Node` → `TreeNode`로 교체되며, Red-Black Tree의 균형을 유지합니다.

### 다시 연결리스트로 — untreeify

데이터가 삭제되어 노드 수가 `UNTREEIFY_THRESHOLD(6)` 이하로 줄어들면 다시 연결리스트로 복원됩니다.

임계값이 8이 아닌 6인 이유는 **oscillation(진동) 방지**입니다.
같은 값으로 설정하면 노드 추가/삭제가 반복될 때 트리↔리스트 전환이 계속 일어나는 오버헤드가 생깁니다.

---

## 전체 흐름 한눈에 보기

```
put(key, value) 호출
        │
        ▼
  hash(key) 계산
  (hashCode() ^ 보조 해시)
        │
        ▼
  index = (capacity - 1) & hash
        │
        ├─ table[index] == null?
        │       └─ YES → 새 Node 저장  ✓
        │
        └─ table[index] != null
                ├─ TreeNode → 트리 삽입 O(log n)
                └─ LinkedList 순회
                        ├─ 같은 키 발견 → value 덮어쓰기  ✓
                        └─ 없음 → 끝에 추가
                                └─ 노드 수 ≥ 8 && capacity ≥ 64?
                                        └─ YES → treeifyBin()
        │
        ▼
  size > threshold (capacity × 0.75)?
        └─ YES → resize() (2배 확장 + rehash)
```

---

## 💭 마치며

이번 편에서 다룬 내용을 정리하면 다음과 같습니다.

| 주제 | 핵심 내용 |
|------|---------|
| 기본 구조 | 버킷 배열 + 연결리스트 + (Java 8+) Red-Black Tree |
| hash() | `hashCode() ^ (h >>> 16)` — 비트 분산으로 충돌 감소 |
| 인덱스 계산 | `(capacity - 1) & hash` — 나눗셈 없이 빠르게 |
| load factor | 기본 0.75 — 공간 효율과 충돌 빈도의 균형점 |
| 해시 충돌 해결 | Separate Chaining (연결리스트) |
| treeifyBin | 노드 8개 이상 + capacity 64 이상 → Red-Black Tree 전환 |
| TREEIFY_THRESHOLD 8 | 포아송 분포 기반 — 정상 환경에서는 거의 발생 안 함 |
| untreeify | 노드 6개 이하 → 다시 연결리스트 (진동 방지) |

다음 편에서는 이 구조가 올바르게 작동하기 위한 전제 조건을 다룰 예정입니다.
