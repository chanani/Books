---
title: "AI 코드 리뷰 구축기"
date: "2026-03-03"
tags: [ "Claude", "코드 리뷰", "claude-code-action" ]
description: "AI 코드 리뷰 구축기"
---

작년 사내에서 사용할 수 있는 **AI 코드 리뷰** 라이브러리를 제작했습니다.
라이브러리를 구축했지만, 특정 언어에만 제한되고 독립 서버에서 실행하기 위한 파이프라인을 서버마다 반복 구축해야되는 번거로움이 있었습니다.

단점을 보완할 수 있는 방법을 찾던 중 우연히 **무신사의 AI 코드 리뷰 구축기** 블로그 글을 보게 되었고,
저와 비슷한 고민을 하고 있어 해당 블로그를 참고해 작업을 이어가기로 했습니다.

## 🙋🔍 AI 코드 리뷰 라이브러리를 구축한 이유

기존에 사내 코드 리뷰 문화가 존재하지 않았고, 코드 리뷰 문화를 도입하려는 시도 역시 물거품으로 돌아갔습니다.
현실적으로 바쁜 일정 속에서 실시간으로 세밀하게 PR을 확인하기에는 물리적인 시간 제약이 컸습니다.
그래서 저는 **리뷰의 진입장벽을 낮추고 최소한의 코드 품질을 자동화된 방식으로 보장**하고자
AI 코드 리뷰 라이브러리를 직접 구축하기로 했습니다.

## ⚠️ 기존 라이브러리의 한계

> 기존 라이브러리는 [여기](https://github.com/chanani/lib-claude-reviewer)서 확인하실 수 있습니다.

### Maven에 배포되어 타 포지션 개발자의 사용에 어려움

기존 라이브러리는 Java/Kotlin 환경에서 사용할 수 있도록 Maven에 배포되었습니다. 그러다 보니 프론트엔드 개발자분들이 사용하기에는 한계가 있었습니다.
모든 포지션에서 사용하기 위해 특정 스택에 종속되지 않는 방법이 필요했습니다.

### 반복되는 파이프라인 중복 작업

새로운 프로젝트에서 AI 코드 리뷰를 도입할 때마다 서버 내부에서 라이브러리 작동에 필요한 파일을 설치해야 하는 중복 작업이 발생했습니다.
비슷한 설정을 프로젝트마다 반복하는 과정에서 휴먼에러가 발생할 가능성이 존재했습니다.

## 🧑🏻‍💻 Claude Code Actions 도입하기

### Claude Code Actions을 선택한 이유

기존 라이브러리의 불편한 점을 찾아 개선해보기로 했습니다. 가장 큰 과제는
**'어떻게 인프라 관리 부담을 없애고, 모든 포지션의 개발자가 손쉽게 사용할 수 있게 하느냐'** 였습니다.

처음에는 라이브러리 자체를 고도화하여 다양한 언어 스택을 지원해 볼까 고민했습니다.
하지만 현재의 리소스로 모든 언어 환경에 완벽히 대응하는 라이브러리를 직접 구축하는 것은 오버헤드가 크다고 판단했습니다.

그러던 중 **Claude Code Actions**을 알게 되었습니다.
이미 검증된 프로세스를 활용하면 언어 제약 문제를 한 번에 해결할 수 있다고 생각했습니다.

사내 형상관리 툴은 GitHub를 사용하고 GitHub Actions는 기존 개발 흐름을 해치지 않습니다.
또한 레포지토리 단위로 손쉽게 적용할 수 있고 공통 워크 플로우를 통해 표준화한 사용 방식을 제공한다는 점에서 최적의 선택지였습니다.

### 1. 토큰 발급하기

Claude Code를 사용하신다면, `~/.claude/.credentials.json` 해당 경로 또는
`security find-generic-password -s "Claude Code-credentials" -w` 명령어를 통해
토큰을 확인할 수 있습니다.

토큰을 소유하고 있지 않을 경우 아래 명령어를 통해 토큰발급이 가능합니다.

```
claude setup-token
```

### 2. Claude App

[GitHub Apps](https://github.com/apps/claude) 다운로드 → Repository access 설정

### 3. GitHub Secrets 등록

소유하고 있는 토큰은 Repository의 Secrets에 등록해줍니다.
저는 `CLAUDE_CODE_OAUTH_TOKEN`으로 Key를 설정했습니다.

### 4. yml 파일 추가

GitHub Actions를 실행하려면 아래 경로에 yml 파일을 작성해야 합니다.  
PR 등록 시 코드 리뷰를 해주는 설정 파일과, 댓글에 `@claude`를 입력하면 PR을 등록해주는 설정 파일을 각각 설정했습니다.

> 상세 내부 설정은 제외한 기본 설정만 공유드리는 점 양해 부탁드립니다.

<details>
<summary><strong>📂 claude-code-review.yml</strong></summary>

```yaml
name: Claude Code Review

on:
  pull_request:
    types: [ opened, synchronize, ready_for_review, reopened ]
jobs:
  claude-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code Review
        id: claude-review
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: |
            PR #${{ github.event.pull_request.number }}을 코드 리뷰해줘.
          claude_args: "--allowedTools Bash(gh pr:*),Bash(gh api:*)"
```

</details>

<details> <summary><strong>📂 claude.yml</strong></summary>

```yaml
name: Claude Code

on:
  issue_comment:
    types: [ created ]
  pull_request_review_comment:
    types: [ created ]
  issues:
    types: [ opened, assigned ]
  pull_request_review:
    types: [ submitted ]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 1

      - name: Run Claude Code
        id: claude
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          additional_permissions: |
            actions: read
```

</details>

### claude code로 한번에 설정하기

지금까지 진행한 내용을 claude code에서 한번에 설치할 수 있습니다.
프로젝트 내부로 진입한 후 `/install-github-app`를 입력해주세요. `gh`도 같이 설치해주셔야합니다.

> 기존 Claude에서 별도의 API Key 발급했어야 했는데, Max/Pro 플랜에 한아여 API 비용이 발생하지 않습니다.
> 플랜의 한도 내에서 무료로 사용 가능합니다.

## 💬 동료 개발자 피드백

지금까지의 프로세스를 도입하고 불편한 부분을 개선하고자 사내 동료 개발자들의 피드백을 받았습니다.
대부분 긍정적인 반응이었지만 생각했던 것처럼 불편한 부분이 존재했습니다.  
문제는 **설정파일을 서버마다 매번 추가**해줘야 한다는 번거로움이 존재한다는 것이었습니다.

<div align="center">
  <img src="../../../assets/images/dev/회고/code-review/code-review-comment.png" alt="nextstep" width="500">
</div>
<div align="center">
  <img src="../../../assets/images/dev/회고/code-review/code-review-comment1.png" alt="nextstep" width="500">
</div>
<div align="center">
  <img src="../../../assets/images/dev/회고/code-review/code-review-comment2.png" alt="nextstep" width="500">
</div>
<div align="center">
  <img src="../../../assets/images/dev/회고/code-review/code-review-comment3.png" alt="nextstep" width="500">
</div>

## 🔧 불편함 개선하기

피드백의 핵심은 **도구는 좋은데, 설정의 번거로움과 비용이 발생한다**는 것입니다.
각 레포지토리마다 수십 줄의 `.yml` 파일을 복사해 넣는 방식은 버전 관리도 어렵고 휴먼 에러의 발생 가능성이 존재합니다.
이를 해결하기 위해 저는 GitHub의 Composite Action을 활용해 로직을 추상화하고 모듈화하기로 했습니다.

먼저 공유 전용 레포지토리를 생성해 기본적인 실행 스텝(.yml)을 정의했습니다.
이후 AI 코드 리뷰를 사용하는 레포지토리에서 공유 레포지토리를 주입받아 설정을 진행했습니다.(uses 항목에서 공유 전용 레포지토리 호출)

> **Composite Action이란 ?**  
> GitHub Compostie Action은 여러 개의 step을 하나의 Action으로 묶어 재사용할 수 있게 만드는 GitHub Actions의 기능입니다.  
> 쉽게 말하면 **공통 작업을 함수처럼 추출해 여러 곳에서 호출하는 것**입니다.

### 결과

이렇게 모듈화를 진행한 결과, 새로운 프로젝트에 AI 리뷰를 도입하는 과정이 **복사 붙여넣기**에서 **참조**로 바뀌었습니다.


## 참고 자료

- [무신사의 AI 코드 리뷰 프로세스 구축기](https://techblog.musinsa.com/%EB%AC%B4%EC%8B%A0%EC%82%AC%EC%9D%98-ai-%EC%BD%94%EB%93%9C-%EB%A6%AC%EB%B7%B0-%ED%94%84%EB%A1%9C%EC%84%B8%EC%8A%A4-%EA%B5%AC%EC%B6%95%EA%B8%B0-3ddb3c674e56)
- [claude-code-action GitHub Repository](https://github.com/anthropics/claude-code-action)
- [Claude Code Github Actions Document](https://code.claude.com/docs/en/github-actions)