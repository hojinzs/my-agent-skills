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
