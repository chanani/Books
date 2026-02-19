# info.json 작성 가이드

각 책 폴더 내부의 `info.json` 파일에 책 정보를 작성합니다.

## 필드 설명

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `title` | string | O | 책 제목 |
| `subtitle` | string | | 부제목 |
| `author` | string | | 저자 |
| `publisher` | string | | 출판사 |
| `totalPages` | number | | 총 페이지 수 |
| `category` | string | | 카테고리 (예: `Programming`, `Design`) |
| `rating` | number | | 평점 (0 ~ 5, 0.5 단위) |
| `tags` | string[] | | 태그 목록 |
| `excerpt` | string | | 한 줄 소개 |
| `date` | string | | 등록일 (`YYYY-MM-DD`) |
| `status` | string | | 독서 상태 |

## status 값

| 값 | 의미 | 뱃지 색상 |
|----|------|-----------|
| `독서중` | 현재 읽고 있는 책 | 주황 |
| `완독` | 다 읽은 책 | 초록 |
| `예정` | 읽을 예정인 책 | 노랑 |
| `중단` | 읽다가 중단한 책 | 빨강 |

## 예시

```json
{
  "title": "오브젝트",
  "subtitle": "역할, 책임, 협력을 향해 객체지향적으로 프로그래밍하라!",
  "author": "조영호",
  "publisher": "위키북스",
  "totalPages": 656,
  "category": "Programming",
  "rating": 5,
  "tags": ["객체지향", "설계", "책임 주도 설계", "OOP"],
  "excerpt": "역할과 책임, 협력을 기반으로 객체지향 설계를 이해하고 구현하는 방법을 다룬 책.",
  "date": "2026-01-30",
  "status": "독서중"
}
```

## 폴더 구조

```
books/
└── 오브젝트/
    ├── info.json        ← 책 정보
    ├── cover.png        ← 표지 이미지 (cover.jpg/png/webp)
    ├── 01-객체-설계.md   ← 챕터 파일
    ├── 02-객체지향.md
    └── 부록/             ← 하위 폴더 (그룹)
        └── 참고자료.md
```
