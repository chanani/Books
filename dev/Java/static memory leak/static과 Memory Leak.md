---
title: "static 키워드 사용 시 Memory Leak 발생 가능 상황"
date: "2026-05-02"
tags: [ "Java", "static", "Memory Leak", "GC", "JVM" ]
description: "static 키워드는 편리하지만 잘못 쓰면 GC가 회수할 수 없는 메모리를 만들어낸다. 실무에서 자주 마주치는 Memory Leak 시나리오와 해결 방법을 정리한다."
---

`static` 키워드는 처음 Java를 배울 때 "어디서든 쓸 수 있게 해주는 것" 정도로 익히고 넘어가는 경우가 많습니다.
그런데 운영 서버에서 OOM(OutOfMemoryError) 로그를 들여다 보면,
범인의 한가운데에 `static` 필드가 앉아있는 경우를 너무 자주 마주치게 됩니다.

## ⭐️ static이 메모리에서 가지는 의미

Java에서 `static` 멤버는 인스턴스가 아니라 **클래스에 소속**됩니다.
JVM이 클래스를 로드할 때 메서드 영역(Method Area, JDK 8 이후로는 Metaspace)에 적재되며,
**해당 클래스를 로드한 ClassLoader가 살아있는 동안에는 GC의 대상이 되지 않습니다.**

이는 다음을 의미합니다.

- `static` 필드는 **GC Root**로 취급된다.
- `static` 필드가 가리키는 객체는 그 참조를 끊지 않는 한 영원히 살아남는다.
- "클래스가 살아있는 동안"의 라이프사이클은 보통 **애플리케이션 수명 전체**와 같다.

| 구분          | 저장 위치                | 라이프사이클       |
|-------------|----------------------|--------------|
| 인스턴스 필드     | Heap (객체 단위)         | 객체가 GC될 때까지  |
| `static` 필드 | Method Area (클래스 단위) | 클래스 언로드 시점까지 |

쉽게 말해, **`static`은 GC가 함부로 손댈 수 없는 안전 지대**입니다.
이 안전 지대에 어떤 객체를 슬쩍 밀어넣으면, 그 객체는 사실상 영원히 메모리에 남게 됩니다.

## 🔎 실무에서 자주 발생하는 Memory Leak 시나리오

### 1. 무한히 커지는 static Collection

가장 흔한 패턴입니다.
캐시처럼 보이지만 만료 정책이 없어서, 애플리케이션이 살아있는 한 계속 자라기만 합니다.

```java
public class UserCache {
    private static final Map<Long, User> CACHE = new HashMap<>();

    public static void put(Long id, User user) {
        CACHE.put(id, user); // 한 번 들어가면 절대 사라지지 않는다
    }

    public static User get(Long id) {
        return CACHE.get(id);
    }
}
```

처음에는 잘 동작합니다.
그러다 트래픽이 쌓이고, 하루, 일주일, 한 달이 지나면
어느 순간 GC 로그에 Full GC가 끊임없이 찍히고 결국 OOM으로 쓰러집니다.

`static` Map은 절대 비워지지 않기 때문에, 한 번 들어간 `User` 객체는 사용 여부와 관계없이 살아남습니다.

### 2. 등록만 하고 해제하지 않는 Listener

이벤트 기반 시스템에서 자주 발생하는 문제입니다.

```java
public class EventBus {
    private static final List<EventListener> LISTENERS = new ArrayList<>();

    public static void register(EventListener listener) {
        LISTENERS.add(listener);
    }

    // unregister가 빠져있다면?
}
```

`EventBus`는 클래스 차원에서 살아있고, 등록된 리스너는 그 안의 `static` 컬렉션이 잡고 있습니다.
리스너가 어떤 화면 객체나 서비스 빈에서 만들어졌다면,
그 객체와 그 객체가 들고 있는 모든 의존성까지 함께 메모리에 갇힙니다.

### 3. static 필드가 참조하는 비-static 내부 클래스

이건 조금 더 미묘합니다.
**비-static 내부 클래스(non-static inner class)는 외부 클래스의 참조를 암묵적으로 보유**합니다.

```java
public class ReportService {
    private final HeavyDependency dependency;

    private class ReportTask implements Runnable {
        @Override
        public void run() {
            // 바깥 ReportService의 dependency를 쓸 수 있다
            dependency.process();
        }
    }

    private static ReportTask currentTask; // 위험!
}
```

`ReportTask`는 `ReportService`의 인스턴스를 암묵적으로 참조합니다.
그런데 이것을 `static` 필드에 넣어버리면, `ReportService` 인스턴스 전체가 GC 대상에서 제외됩니다.
필요한 것은 작은 작업 객체였는데, 거대한 서비스 객체와 그 의존성 트리 전체가 함께 살아남게 되는 것입니다.

> 내부 클래스를 `static` 필드에 보관할 일이 있다면, 거의 항상 `static class`로 선언해야 한다.

### 4. ThreadLocal과 Thread Pool의 조합

`ThreadLocal` 자체는 `static`이 아닐 수도 있지만, **거의 대부분 `static final`로 선언**됩니다.
단독으로 쓸 때는 문제가 없지만, **Thread Pool**과 만나면 이야기가 달라집니다.

```java
public class RequestContext {
    private static final ThreadLocal<UserSession> SESSION = new ThreadLocal<>();

    public static void set(UserSession session) {
        SESSION.set(session);
    }

    public static UserSession get() {
        return SESSION.get();
    }
}
```

웹 서버의 요청 처리 스레드는 보통 풀에서 재사용됩니다.
요청이 끝난 뒤 `SESSION.remove()`를 호출하지 않으면,
그 `UserSession` 객체는 다음 요청 때까지 그 스레드에 남아있게 됩니다.
단순한 누수를 넘어, **이전 사용자의 정보가 다음 요청에 노출되는 보안 사고**로 이어질 수도 있는 위험한 패턴입니다.

```java
try {
    RequestContext.set(currentSession);
    // 요청 처리
} finally {
    RequestContext.clear(); // 반드시 정리한다
}
```

### 5. 종료되지 않는 static 자원

`static`으로 선언된 스레드, 스케줄러, 커넥션, 파일 핸들도 같은 문제를 일으킵니다.

```java
public class Scheduler {
    private static final ScheduledExecutorService EXECUTOR =
        Executors.newScheduledThreadPool(4);

    // shutdown 코드가 없다면 애플리케이션 종료 시점에도 문제가 된다
}
```

특히 WAS에서 애플리케이션을 재배포(hot deploy)할 때, 이전 ClassLoader가 GC되지 않고 남아
**ClassLoader Leak**의 주요 원인이 되기도 합니다.
배포할수록 메모리가 줄어드는 현상의 범인은 대부분 이 부류입니다.

## 🛠️ 어떻게 막을 것인가

### WeakReference 계열 활용

키가 다른 곳에서 더 이상 참조되지 않을 때 자동으로 사라지길 원한다면 `WeakHashMap`이 도움이 됩니다.

```java
private static final Map<Key, Value> CACHE = new WeakHashMap<>();
```

다만 **값(Value)이 아니라 키(Key)에 대한 약한 참조**라는 점,
그리고 키를 어딘가에서 강하게 잡고 있으면 의미가 없어진다는 점을 잘 이해하고 써야 합니다.

### 만료 정책이 있는 캐시 라이브러리 사용

직접 만든 `static Map` 캐시는 시한폭탄과 같습니다.
**Caffeine**, **Guava Cache** 같은 검증된 라이브러리로 크기 제한과 만료 시간을 명시하는 편이 안전합니다.

```java
private static final Cache<Long, User> CACHE = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(10))
    .build();
```

### 등록한 만큼 해제하기

리스너, 옵저버, ThreadLocal은 **등록과 해제가 한 쌍으로 움직여야 합니다.**
가능하다면 `try-finally`로 묶어서 누락 가능성을 구조적으로 차단하는 편이 좋습니다.

### 그리고, 정말 static이어야 하는지 다시 묻기

가장 근본적인 방안입니다.
어떤 필드를 `static`으로 선언하기 전에 한 번만 더 생각해보면 많은 누수를 미리 막을 수 있습니다.

- 정말 클래스 단위로 공유되어야 하는 상태인가?
- 인스턴스 단위로 가져도 되지 않는가?
- DI 컨테이너가 관리하는 빈으로 두면 충분하지 않은가?

대부분의 경우 `static` 가변 상태(mutable state)는
**전역 변수의 다른 이름**일 뿐이고, 전역 변수가 그렇듯 시간이 지나면 발목을 잡습니다.

## 💭 마치며

`static`은 편리합니다.
어디서든 호출할 수 있고, 인스턴스를 만들 필요도 없고, 마치 공짜처럼 느껴집니다.
그래서 처음에는 "공통으로 쓰니까 static으로"라는 결정이 가볍게 내려지곤 합니다.

하지만 `static`이 약속하는 "클래스 수명만큼의 영속성"은 양날의 검입니다.
필요할 때는 강력한 도구이지만, 그렇지 않을 때는 GC가 손댈 수 없는 누수의 통로가 됩니다.

Memory Leak이 무서운 이유는 단번에 드러나지 않는다는 점입니다.
개발 환경, 단위 테스트, 짧은 부하 테스트로는 잘 잡히지 않다가,
운영 환경에서 며칠 혹은 몇 주가 지난 뒤에야 OOM과 함께 신호를 보냅니다.
그래서 발견했을 때는 이미 원인 추적이 어렵고 한참을 헤매게 됩니다.

결국 `static`을 잘 쓴다는 건 단순한 문법의 문제가 아니라,
**"이 상태는 누가 책임지고 정리할 것인가?"** 라는 질문에 답할 수 있느냐의 문제인 것 같습니다.
편리함을 빌려오는 만큼, 그 책임을 잊지 않으려 합니다.