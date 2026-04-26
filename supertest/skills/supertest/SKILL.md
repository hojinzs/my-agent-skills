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
