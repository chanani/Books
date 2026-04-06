# dev 폴더 구조 가이드

dev 폴더는 **일반 글**과 **시리즈** 두 가지 유형을 지원합니다.

---

## 1. 일반 글

```
dev/
└── {category}/
    └── {slug}/
        ├── {slug}.md    ← 본문
        └── cover.png    ← 대표 이미지 (선택)
```

### front matter 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | string | O | 글 제목 |
| `date` | string | O | 작성일 (`YYYY-MM-DD`) |
| `tags` | string[] | | 태그 목록 |
| `description` | string | | 글 간단 설명 |

### 예시

```
dev/
└── spring/
    └── circuit-breaker/
        ├── circuit-breaker.md
        └── cover.png
```

```yaml
---
title: "서킷 브레이커 패턴 이해하기"
date: "2026-04-06"
tags: ["Spring", "MSA", "Resilience4j"]
description: "서킷 브레이커 패턴의 개념과 Spring 적용 방법을 정리합니다."
---
```

---

## 2. 시리즈

```
dev/
└── {category}/
    └── {series-slug}/
        ├── info.json              ← 시리즈 정보 (필수, 시리즈 판별 기준)
        ├── cover.png              ← 시리즈 대표 이미지 (선택)
        ├── {episode-slug}/        ← 에피소드 폴더
        │   └── {episode-slug}.md
        └── {episode-slug}/
            └── {episode-slug}.md
```

### info.json 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | string | O | 시리즈 제목 |
| `description` | string | | 시리즈 설명 |
| `status` | string | | 연재 상태 (`연재중` \| `완결`) |
| `tags` | string[] | | 태그 목록 |
| `updatedAt` | string | | 최근 업데이트일 (`YYYY-MM-DD`) |

### status 값

| 값 | 의미 | 뱃지 색상 |
|----|------|-----------|
| `연재중` | 진행 중인 시리즈 | 파랑 |
| `완결` | 완료된 시리즈 | 초록 |

### 에피소드 front matter 필드

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | string | O | 에피소드 제목 |
| `date` | string | O | 작성일 (`YYYY-MM-DD`) |
| `episode` | number | O | 에피소드 번호 (목록 정렬 기준) |
| `tags` | string[] | | 태그 목록 |
| `description` | string | | 에피소드 간단 설명 |

### 예시

```
dev/
└── java/
    └── jvm-deep-dive/
        ├── info.json
        ├── cover.png
        ├── ep1-jvm-intro/
        │   └── ep1-jvm-intro.md
        └── ep2-classloader/
            └── ep2-classloader.md
```

**info.json**

```json
{
  "title": "JVM Deep Dive",
  "description": "JVM 내부 구조를 단계별로 파헤치는 시리즈.",
  "status": "연재중",
  "tags": ["JVM", "Java", "백엔드"],
  "updatedAt": "2026-04-06"
}
```

**ep1-jvm-intro.md**

```yaml
---
title: "[JVM] EP.1 JVM이란 무엇인가"
date: "2026-04-06"
episode: 1
tags: ["JVM", "Java"]
description: "JVM의 기본 개념과 전체 구조를 살펴봅니다."
---
```

---

## 주의사항

- `info.json`이 없으면 시리즈가 아닌 일반 글 폴더로 인식합니다.
- 에피소드는 반드시 **폴더** 형태여야 하며, 폴더명이 URL slug로 사용됩니다.
- 에피소드 폴더명은 영문 소문자와 하이픈(`-`)을 권장합니다.
- `episode` 필드가 없으면 목록에서 폴더명 기준으로 정렬됩니다.
