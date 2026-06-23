# 원티드 저장 기업(북마크) 수집 — 상세 절차

`screen-companies` 입력 경로 B에서 크롬이 연결돼 있을 때 사용한다. 사용자가 원티드(wanted.co.kr)에 **저장(북마크)** 한 채용공고의 회사 목록을 읽어와 스크리닝 대상으로 삼는다. 검증된 방식은 2026-05-22 실행에서 확립되었다.

## 핵심 원리

- 북마크 **목록**은 사용자의 로그인 세션에서만 보인다 → 크롬 연결(사용자가 원티드에 로그인된 상태)이 전제다.
- 회사별 **재무·인원** 데이터는 원티드가 노출하는 비로그인 데이터(KODATA 출처)로 충분히 얻을 수 있다 → 캐치를 거칠 필요 없다.

## 1. 크롬 연결 확인 (전제)

1. `ToolSearch`로 `mcp__claude-in-chrome__tabs_context_mcp`, `navigate`, `get_page_text`, `javascript_tool`, `tabs_create_mcp`를 한 번에 로드.
2. `tabs_context_mcp` 호출로 연결/탭 상황 파악.
3. **크롬 미연결이거나 원티드 로그인 세션이 아니면** → 자동 수집 불가. 사용자에게 알리고 입력 경로 A(회사명 직접 붙여넣기)로 폴백한다. 무한 재시도 금지.

## 2. 북마크 회사 목록 수집

1. 새 탭 생성 후 **북마크 목록 페이지로 바로 이동**한다: `https://www.wanted.co.kr/profile/bookmarks`
   - (`/bookmarks`는 404다. 반드시 `/profile/bookmarks`.) 미로그인이면 로그인 화면이 뜨므로 그때만 경로 A 폴백.
2. **끝까지 스크롤해 무한로딩**시킨다(20건 단위 lazy-load). `window.scrollTo(0, document.documentElement.scrollHeight)`를 `docH`/카드수가 안 늘 때까지 반복.
   - ⚠️ 스크롤 루프를 너무 길게 한 번에 돌리면 `javascript_tool`이 CDP 타임아웃(45s)을 낼 수 있다. 타임아웃이 나도 스크롤은 진행됐을 수 있으니, **다음 호출에서 카드 수를 다시 확인**하면 된다(예: 80건 로드 완료).
   - 카드 텍스트에서 `jobId`(링크 `/wd/{jobId}`), 포지션명, 회사명을 추출한다. (`합격보상금…` 라인은 버린다.)
3. **companyId·마감일**은 공고 상세 API에서 한 번에 얻는다(같은 응답):
   ```
   https://www.wanted.co.kr/api/chaos/jobs/v4/{jobId}/details
     → data.job.company.id   (companyId)
     → data.job.due_time     (마감일, ISO 문자열 / null = 상시채용)
   ```
   회사명 기준으로 중복 제거(companyId 기준 맵). **마감일은 공고(jobId) 단위로 보존**한다(같은 회사도 공고마다 마감 다름).
   - 마감일은 `due_time` 그대로 저장하고, 오늘 날짜(currentDate)와 비교해 **남은 일수**를 계산한다. `null`이면 "상시채용".

## 3. 회사별 재무·인원 수집

> **중요(2026-06-22 갱신):** 재무 API는 companyId가 아니라 **regNoHash**(사업자번호 해시, 40자 hex)를 받는다. companyId로 호출하면 404. regNoHash는 회사 페이지 `__NEXT_DATA__`의 `companyInfo` 쿼리에만 있으므로, 회사 페이지 HTML이 필요하다.

권장 방식 — **페이지 이동 없이 `fetch`로 한 번에**(아래 ⚠️ 상태 유실 주의 참고):

각 companyId에 대해 `javascript_tool` 안에서:
1. 회사 페이지 HTML을 `fetch('https://www.wanted.co.kr/company/{companyId}', {credentials:'include'}).then(r=>r.text())`로 받아, `<script id="__NEXT_DATA__">…</script>` 안의 JSON을 파싱.
   - `props.pageProps.dehydrateState.queries` 배열에서:
     - `companyInfo` 쿼리 → `state.data.regNoHash` (재무 API 키), `name`
     - `companySummary` 쿼리 → `state.data.employee.total`(인원), `state.data.salary.salary`(평균연봉, 원 단위)
   - ⚠️ `companySummary` 쿼리 자체가 없는 회사가 있다(재무·인원 미노출). 이 경우 regNoHash도 없을 수 있음 → ⚠️확인불가.
2. **연도별 매출·영업이익**:
   ```
   https://www.wanted.co.kr/api/krs/v1/company/{regNoHash}/financial-report-for-wanted
   ```
   응답 `financialReport[]`의 각 원소 = `{year, salesAmount, operatingIncome, netIncome}`. 최근 3개년 `salesAmount`(추이)와 최근 `operatingIncome`(적자=마이너스) 사용.
3. 데이터가 없으면(미공개/신생, `companySummary`·`regNoHash` 부재) → ⚠️확인불가, 임의 추정 금지.

> ⚠️ **window 상태 유실 주의:** 다른 URL로 `navigate`하면 페이지 컨텍스트가 초기화돼 `window.__bm` 같은 수집 데이터가 날아간다. 북마크 목록 수집 → companyId 수집 → 회사별 재무 수집을 **모두 같은 탭의 같은 페이지(`/profile/bookmarks`)에서 `fetch`로** 수행하라. 회사 페이지로 일일이 navigate하지 말 것(75개면 navigate 150회 + 상태 유실).

## 4. 판정·출력

- 판정 기준은 SKILL.md 본문의 "거름(❌) 기준"과 동일하다(적자+매출 정체/감소, 또는 인원 5명 이하).
- 출력 표는 본문의 표 형식을 따르되, 원티드 데이터 특성상 **평균연봉** 열을 추가할 수 있다.
- **마감일 열을 추가**하고(상시채용은 "상시"), 별도로 **"마감 임박 공고" 섹션**을 마감일 오름차순으로 정리한다(상시채용 제외). 오늘 기준 남은 일수를 함께 표기하고, 당일/임박 건은 강조(🔴)해 먼저 챙기도록 안내한다.
- 파일은 `data/company-screening/<날짜>-wanted-bookmarks.md`로 저장한다(경로 A의 `<날짜>.md`와 구분).
- 표 상단에 출처 한 줄을 명시한다: "출처: 원티드 — 회사 페이지 `__NEXT_DATA__`(인원·연봉) + `financial-report-for-wanted` API(연도별 매출·영업이익, KODATA) + 공고 상세 API `due_time`(마감일)".

## 에러 핸들링

- 북마크 목록 페이지 접근 실패(로그인 만료 등) → 사용자에게 보고, 경로 A 폴백.
- 특정 회사 API가 404/빈 응답 → 해당 회사만 ⚠️확인불가, 나머지 계속.
- alert/confirm 유발 요소 클릭 금지. 조회 2~3회 연속 실패 시 멈추고 보고.
