# 예비대학원생 김씨

Jekyll과 Minimal Mistakes 테마를 기반으로 운영하는 개인 기술 블로그입니다.

- 블로그: <https://brian-223134.github.io/>
- 저장소: <https://github.com/brian-223134/brian-223134.github.io>
- 테마: Minimal Mistakes
- 스킨: `plum`
- 로케일: `ko-KR`
- 배포: GitHub Pages

## 프로젝트 구조

```text
.
├── _config.yml              # 사이트 전역 설정
├── _data/navigation.yml     # 상단 메뉴와 사이드바 카테고리
├── _pages/                  # About, 카테고리 아카이브 페이지
├── _posts/                  # 블로그 포스트
├── _layouts/                # Jekyll 레이아웃
├── _includes/               # 공통 UI 조각
├── _sass/                   # Minimal Mistakes 스타일 소스
└── assets/                  # CSS, JS, 이미지, 폰트, favicon
```

## 로컬 실행

Ruby/Jekyll 의존성을 설치한 뒤 로컬 서버를 실행합니다.

```bash
bundle install
bundle exec jekyll serve
```

브라우저에서 `http://127.0.0.1:4000`으로 확인합니다.

JS 번들 파일을 수정해야 할 때만 Node 의존성을 설치하고 빌드합니다.

```bash
npm install
npm run build:js
```

## 주요 설정

핵심 사이트 설정은 `_config.yml`에서 관리합니다.

- `title`: 상단 헤더와 SEO 제목에 사용되는 블로그 이름
- `description`: 검색 결과와 메타 태그에 사용되는 설명
- `url`: GitHub Pages URL, 현재 `https://brian-223134.github.io`
- `minimal_mistakes_skin`: 현재 `plum`
- `timezone`: 현재 `Asia/Seoul`
- `comments.provider`: 현재 `utterances`
- `analytics.provider`: 현재 `google-gtag`

카테고리 사이드바는 `_data/navigation.yml`에서 관리하고, 실제 카테고리 페이지는 `_pages/categories/`에 둡니다.

## 포스트 작성 규칙

포스트는 `_posts/YYYY-MM-DD-topic-number.md` 형식으로 작성합니다.

```yaml
---
title: "[카테고리] 제목"
excerpt: "한 줄 요약"

categories:
  - Company
tags:
  - [블리츠다이나믹스, 산학, 인턴, 인프라]

permalink: /intern/company/blitz-dynamics/01/

toc: true
toc_sticky: true

date: 2026-03-04
last_modified_at: 2026-03-04
---
```

작성 기준은 다음과 같습니다.

- 새 카테고리는 임의로 만들지 않고 `_data/navigation.yml`과 `_pages/categories/`를 함께 수정합니다.
- `categories` 값은 카테고리 페이지의 `taxonomy` 값과 일치해야 합니다.
- 기존 포스트의 `permalink`는 SEO 영향을 고려해 불필요하게 변경하지 않습니다.
- `date`는 게시일, `last_modified_at`은 마지막 수정일로 관리합니다.
- 코드블록에는 가능한 한 언어명을 명시합니다.

## 현재 카테고리

주요 카테고리와 URL은 다음과 같습니다.

| 분류 | URL | taxonomy |
| --- | --- | --- |
| 논문/NLP | `/categories/paper/nlp/` | `NLP` |
| 강의/AWS 기초 | `/categories/lecture/aws_basic/` | `AWS_BASIC` |
| 강의/CS224N | `/categories/lecture/cs224n/` | `CS224N` |
| 서적/DDIA | `/categories/book/designing-data-intensive-applications/` | `Designing-Data-Intensive-Applications` |
| 서적/데이터 엔지니어링 | `/categories/book/fundamentals-of-data-engineering/` | `Fundamentals-Of-Data-Engineering` |
| 면접 | `/categories/interview/` | `Interview` |
| 장학생/KT | `/categories/scholarship/kt/` | `KT-Scholarship` |
| 인턴/연구실 | `/categories/intern/lab/` | `Laboratory` |
| 인턴/회사 | `/categories/intern/company/` | `Company` |

## 운영 체크리스트

현재 저장소 기준으로 확인이 필요한 항목입니다.

- `_includes/comments-providers/utterances.html`의 `repo` 값이 placeholder인지 확인하고 실제 댓글 저장소로 교체합니다.
- `_config.yml`의 `analytics.google.tracking_id`를 실제 Google Analytics ID로 교체하거나, 사용하지 않으면 analytics 설정을 비활성화합니다.
- Google Search Console 인증 코드를 `_config.yml`의 `google_site_verification`에 추가합니다.
- Search Console에 `https://brian-223134.github.io/sitemap.xml`을 제출합니다.
- `_data/navigation.yml`의 `ANDREW_NG_DEEP_LEARNING` 카테고리는 실제 페이지가 없으므로 페이지를 추가하거나 메뉴에서 제거합니다.
- `/intern/paper/nlp/01` 형태의 permalink가 카테고리 구조와 맞는지 점검합니다.

## 참고

- favicon 파일은 `assets/images/favicon/`에 있습니다.
- 프로필 이미지는 `_config.yml`의 `author.avatar`에서 지정합니다.
- 테마 색상은 `_sass/minimal-mistakes/skins/_plum.scss`에서 관리합니다.
- 단순 CSS 오버라이드는 가능하면 `assets/css/main.scss`에 추가합니다.

## 차후 업데이트 사항

- Agent가 디렉토리를 스캔할 수 있도록 agent skills 추가
- Muli-directory 환경에서 활동할 수 있도록 네이밍 컨벤션 로직 전반적으로 수정 

