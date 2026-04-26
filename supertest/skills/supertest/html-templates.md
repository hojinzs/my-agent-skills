# HTML Report Templates

Phase 2에서 plan-report.html을, Phase 4에서 result-report.html과 executive-summary.html을 생성할 때 이 템플릿을 사용한다.

보고서는 Write 도구로 직접 파일에 저장한다. 저장 경로: `.supertest/{YYYY-MM-DD_HH-MM}/`

## 공통 CSS 스타일

모든 보고서의 `<head>` 안에 아래 `<style>` 블록을 인라인으로 포함한다:

```css
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
```

---

## Template 1: plan-report.html

Replace all `{PLACEHOLDER}` values with actual data when generating.

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>SuperTest — 테스트 계획서</title>
<style>
/* 공통 CSS 스타일 삽입 */
</style>
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

<div class="card">
  <h2>테스트 대상</h2>
  <table>
    <tr><th>분류</th><th>기능/변경 항목</th><th>테스트 방법</th></tr>
    <!-- 변경 분류별로 행 추가 -->
    <tr>
      <td><span class="badge badge-green">API 변경</span></td>
      <td>{기능명}</td>
      <td>단위 + API 호출</td>
    </tr>
  </table>
</div>

<div class="card">
  <h2>테스트 케이스 목록 (총 {TC_TOTAL}개)</h2>
  <table>
    <tr><th>TC</th><th>구분</th><th>기능</th><th>입력</th><th>예상 결과</th><th>방법</th></tr>
    <!-- TC별 행 반복 -->
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

<div class="card">
  <h2>테스트 전략</h2>
  <table>
    <tr><th>항목</th><th>내용</th></tr>
    <tr><td>실행 순서</td><td>단위/통합 → API → E2E</td></tr>
    <tr><td>단위 테스트</td><td>{unit_command}</td></tr>
    <tr><td>API 테스트</td><td>curl, 인증: {auth_method}</td></tr>
    <tr><td>E2E 테스트</td><td>{e2e_command}</td></tr>
    <tr><td>예상 소요시간</td><td>약 {ESTIMATED_MINUTES}분</td></tr>
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
<style>
/* 공통 CSS 스타일 삽입 */
</style>
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

<div class="card">
  <h2>TC별 결과</h2>
  <table>
    <tr><th>TC</th><th>구분</th><th>기능</th><th>방법</th><th>결과</th><th>상세</th></tr>
    <!-- PASS 행 -->
    <tr>
      <td>TC-001</td>
      <td><span class="badge badge-green">GREEN</span></td>
      <td>{기능명}</td>
      <td>api</td>
      <td><span class="badge badge-pass">PASS</span></td>
      <td></td>
    </tr>
    <!-- FAIL 행 -->
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
    <!-- SKIP 행 -->
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

위험도 결정 규칙:
- GREEN CASE가 하나라도 FAIL → 🔴 배포 보류 권장
- EDGE CASE만 FAIL → 🟡 조건부 배포 가능
- 전체 PASS 또는 SKIP만 → 🟢 배포 가능

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<title>SuperTest — 종합 요약</title>
<style>
/* 공통 CSS 스타일 삽입 */
</style>
</head>
<body>

<div class="header">
  <h1>🧪 테스트 종합 요약</h1>
  <div class="meta">
    프로젝트: {PROJECT_NAME} &nbsp;|&nbsp; {DATETIME}
  </div>
</div>

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

<div class="card">
  <h2>배포 판정</h2>
  <!-- 아래 3가지 중 실제 결과에 맞는 것 하나만 남기고 나머지는 제거한다 -->
  <p style="font-size:1.4rem" class="risk-green">🟢 배포 가능</p>
  <!-- <p style="font-size:1.4rem" class="risk-red">🔴 배포 보류 권장</p> -->
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

</body>
</html>
```

---

## 보고서 생성 시 주의사항

- `{...}` 플레이스홀더를 실제 값으로 모두 치환한다. 플레이스홀더가 남아있으면 안 된다.
- TC 행은 실제 TC 수만큼 반복 생성한다 (예시 행 3개를 실제 TC 수로 교체).
- FAIL 행에는 반드시 에러 원문과 원인 추정을 채운다.
- `<style>` 주석을 실제 CSS 내용으로 교체한다 (공통 CSS 스타일 섹션 참조).
- executive-summary의 배포 판정은 실제 결과에 맞는 항목 하나만 남긴다.
