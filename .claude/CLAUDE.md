# CLAUDE.md — brian-223134.github.io
이 파일은 Claude Code가 프로젝트 작업 시 참고하는 지침 파일입니다.

## 프로젝트 개요
- Jekyll 블로그 (Minimal Mistakes 테마, plum 스킨)
- 블로그명: 예비대학원생 김씨
- URL: https://brian-223134.github.io
- 배포: push 시 GitHub Actions가 자동 빌드·배포 (로컬 빌드 불필요)

## 카테고리 & Permalink 파악 방법
새 포스트 작성 전, 반드시 아래 두 파일을 먼저 읽어 기존 카테고리와 permalink 패턴을 파악할 것:
- `_pages/categories/`    — 카테고리별 페이지 목록
- `_data/navigation.yml`  — 사이드바 구조 및 permalink prefix 확인

새 카테고리가 필요한 경우, 임의로 생성하지 말고 사용자에게 먼저 확인할 것.
새 카테고리 추가 시 `_data/navigation.yml`과 `_pages/categories/`도 함께 수정.

## 파일 명명 규칙
- 형식: `YYYY-MM-DD-카테고리소문자-번호.md`
- 예시: `2026-03-15-nlp-01.md`
- 위치: `_posts/`

## Front Matter 템플릿
```yaml
---
title: "[카테고리] 제목"
excerpt: "한 줄 설명"
categories:
  - CategoryName
tags:
  - tag1
  - tag2
permalink: /경로/번호
toc: true
toc_sticky: true
author_profile: true
date: YYYY-MM-DD
last_modified_at: YYYY-MM-DD
---
```

## Front Matter 템플릿 작성 예시 (2026-03-17-blitz-dynamics-12.md를 참고)
```yaml
---
title: "[인턴] Datadog으로 서비스 상태 모니터링하고 분석하기"
excerpt: "Datadog의 Metrics, Logs, APM, Dashboard, Monitor를 실제 업무에서 어떻게 활용했는지 정리한 기록"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라, Datadog, 모니터링]

permalink: /intern/company/blitz-dynamics/12/

toc: true
toc_sticky: true

date: 2026-03-17
last_modified_at: 2026-03-17
---
```

## 포스트 작성 스타일
- 언어: 한국어 (코드·고유명사는 영어 그대로)
- 논문 리뷰 구성: 배경 → 방법론 → 실험 결과 → 한계 → 요약
- 수식: MathJax — 블록 `$$...$$`, 인라인 `$...$`
- 이미지 경로: `/assets/images/posts/카테고리/파일명.png` (다만 일부는 하드 코딩된 경우가 있음.)
- 코드블록 언어 명시 필수 (```python, ```bash 등)

## 주요 파일 위치
- 사이트 설정:      `_config.yml`
- 사이드바 네비:    `_data/navigation.yml`
- 카테고리 페이지:  `_pages/categories/`
- 테마 색상:        `_sass/minimal-mistakes/skins/_plum.scss`
- 포스트:           `_posts/`

## 주의사항
- `_config.yml`의 `url`, `baseurl` 수정 금지
- `_sass/` 파일 수정 시 먼저 사용자에게 확인 요청
- 기존 포스트의 permalink 변경 금지 (SEO)
- 포스트 번호는 카테고리별로 독립적으로 증가


## 사용 가능한 스킬
- 블로그 포스트 요약: `.claude/skills/blog-post-summarizer/SKILL.md` 참조
- frontmatter validator: `.claude/skills/frontmatter-validator/SKILL.md` 참조
- 블로그 디자인 리뷰: `.claude/skills/blog-design-reviewer/SKILL.md` 참조