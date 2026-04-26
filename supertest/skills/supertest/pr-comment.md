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
  -d "{\"body\": \"$(cat pr-comment-body.md | sed 's/\"/\\\"/g' | tr '\n' ' ')\"}" \
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
