---
title: "Mock API 도입기"
date: "2026-03-22"
tags: [ "Mock API" ]
description: "Mock API 도입기"
---

프론트엔드 개발자와 협업하다 보면 한 번쯤 이런 상황을 겪어 봤을 것 같습니다.
**'혹시 API 언제쯤 호출해 볼 수 있을까요 ?'** 백엔드 개발이 완료될 때까지 프론트엔드 개발자가 기다려야 하는 상황,
'퍼블리싱부터 진행하시면 안될까요...?'라고 했지만, 근본적인 해결책이 필요했습니다.

## ❓ Mock API를 도입하게된 이유

저희 팀은 프론트엔드 개발자와 백엔드 개발자가 동시에 개발을 진행하는 구조입니다.
그런데 실제로는 **백엔드 API가 완성될 때까지 프론트엔드 개발자가 온전히 작업을 진행하기 어려운** 상황이 반복됐습니다.

#### 👉🏻 UI는 완성됐는데 연동할 API가 없어 대기

#### 👉🏻 서로 다른 속도로 개발이 진행되다 보니 병목이 생기는 구간 명확

단순히 속도의 문제가 아니라 **팀 전체 흐름이 끊기는 문제**였습니다.
그래서 백엔드 개발이 완료되지 않아도 프론트 개발자가 연동 작업을 진행할 수 있는 환경을 만들기로 했습니다.

## 🤔 스택 선택하기

기존에는 프론트에서 더미 데이터를 만들어 사용하고 있었습니다.
Mock 데이터를 전달하기 위해 **Postman Mock Server**, **WireMock**, **Spring Mock API 구축**의 선택지가 있었고 동료들과
논의한 결과 **Spring Mock API 구축**이 가장 적합하다고 판단했습니다.

### 1. Postman Mock Server

프론트 측에서 기존과 같이 직접 Mock을 관리하는 방식입니다.
빠르게 시작할 수 있지만, 응답 스펙이 백엔드 코드와 분리되어 있어 실제 API와 불일치가 생길 가능성이 있었습니다.
또한 프론트 코드 베이스에 Mock 로직이 섞이는게 부담스러울 수 있다고 판단되었습니다.

### 2. WireMock

외부 API 격리 테스트에 강력하지만, 저희 목적은 **개발 서버에서 프론트가 실제처럼 연동하는 것**이었기 때문에
설정 대비 오버스펙이라 판단했습니다.

### 3. Spring Mock API 구축

결론적으로 **백엔드가 직접 Mock Controller를 제공하는 방식**이
스펙 불일치를 최소화하고, 프론트 입장에서도 실제 API와 동일한 경험을 줄 수 있어 최적이라고 판단했습니다.

## 🛠️ Spring Mock API 구축하기

- 실제 API와 **동일한 Endpoint, 동일한 응답 구조** 사용
- `@Profile("mock")`으로 Mock / 실제 환경 분리
- Mock 전용 패키지로 나중에 **한 번에 제거** 가능하게 구성

### 1. 패키지 구조

```
src/main/java/com/example/
├── controller/
│   └── UserController.java        ← 실제 컨트롤러
├── mock/
│   └── MockUserController.java    ← Mock 컨트롤러
└── dto/
    └── UserDto.java               ← DTO (미리 확정)
```

DTO와 Endpoint는 처음부터 확정하고 공유하는 것이 핵심입니다.
프론트가 Mock을 보고 개발하는 동안 스펙이 흔들리면 의미가 없어지기 때문입니다.

### 2. Mock Controller 작성

```java
@RestController
@RequestMapping("/api/users")
@Profile("mock")
public class MockUserController {

    @GetMapping("/{id}")
    public ResponseEntity getUser(@PathVariable Long id) {
        // 특정 ID로 에러 시나리오도 함께 제공
        if (id == 999L) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "유저 없음");
        }
        return ResponseEntity.ok(
            new UserDto(id, "홍길동", "hong@email.com", "ACTIVE")
        );
    }

    @GetMapping
    public ResponseEntity<List> getUsers() throws InterruptedException {
        Thread.sleep(300); // 실제 API처럼 로딩 상태 테스트 가능
        return ResponseEntity.ok(MockUserData.users());
    }
}
```

### 3. 실제 컨트롤러와 충돌 방지

같은 Endpoint를 두 컨트롤러가 사용하더라도 프로필로 분리하면 충돌이 없습니다.

```java
@RestController
@RequestMapping("/api/users")
@Profile("!mock")  // mock이 아닐 때만 활성화
public class UserController {
    // 실제 구현
}
```

### 4. Mock 데이터 분리

```java
public class MockUserData {
    public static List users() {
        return List.of(
            new UserDto(1L, "홍길동", "hong@email.com", "ACTIVE"),
            new UserDto(2L, "김철수", "kim@email.com", "INACTIVE")
        );
    }
}
```

## ✅ 도입 결과

Mock API를 배포한 뒤 프론트엔드 팀에서 바로 연동 작업을 시작할 수 있었습니다.
특히 에러 시나리오를 미리 제공해 에러 핸들링 UI도 병렬로 개발할 수 있어 생각보다 효과적이었습니다.

백엔드 개발이 완료된 후 프로필만 바꿔서 배포했고,
프론트 코드는 Endpoint와 응답 구조가 동일하기 때문에 수정이 거의 없었습니다.

## 💭 마치며

이 작업은 기술보다 **협업 흐름**에 가까운 문제였습니다.
백엔드가 조금 더 준비하면 프론트 개발자가 **기다리지 않아도 되는 구조**가 Mock API 도입의 핵심이었습니다.

구현 자체는 어렵지 않았습니다. 하지만 팀 전체 개발 흐름이 끊기는 문제를 해결하는 데 큰 도움이 되었습니다.
앞으로도 이런 작은 개선들이 팀 전체의 생산성을 높이는 데 큰 역할을 할 수 있다고 생각합니다.