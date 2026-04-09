---
name: dylabs-techblog
description: "Use when writing, editing, or publishing blog posts on the DYLabs tech blog (dylabs.github.io), adding new authors, or managing blog content. Triggers on: 블로그 글 작성, 기술블로그, 포스팅, 블로그 포스트, techblog, blog post, 블로그 배포, 저자 추가, new author, 글 발행"
---

# DYLabs Tech Blog 포스팅 가이드

**레포**: `dylabs/dylabs.github.io` (private)
**URL**: https://dylabs.github.io
**스택**: Astro 6 + Tailwind CSS v4 + Shiki
**로컬 경로**: `/Users/hyeonwoo/DEV/dylabs.github.io`

## 글 작성

### 1. 파일 생성

`src/content/posts/<slug>.md` 에 Markdown 파일 생성. slug는 URL이 된다 (`/posts/<slug>/`).

```markdown
---
title: "글 제목"
description: "글 요약 (목록 + OG 태그에 사용)"
author: "hyeonwoo"
date: 2026-04-10
tags: ["kubernetes", "devops"]
thumbnail: "/images/posts/example.png"
draft: false
---

## 본문 시작

Markdown으로 작성. h2, h3가 자동으로 TOC에 표시됨.
```

### 2. Frontmatter 필드

| 필드 | 필수 | 설명 |
|------|------|------|
| `title` | O | 글 제목 |
| `description` | O | 글 요약 (1~2문장) |
| `author` | O | `src/data/authors.json`의 `id`와 매칭 |
| `date` | O | 게시일 (YYYY-MM-DD) |
| `tags` | O | 태그 배열 (1개 이상) |
| `thumbnail` | X | 썸네일 이미지 경로 (없으면 그라데이션 표시) |
| `draft` | X | `true`면 빌드에서 제외 (기본값 `false`) |

### 3. 코드 블록

Shiki 자동 하이라이팅. 언어 명시하면 됨:

````markdown
```typescript
const hello = "world";
```
````

라이트/다크 모드에 따라 테마 자동 전환 (`github-light` / `github-dark`).

### 4. 이미지

`public/images/posts/` 에 이미지 저장, Markdown에서 참조:

```markdown
![설명](/images/posts/my-image.png)
```

## 저자 추가

`src/data/authors.json` 배열에 새 항목 추가:

```json
{
  "id": "github-username",
  "name": "표시 이름",
  "role": "직무 (예: Frontend Engineer)",
  "github": "github-username",
  "bio": "한 줄 소개"
}
```

아바타는 `https://github.com/<github>.png` 에서 자동 로드.

## 로컬 미리보기

```bash
cd /Users/hyeonwoo/DEV/dylabs.github.io
pnpm dev
# http://localhost:4321 에서 확인
```

## 배포 플로우

1. `src/content/posts/<slug>.md` 작성
2. PR 생성 → 리뷰
3. `main` 머지 → GitHub Actions 자동 빌드/배포
4. https://dylabs.github.io 에 반영

**draft 글**: `draft: true`로 설정하면 빌드에서 제외. 리뷰 준비되면 `false`로 변경.

## 기능 요약

- **검색**: Cmd+K (빌드 시 생성된 JSON 인덱스 기반 클라이언트 검색)
- **태그**: `/tags/` 페이지, 홈 상단 태그 필터 바
- **저자**: `/authors/` 페이지, 저자별 글 목록
- **TOC**: h2/h3 자동 추출, 상단 + 스크롤 시 우측 사이드바
- **다크모드**: 라이트(기본) / 다크 전환, localStorage 저장
- **RSS**: `/rss.xml`
- **SEO**: OG/Twitter 메타태그, sitemap.xml 자동 생성
- **읽기 시간**: 한국어 기준 분당 500자 자동 계산

## 주의사항

- `main` push 시 자동 배포됨 — PR 리뷰 후 머지
- `pnpm-lock.yaml` 커밋 필수 (CI에서 필요)
- Node.js 22+ 필요 (Astro 6 요구사항)
- `packageManager` 필드가 `package.json`에 있어야 CI pnpm 동작
