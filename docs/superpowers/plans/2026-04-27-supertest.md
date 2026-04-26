# supertest Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** QA 전문가 수준의 테스트 계획 수립, 자동 실행, HTML 보고서 3종 생성을 수행하는 Claude Code 스킬 `supertest`를 구현한다.

**Architecture:** 6개의 마크다운 파일로 구성된 스킬 문서 세트. SKILL.md가 Phase 0~4 플로우를 제어하고, 각 단계에서 필요한 보조 파일(init.md, tc-strategy.md, html-templates.md, pr-comment.md)을 참조한다. 모노리포 감지, 컨텍스트 적응형 테스트 전략, HTML 보고서 생성이 핵심이다.

**Tech Stack:** Markdown (SKILL.md), Bash (테스트 실행 명령), HTML/CSS (보고서 템플릿), gh CLI (PR 코멘트), curl (API 테스트)

---

## File Map

| 파일 | 역할 | 로드 시점 |
|------|------|-----------|
| `supertest/.claude-plugin/plugin.json` | 플러그인 메타데이터 | 설치 시 |
| `supertest/skills/supertest/SKILL.md` | Phase 0~4 플로우 제어 | 항상 |
| `supertest/skills/supertest/init.md` | init 인터뷰 + SUPERTEST.md 생성 | Phase 0, 파일 없을 때 |
| `supertest/skills/supertest/tc-strategy.md` | TC 작성 전략 (GREEN/EDGE, 분류별) | Phase 2 |
| `supertest/skills/supertest/html-templates.md` | HTML 보고서 3종 템플릿 | Phase 2, 4 |
| `supertest/skills/supertest/pr-comment.md` | PR 코멘트 생성 지침 + 기본 템플릿 | Phase 4, PR 있을 때 |
| `.claude-plugin/marketplace.json` | 마켓플레이스에 supertest 등록 | 설치 시 |

---

## Task 1: 플러그인 뼈대 생성

**Files:**
- Create: `supertest/.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: supertest 디렉토리 구조 생성**

```bash
mkdir -p supertest/.claude-plugin
mkdir -p supertest/skills/supertest
```

- [ ] **Step 2: plugin.json 작성**

`supertest/.claude-plugin/plugin.json` 내용:

```json
{
  "name": "supertest",
  "description": "QA expert-style test planning and execution with HTML reports.",
  "version": "1.0.0",
  "author": {
    "name": "hojinzs"
  }
}
```

- [ ] **Step 3: marketplace.json에 supertest 등록**

`/home/hojinzs/projects/my-agent-skills/.claude-plugin/marketplace.json` 수정:

```json
{
  "name": "my-agent-skills",
  "owner": {
    "name": "hojinzs"
  },
  "plugins": [
    {
      "name": "coolify-cli",
      "source": "./coolify-cli",
      "description": "Deploy applications, check deployment status, and manage settings via the Coolify CLI."
    },
    {
      "name": "supertest",
      "source": "./supertest",
      "description": "QA expert-style test planning and execution with HTML reports."
    }
  ]
}
```

- [ ] **Step 4: 구조 확인**

```bash
find supertest -type f
```

Expected output:
```
supertest/.claude-plugin/plugin.json
```

- [ ] **Step 5: Commit**

```bash
git add supertest/.claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "feat: scaffold supertest plugin structure"
```

---

## Task 2: SKILL.md — 메인 플로우 제어 문서

**Files:**
- Create: `supertest/skills/supertest/SKILL.md`

이 파일은 스킬이 항상 로드하는 진입점이다. Phase 0~4 분기 로직을 명확하게 기술한다.

- [ ] **Step 1: SKILL.md 작성**

`supertest/skills/supertest/SKILL.md` 내용:

```markdown
---
name: supertest
description: |
  Use when the user wants to run tests, verify features before or after a PR,
  perform QA testing, generate test reports, write test cases, or check if
  changes are safe to deploy. Also trigger on: "테스트 해줘", "QA 돌려줘",
  "PR 테스트", "test this feature", "run tests", "check for regressions".
---

# SuperTest Skill

QA 전문가처럼 테스트 계획을 수립하고, 자동으로 테스트를 실행하며, HTML 보고서를 생성한다.
코드 리뷰는 하지 않는다. 테스트 실행과 결과 보고에만 집중한다.

## Phase 0: SUPERTEST.md 확인

**먼저 실행:**
```bash
ls SUPERTEST.md 2>/dev/null && echo "EXISTS" || echo "MISSING"
```

```
SUPERTEST.md 존재?
├─ EXISTS → 파일 내용 읽기 → Phase 1로 진행
├─ MISSING + 대화형 → init.md 지침에 따라 인터뷰 진행 → SUPERTEST.md 생성 → Phase 1
└─ MISSING + 비대화형 → 컨텍스트 자동 감지만으로 Phase 1 진행
```

**비대화형 환경 판단:**
```bash
# CI 환경변수 확인
echo "${CI}${GITHUB_ACTIONS}${GITLAB_CI}"
```
CI=true, GITHUB_ACTIONS, GITLAB_CI 중 하나라도 설정되어 있으면 비대화형으로 처리한다.

**비대화형 자동 감지 순서:**
```bash
# 1. JS/TS 테스트 프레임워크
cat package.json 2>/dev/null | grep -E '"test":|jest|vitest|mocha'

# 2. Python
ls pytest.ini pyproject.toml setup.cfg 2>/dev/null

# 3. E2E
ls playwright.config.* cypress.config.* 2>/dev/null

# 4. API URL
grep -r "BASE_URL\|API_URL\|localhost" .env .env.local 2>/dev/null | head -5

# 5. Git 플랫폼
git remote -v | grep -E "github|gitlab"
```

## Phase 1: 컨텍스트 분석

**PR 감지:**
```bash
# GitHub
gh pr view --json number,files 2>/dev/null

# 현재 브랜치 기반
git log --oneline -1
git diff --name-only HEAD~1 HEAD 2>/dev/null || git diff --name-only origin/main...HEAD 2>/dev/null
```

PR이 있으면 변경 파일을 분류한다. PR이 없으면 사용자에게 테스트할 기능을 질문한다
(비대화형이면 전체 프로젝트 스캔).

**변경 분류 규칙:**

| 분류 | 감지 경로/패턴 | 실행할 테스트 |
|------|----------------|--------------|
| API 변경 | `routes/`, `controllers/`, `*.api.*`, `handler` | 단위 + API 호출 |
| UI 변경 | `components/`, `pages/`, `*.tsx`, `*.vue`, `*.svelte` | 단위 + E2E |
| 로직 변경 | `services/`, `utils/`, `lib/`, `helpers/` | 단위 + 통합 |
| 설정 변경 | `*.config.*`, `.env*`, `Dockerfile` | 연기 (환경 의존) |
| 복합 변경 | 위 2가지 이상 | 전체 조합 |

모노리포인 경우, 변경 파일의 경로 prefix로 해당 워크스페이스를 매핑하여 워크스페이스별로 분류한다.

## Phase 2: 테스트 계획 수립

**REQUIRED:** tc-strategy.md 지침을 읽고 TC를 작성한다.

- SUPERTEST.md의 `level` 기준으로 TC 수 조절:
  - `minimal`: 주요 GREEN CASE만 (기능당 1~2개)
  - `standard`: GREEN + 주요 EDGE CASE (기능당 3~5개, 기본값)
  - `thorough`: GREEN + 전체 EDGE CASE + 경계값 (기능당 5~10개)
- TC 작성 완료 후 **REQUIRED:** html-templates.md를 읽고 `plan-report.html` 생성

보고서 저장 경로:
```bash
REPORT_DIR=".supertest/$(date +%Y-%m-%d_%H-%M)"
mkdir -p "$REPORT_DIR"
# plan-report.html → $REPORT_DIR/plan-report.html
```

## Phase 3: 테스트 실행

실행 순서: 단위/통합 → API → E2E

**단위/통합:**
```bash
# SUPERTEST.md에 unit_command 있으면 사용, 없으면 자동 감지
# 예: npm test, pytest, go test ./...
```

**API 테스트 (auth_method: bearer 예시):**
```bash
TOKEN=$(printenv "$AUTH_TOKEN_ENV")
curl -s -w "\n%{http_code}" \
  -H "Authorization: Bearer $TOKEN" \
  "$API_BASE_URL/endpoint"
```

**E2E:**
```bash
# playwright.config.* 감지 시
npx playwright test --reporter=json 2>&1

# cypress.config.* 감지 시
npx cypress run --reporter json 2>&1
```

각 TC 실행 결과를 기록:
- PASS: 예상 결과 일치
- FAIL: 에러 메시지 캡처 + 원인 추정
- SKIP: 실행 환경 부족 (인증 정보 없음, E2E 설정 없음 등)

## Phase 4: 보고서 생성

**REQUIRED:** html-templates.md를 읽고 보고서를 생성한다.

```bash
# result-report.html
# executive-summary.html
# 저장: $REPORT_DIR/
```

**PR 코멘트 (조건: PR 있음 + SUPERTEST.md pr_comment.enabled: true):**

**REQUIRED:** pr-comment.md를 읽고 코멘트를 게시한다.

```bash
# GitHub
gh pr comment <PR번호> --body "$(cat pr-comment-body.md)"

# GitLab: curl로 MR Note API 호출
```

완료 후 사용자에게 보고서 경로를 알린다:
```
테스트 완료.
- 계획서: .supertest/2026-04-27_14-30/plan-report.html
- 결과:   .supertest/2026-04-27_14-30/result-report.html
- 요약:   .supertest/2026-04-27_14-30/executive-summary.html
```
```

- [ ] **Step 2: 파일 확인**

```bash
wc -l supertest/skills/supertest/SKILL.md
cat supertest/skills/supertest/SKILL.md | head -5
```

Expected: 파일이 존재하고 frontmatter가 `---`로 시작함

- [ ] **Step 3: Commit**

```bash
git add supertest/skills/supertest/SKILL.md
git commit -m "feat: add supertest SKILL.md with phase 0-4 flow"
```

---

## Task 3: init.md — 인터뷰 및 SUPERTEST.md 생성

**Files:**
- Create: `supertest/skills/supertest/init.md`

- [ ] **Step 1: init.md 작성**

`supertest/skills/supertest/init.md` 내용:

```markdown
# SuperTest Init Guide

SUPERTEST.md가 없을 때 이 지침을 따라 인터뷰를 진행하고 파일을 생성한다.

## Step 0: 모노리포 / 멀티 언어 감지

먼저 아래 명령을 실행한다:

```bash
# 모노리포 마커 확인
ls pnpm-workspace.yaml nx.json turbo.json lerna.json 2>/dev/null

# 멀티 package.json 확인
find . -name "package.json" -not -path "*/node_modules/*" -not -path "*/.git/*" | head -20

# 멀티 언어 확인
find . -name "pyproject.toml" -o -name "go.mod" -o -name "Cargo.toml" -o -name "build.gradle" \
  | grep -v node_modules | grep -v .git | head -10
```

**단일 프로젝트:** 모노리포 마커 없고, package.json이 루트 1개만 있고, 단일 언어 → 기본 인터뷰 진행

**모노리포 / 멀티 언어 감지 시:**

각 워크스페이스/디렉토리를 분석한다:
```bash
# 각 디렉토리별 테스트 프레임워크 감지
for dir in $(find . -name "package.json" -not -path "*/node_modules/*" -not -path "*/.git/*" \
  | xargs -I{} dirname {}); do
  echo "=== $dir ==="
  cat "$dir/package.json" 2>/dev/null | grep -E '"test":|jest|vitest|playwright|cypress' | head -5
done

# Python 워크스페이스
find . -name "pytest.ini" -o -name "pyproject.toml" | grep -v node_modules | grep -v .git
```

감지 결과를 바탕으로 **확인 테이블**을 사용자에게 제시한다:

```
감지된 프로젝트 구조를 확인해주세요:

| # | 경로              | 타입        | 테스트 프레임워크 | E2E        | API 테스트 | 실행 명령     |
|---|-------------------|-------------|-------------------|------------|------------|---------------|
| 1 | packages/frontend | frontend    | Vitest            | Playwright | -          | pnpm test     |
| 2 | packages/backend  | api-only    | Jest              | -          | curl       | pnpm test     |
| 3 | services/auth     | api-only    | Pytest            | -          | curl       | pytest        |

수정할 항목이 있으면 번호와 변경 내용을 알려주세요. 없으면 '확인'을 입력하세요.
```

사용자가 수정 요청 시 테이블을 업데이트하고 재확인한다.

## Step 1~10: 인터뷰 질문 (단일 프로젝트 또는 워크스페이스 공통 항목)

모노리포인 경우, 각 워크스페이스에 대해 4~6번 항목을 개별로 질문한다.

1. **프로젝트 타입** (api-only / frontend-only / fullstack / mobile)
2. **테스트 프레임워크 및 실행 명령** (예: `npm test`, `pytest`, `go test ./...`)
3. **E2E 도구** (Playwright / Cypress / 없음)
4. **API 베이스 URL** (예: `http://localhost:3000`)
5. **인증 방식** (bearer / basic / cookie / none) → bearer/basic이면 환경변수명 추가 질문
6. **브라우저** (chromium / firefox / webkit) + headless 여부
7. **테스트 깊이** (minimal / standard / thorough — 기본: standard)
8. **PR 코멘트 자동 게시** (yes/no) → yes면 플랫폼(github/gitlab) 추가 질문
9. **PR 코멘트 상세도** (summary / detailed / minimal — 기본: summary)
10. **제외할 경로** (기본: node_modules, dist, .next, coverage)

## SUPERTEST.md 생성

인터뷰 완료 후 아래 형식으로 프로젝트 루트에 SUPERTEST.md를 생성한다.

**단일 프로젝트:**
```markdown
# SUPERTEST Configuration

## Project
- name: <프로젝트명>
- type: <타입>

## Test Scope
- unit: true
- integration: true
- e2e: <true|false>
- api: <true|false>

## Test Commands
- unit: <명령>
- e2e: <명령 또는 생략>

## API Testing
- base_url: <URL>
- auth_method: <방식>
- auth_token_env: <환경변수명>

## Browser Testing
- browser: <브라우저>
- headless: true
- base_url: <URL>

## Reports
- output_dir: .supertest

## PR Comment
- enabled: <true|false>
- post_on: <github|gitlab|none>
- format: <summary|detailed|minimal>
- include_tc_table: true
- include_failed_logs: true
- include_coverage: false

## PR Comment Template
- header: "## 🧪 SuperTest Report"
- pass_icon: "✅"
- fail_icon: "❌"
- skip_icon: "⏭️"
- footer: "Generated by supertest skill"

## Test Depth
- level: <minimal|standard|thorough>

## Exclusions
- paths:
    - node_modules
    - dist
    - .next
    - coverage
```

**모노리포 추가 섹션:**
```markdown
## Workspaces
- monorepo: true

### <workspace-path>
- type: <타입>
- unit_command: <명령>
- e2e_command: <명령>
- api_base_url: <URL>
- auth_method: <방식>
- auth_token_env: <환경변수명>
```

생성 후 사용자에게 알린다:
```
SUPERTEST.md 생성 완료. 나중에 직접 수정하실 수 있습니다.
이제 테스트를 시작합니다.
```
```

- [ ] **Step 2: Commit**

```bash
git add supertest/skills/supertest/init.md
git commit -m "feat: add supertest init.md with monorepo detection and interview flow"
```

---

## Task 4: tc-strategy.md — TC 작성 전략

**Files:**
- Create: `supertest/skills/supertest/tc-strategy.md`

- [ ] **Step 1: tc-strategy.md 작성**

`supertest/skills/supertest/tc-strategy.md` 내용:

```markdown
# TC Strategy Guide

Phase 2에서 이 지침을 읽고 TC를 작성한다.

## TC 번호 체계

`TC-{NNN}` 형식. 001부터 순서대로. 모노리포는 워크스페이스 prefix 사용:
- `TC-FE-001` (frontend)
- `TC-BE-001` (backend)

## TC 구조

```
TC-001 | 기능명
  type:     GREEN | EDGE
  input:    { 구체적인 입력값, 엔드포인트, 페이로드, 화면 액션 }
  expected: { HTTP 상태코드, 응답 필드, 화면 요소, 값 }
  method:   unit | api | e2e
  status:   PENDING
  error:    (실행 후 FAIL 시 채움)
```

## GREEN CASE 작성 기준

정상 입력으로 예상 결과가 나오는 케이스. 기능의 핵심 동작을 검증한다.

예시 (API):
```
TC-001 | POST /api/users — 사용자 생성 성공
  type:     GREEN
  input:    POST /api/users { name: "홍길동", email: "test@example.com" }
  expected: HTTP 201, { id: <number>, name: "홍길동", email: "test@example.com" }
  method:   api
  status:   PENDING
```

예시 (E2E):
```
TC-002 | 로그인 페이지 — 정상 로그인
  type:     GREEN
  input:    email="user@test.com", password="correct-password" 입력 후 로그인 버튼 클릭
  expected: /dashboard로 리다이렉트, 사용자 이름 헤더에 표시
  method:   e2e
  status:   PENDING
```

## EDGE CASE 작성 기준

경계값, 비정상 입력, 권한 오류, 빈 값, 최대값 등을 검증한다.

**공통 EDGE CASE 패턴:**

| 패턴 | 예시 input | 예시 expected |
|------|-----------|---------------|
| 필수 필드 누락 | `{ email: "test@test.com" }` (name 없음) | HTTP 400, 에러 메시지 포함 |
| 잘못된 형식 | `{ email: "not-an-email" }` | HTTP 400 |
| 중복 데이터 | 이미 존재하는 email로 생성 | HTTP 409 |
| 권한 없음 | 토큰 없이 보호된 엔드포인트 호출 | HTTP 401 |
| 존재하지 않는 리소스 | GET /api/users/99999 | HTTP 404 |
| 빈 문자열 | `{ name: "" }` | HTTP 400 |
| 최대 길이 초과 | name이 256자 | HTTP 400 |
| SQL/스크립트 인젝션 | `{ name: "<script>alert(1)</script>" }` | HTTP 400 또는 이스케이프 처리 |

**test_depth별 포함 범위:**

| level | GREEN | EDGE |
|-------|-------|------|
| minimal | 기능당 1~2개 | 없음 |
| standard | 기능당 2~3개 | 기능당 2~3개 (필수 누락, 권한 오류) |
| thorough | 기능당 3~5개 | 기능당 5~10개 (전체 경계값 포함) |

## 분류별 TC 패턴

### API 변경 시 TC 패턴
1. 엔드포인트별 GREEN CASE (정상 요청/응답)
2. 인증 없이 호출 (401/403 확인)
3. 필수 파라미터 누락 (400 확인)
4. 잘못된 ID/존재하지 않는 리소스 (404 확인)
5. 중복 생성 시도 (409 확인, 해당되는 경우)

### UI 변경 시 TC 패턴
1. 핵심 사용자 흐름 (E2E)
2. 폼 제출 정상 동작
3. 폼 유효성 검사 (빈 값, 잘못된 형식)
4. 버튼/링크 동작
5. 로딩 상태, 에러 메시지 표시

### 로직 변경 시 TC 패턴
1. 핵심 함수 단위 테스트
2. 경계값 (0, -1, 최대값, null, undefined)
3. 비동기 처리 (timeout, 에러 핸들링)
```

- [ ] **Step 2: Commit**

```bash
git add supertest/skills/supertest/tc-strategy.md
git commit -m "feat: add supertest tc-strategy.md with GREEN/EDGE case patterns"
```

---

## Task 5: html-templates.md — 보고서 HTML 템플릿

**Files:**
- Create: `supertest/skills/supertest/html-templates.md`

- [ ] **Step 1: html-templates.md 작성**

`supertest/skills/supertest/html-templates.md` 내용:

````markdown
# HTML Report Templates

Phase 2에서 plan-report.html을, Phase 4에서 result-report.html과 executive-summary.html을 생성할 때 이 템플릿을 사용한다.

보고서는 `Write` 도구로 직접 파일에 저장한다. 저장 경로: `.supertest/{YYYY-MM-DD_HH-MM}/`

## 공통 CSS 스타일

모든 보고서에 아래 `<style>` 블록을 포함한다:

```html
<style>
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
         margin: 0; padding: 24px; background: #f8f9fa; color: #212529; }
  .header { background: #1a1a2e; color: white; padding: 24px 32px;
            border-radius: 12px; margin-bottom: 24px; }
  .header h1 { margin: 0; font-size: 1.6rem; }
  .header .meta { opacity: 0.7; font-size: 0.9rem; margin-top: 8px; }
  .card { background: white; border-radius: 8px; padding: 20px 24px;
          margin-bottom: 16px; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
  .card h2 { margin: 0 0 16px; font-size: 1.1rem; color: #1a1a2e; }
  table { width: 100%; border-collapse: collapse; font-size: 0.9rem; }
  th { background: #f1f3f5; text-align: left; padding: 10px 12px;
       font-weight: 600; border-bottom: 2px solid #dee2e6; }
  td { padding: 10px 12px; border-bottom: 1px solid #f1f3f5; vertical-align: top; }
  tr:hover td { background: #f8f9fa; }
  .badge { display: inline-block; padding: 2px 8px; border-radius: 4px;
           font-size: 0.78rem; font-weight: 600; }
  .badge-green  { background: #d3f9d8; color: #2f9e44; }
  .badge-edge   { background: #fff3bf; color: #e67700; }
  .badge-pass   { background: #d3f9d8; color: #2f9e44; }
  .badge-fail   { background: #ffe3e3; color: #c92a2a; }
  .badge-skip   { background: #e9ecef; color: #495057; }
  .stat-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
               gap: 16px; margin-bottom: 24px; }
  .stat-box { background: white; border-radius: 8px; padding: 16px 20px;
              text-align: center; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
  .stat-box .num { font-size: 2rem; font-weight: 700; }
  .stat-box .label { font-size: 0.8rem; color: #868e96; margin-top: 4px; }
  .risk-red    { color: #c92a2a; font-weight: 700; }
  .risk-yellow { color: #e67700; font-weight: 700; }
  .risk-green  { color: #2f9e44; font-weight: 700; }
  details { margin-top: 8px; }
  summary { cursor: pointer; color: #1971c2; font-size: 0.85rem; }
  pre { background: #f1f3f5; padding: 12px; border-radius: 4px;
        font-size: 0.8rem; overflow-x: auto; white-space: pre-wrap; }
</style>
```

---

## Template 1: plan-report.html

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>SuperTest — 테스트 계획서</title>
<!-- 공통 CSS 삽입 -->
</head>
<body>

<div class="header">
  <h1>🧪 테스트 계획서</h1>
  <div class="meta">
    프로젝트: {PROJECT_NAME} &nbsp;|&nbsp;
    생성일시: {DATETIME} &nbsp;|&nbsp;
    테스트 깊이: {TEST_DEPTH}
  </div>
</div>

<!-- 테스트 대상 요약 -->
<div class="card">
  <h2>테스트 대상</h2>
  <table>
    <tr><th>분류</th><th>기능/변경 항목</th><th>테스트 방법</th></tr>
    <!-- 반복: 변경 분류별로 행 추가 -->
    <tr>
      <td><span class="badge badge-green">API 변경</span></td>
      <td>{기능명}</td>
      <td>단위 + API 호출</td>
    </tr>
  </table>
</div>

<!-- TC 목록 -->
<div class="card">
  <h2>테스트 케이스 목록 (총 {TC_TOTAL}개)</h2>
  <table>
    <tr><th>TC</th><th>구분</th><th>기능</th><th>입력</th><th>예상 결과</th><th>방법</th></tr>
    <!-- 반복: TC별 행 -->
    <tr>
      <td>TC-001</td>
      <td><span class="badge badge-green">GREEN</span></td>
      <td>{기능명}</td>
      <td>{input 요약}</td>
      <td>{expected 요약}</td>
      <td>api</td>
    </tr>
    <tr>
      <td>TC-002</td>
      <td><span class="badge badge-edge">EDGE</span></td>
      <td>{기능명}</td>
      <td>{input 요약}</td>
      <td>{expected 요약}</td>
      <td>api</td>
    </tr>
  </table>
</div>

<!-- 테스트 전략 -->
<div class="card">
  <h2>테스트 전략</h2>
  <table>
    <tr><th>항목</th><th>내용</th></tr>
    <tr><td>실행 순서</td><td>단위/통합 → API → E2E</td></tr>
    <tr><td>단위 테스트</td><td>{unit_command}</td></tr>
    <tr><td>API 테스트</td><td>curl, 인증: {auth_method}</td></tr>
    <tr><td>E2E 테스트</td><td>{e2e_command 또는 "해당 없음"}</td></tr>
    <tr><td>예상 소요시간</td><td>약 {예상 시간}분</td></tr>
  </table>
</div>

</body>
</html>
```

---

## Template 2: result-report.html

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<title>SuperTest — 테스트 결과</title>
<!-- 공통 CSS 삽입 -->
</head>
<body>

<div class="header">
  <h1>🧪 테스트 결과 보고서</h1>
  <div class="meta">
    프로젝트: {PROJECT_NAME} &nbsp;|&nbsp;
    실행일시: {DATETIME} &nbsp;|&nbsp;
    소요시간: {DURATION}
  </div>
</div>

<!-- 통계 -->
<div class="stat-grid">
  <div class="stat-box">
    <div class="num">{TC_TOTAL}</div>
    <div class="label">총 TC</div>
  </div>
  <div class="stat-box">
    <div class="num" style="color:#2f9e44">{PASS_COUNT}</div>
    <div class="label">✅ PASS</div>
  </div>
  <div class="stat-box">
    <div class="num" style="color:#c92a2a">{FAIL_COUNT}</div>
    <div class="label">❌ FAIL</div>
  </div>
  <div class="stat-box">
    <div class="num" style="color:#868e96">{SKIP_COUNT}</div>
    <div class="label">⏭️ SKIP</div>
  </div>
  <div class="stat-box">
    <div class="num">{PASS_RATE}%</div>
    <div class="label">통과율</div>
  </div>
</div>

<!-- TC 결과 테이블 -->
<div class="card">
  <h2>TC별 결과</h2>
  <table>
    <tr><th>TC</th><th>구분</th><th>기능</th><th>방법</th><th>결과</th><th>상세</th></tr>
    <!-- PASS 행 예시 -->
    <tr>
      <td>TC-001</td>
      <td><span class="badge badge-green">GREEN</span></td>
      <td>{기능명}</td>
      <td>api</td>
      <td><span class="badge badge-pass">PASS</span></td>
      <td></td>
    </tr>
    <!-- FAIL 행 예시 -->
    <tr>
      <td>TC-007</td>
      <td><span class="badge badge-edge">EDGE</span></td>
      <td>{기능명}</td>
      <td>api</td>
      <td><span class="badge badge-fail">FAIL</span></td>
      <td>
        <details>
          <summary>에러 상세 보기</summary>
          <strong>에러:</strong>
          <pre>{에러 메시지 원문}</pre>
          <strong>원인 추정:</strong>
          <p>{원인 추정 텍스트}</p>
        </details>
      </td>
    </tr>
    <!-- SKIP 행 예시 -->
    <tr>
      <td>TC-010</td>
      <td><span class="badge badge-edge">EDGE</span></td>
      <td>{기능명}</td>
      <td>e2e</td>
      <td><span class="badge badge-skip">SKIP</span></td>
      <td>E2E 설정 없음</td>
    </tr>
  </table>
</div>

</body>
</html>
```

---

## Template 3: executive-summary.html

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<title>SuperTest — 종합 요약</title>
<!-- 공통 CSS 삽입 -->
</head>
<body>

<div class="header">
  <h1>🧪 테스트 종합 요약</h1>
  <div class="meta">
    프로젝트: {PROJECT_NAME} &nbsp;|&nbsp; {DATETIME}
  </div>
</div>

<!-- 핵심 수치 -->
<div class="stat-grid">
  <div class="stat-box">
    <div class="num">{TC_TOTAL}</div><div class="label">총 TC</div>
  </div>
  <div class="stat-box">
    <div class="num" style="color:#2f9e44">{PASS_COUNT}</div><div class="label">✅ PASS</div>
  </div>
  <div class="stat-box">
    <div class="num" style="color:#c92a2a">{FAIL_COUNT}</div><div class="label">❌ FAIL</div>
  </div>
  <div class="stat-box">
    <div class="num" style="color:#868e96">{SKIP_COUNT}</div><div class="label">⏭️ SKIP</div>
  </div>
</div>

<!-- 결론 -->
<div class="card">
  <h2>배포 판정</h2>
  <!-- FAIL 0개일 때 -->
  <p style="font-size:1.4rem" class="risk-green">🟢 배포 가능</p>
  <!-- 핵심 기능 FAIL 있을 때 -->
  <!-- <p style="font-size:1.4rem" class="risk-red">🔴 배포 보류 권장</p> -->
  <!-- 엣지케이스만 FAIL일 때 -->
  <!-- <p style="font-size:1.4rem" class="risk-yellow">🟡 조건부 배포 가능</p> -->

  <table style="margin-top:16px">
    <tr><th>항목</th><th>내용</th></tr>
    <tr><td>위험도</td><td>{위험도 평가 텍스트}</td></tr>
    <tr><td>FAIL 항목</td><td>{핵심 기능 FAIL 목록 또는 "없음"}</td></tr>
    <tr><td>테스트 범위</td><td>{unit | api | e2e} — {테스트한 기능 목록}</td></tr>
    <tr><td>제외 항목</td><td>{SKIP된 TC 이유}</td></tr>
    <tr><td>소요시간</td><td>{DURATION}</td></tr>
  </table>
</div>

<!-- 위험도 평가 기준 -->
<!--
  위험도 결정 규칙:
  - GREEN CASE가 하나라도 FAIL → 🔴 배포 보류 권장
  - EDGE CASE만 FAIL → 🟡 조건부 배포 가능
  - 전체 PASS 또는 SKIP만 → 🟢 배포 가능
-->

</body>
</html>
```

## 보고서 생성 시 주의사항

- `{...}` 플레이스홀더를 실제 값으로 모두 치환한다. 플레이스홀더가 남아있으면 안 된다.
- TC 행은 실제 TC 수만큼 반복 생성한다.
- FAIL 행에는 반드시 에러 원문과 원인 추정을 채운다.
- 위험도는 FAIL TC의 type(GREEN/EDGE) 기준으로 결정한다.
- 공통 CSS를 `<style>` 태그로 `<head>` 안에 인라인 삽입한다.
````

- [ ] **Step 2: Commit**

```bash
git add supertest/skills/supertest/html-templates.md
git commit -m "feat: add supertest html-templates.md with 3 report templates"
```

---

## Task 6: pr-comment.md — PR 코멘트 생성

**Files:**
- Create: `supertest/skills/supertest/pr-comment.md`

- [ ] **Step 1: pr-comment.md 작성**

`supertest/skills/supertest/pr-comment.md` 내용:

```markdown
# PR Comment Guide

Phase 4에서 PR 컨텍스트가 있고 SUPERTEST.md의 `pr_comment.enabled: true`일 때 이 지침을 따른다.

## 플랫폼 감지

```bash
git remote -v | grep -o 'github\|gitlab' | head -1
```

## GitHub — gh CLI 사용

```bash
# PR 번호 확인
PR_NUM=$(gh pr view --json number -q .number 2>/dev/null)

# 코멘트 게시
gh pr comment "$PR_NUM" --body "$(cat <<'COMMENT'
{PR 코멘트 내용}
COMMENT
)"
```

## GitLab — curl MR Note API

```bash
# 필요 환경변수: GITLAB_TOKEN, GITLAB_PROJECT_ID, MR_IID
curl -s -X POST \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"body\": \"$(cat pr-comment-body.md | sed 's/"/\\"/g' | tr '\n' ' ')\"}" \
  "https://gitlab.com/api/v4/projects/$GITLAB_PROJECT_ID/merge_requests/$MR_IID/notes"
```

## PR 코멘트 형식

SUPERTEST.md의 `format` 값에 따라 선택한다.

### format: summary (기본)

```markdown
{header}

| 항목 | 결과 |
|------|------|
| 총 TC | {TC_TOTAL} |
| {pass_icon} PASS | {PASS_COUNT} |
| {fail_icon} FAIL | {FAIL_COUNT} |
| {skip_icon} SKIP | {SKIP_COUNT} |

**위험도:** {위험도 이모지} {배포 판정}

<details>
<summary>실패 TC 보기 ({FAIL_COUNT}건)</summary>

| TC | 기능 | 에러 | 원인 추정 |
|----|------|------|-----------|
{FAIL_TC_ROWS}

</details>

{footer}
```

### format: detailed

summary 형식에 아래를 추가:

```markdown
<details>
<summary>전체 TC 결과</summary>

| TC | 구분 | 기능 | 결과 |
|----|------|------|------|
{ALL_TC_ROWS}

</details>
```

### format: minimal

```markdown
{header}
{pass_icon} {PASS_COUNT} / {TC_TOTAL} — {위험도 이모지} {배포 판정}
{footer}
```

## 플레이스홀더 치환 규칙

- `{header}`: SUPERTEST.md의 `pr_comment_template.header`
- `{pass_icon}`, `{fail_icon}`, `{skip_icon}`: SUPERTEST.md의 아이콘 설정
- `{footer}`: SUPERTEST.md의 `pr_comment_template.footer`
- `{FAIL_TC_ROWS}`: FAIL인 TC만 마크다운 표 행으로 나열
- `{ALL_TC_ROWS}`: 전체 TC 마크다운 표 행
- `{위험도 이모지}`: 🔴 / 🟡 / 🟢
- `{배포 판정}`: "배포 보류 권장" / "조건부 배포 가능" / "배포 가능"

## include_failed_logs: true일 때

FAIL TC의 `<details>` 블록에 에러 로그 원문을 포함한다 (100줄 초과 시 앞 50줄 + 뒤 50줄).

## PR 코멘트 없는 경우

`pr_comment.enabled: false` 또는 `post_on: none`이면 이 파일의 지침은 무시한다.
```

- [ ] **Step 2: Commit**

```bash
git add supertest/skills/supertest/pr-comment.md
git commit -m "feat: add supertest pr-comment.md with GitHub/GitLab support"
```

---

## Task 7: 최종 검증 및 마켓플레이스 등록

**Files:**
- Verify: 모든 스킬 파일 존재 확인
- Verify: marketplace.json 등록 확인

- [ ] **Step 1: 파일 구조 최종 확인**

```bash
find supertest -type f | sort
```

Expected:
```
supertest/.claude-plugin/plugin.json
supertest/skills/supertest/SKILL.md
supertest/skills/supertest/html-templates.md
supertest/skills/supertest/init.md
supertest/skills/supertest/pr-comment.md
supertest/skills/supertest/tc-strategy.md
```

- [ ] **Step 2: frontmatter 확인**

```bash
head -6 supertest/skills/supertest/SKILL.md
```

Expected: `---`로 시작하고 `name: supertest`와 `description:` 포함

- [ ] **Step 3: marketplace.json 확인**

```bash
cat .claude-plugin/marketplace.json | grep -A3 "supertest"
```

Expected: supertest 플러그인 항목이 포함되어 있음

- [ ] **Step 4: 최종 커밋**

```bash
git add -A
git status  # 누락된 파일 없는지 확인
git commit -m "feat: complete supertest skill implementation"
```

- [ ] **Step 5: 사용자에게 설치 방법 안내**

```
supertest 스킬 구현 완료.

설치:
  /plugin (my-agent-skills 마켓플레이스에서)
  /reload-plugins

사용:
  /supertest:supertest 또는 "테스트 해줘"라고 말하면 자동 트리거

첫 실행 시 SUPERTEST.md가 없으면 init 인터뷰를 시작합니다.
```

---

## Self-Review

**스펙 커버리지 확인:**

| 스펙 요구사항 | 구현 태스크 |
|--------------|------------|
| Phase 0: SUPERTEST.md 확인 | Task 2 (SKILL.md) |
| 비대화형 환경 폴백 | Task 2 (SKILL.md Phase 0) |
| 모노리포 감지 + 확인 테이블 | Task 3 (init.md Step 0) |
| init 인터뷰 10개 질문 | Task 3 (init.md Step 1~10) |
| Phase 1: 컨텍스트 분석 + 변경 분류 | Task 2 (SKILL.md Phase 1) |
| Phase 2: TC 작성 + plan-report.html | Task 2, 4, 5 |
| GREEN / EDGE CASE 패턴 | Task 4 (tc-strategy.md) |
| Phase 3: 단위/API/E2E 실행 | Task 2 (SKILL.md Phase 3) |
| Phase 4: result + executive-summary HTML | Task 5 (html-templates.md) |
| PR 코멘트 (GitHub/GitLab) | Task 6 (pr-comment.md) |
| 보고서 경로: .supertest/{date}/ | Task 2 (SKILL.md Phase 2) |
| plugin.json + marketplace.json | Task 1 |
