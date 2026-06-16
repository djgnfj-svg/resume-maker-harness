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

1. 새 탭 생성 후 원티드의 **저장한 공고/북마크 목록** 페이지로 이동한다.
   - 진입점이 불확실하면 `https://www.wanted.co.kr/` 로 이동해 로그인 상태를 확인하고, 상단 메뉴의 북마크/저장 항목을 통해 목록 페이지로 들어간다.
2. 목록 페이지에서 **회사명**과 **회사 키(companyId)** 를 추출한다.
   - 우선 `get_page_text`로 회사명을 읽고, 정확한 companyId가 필요하면 `javascript_tool`로 페이지의 `__NEXT_DATA__`(`<script id="__NEXT_DATA__">`의 JSON) 또는 공고 링크 `…/company/{companyId}` 패턴에서 추출한다.
3. 무한 스크롤/페이지네이션이면 끝까지 로드해 누락이 없게 한다. 수집된 회사 수를 로그로 남긴다.

## 3. 회사별 재무·인원 수집

각 회사에 대해:

1. **인원·평균연봉**: 회사 페이지(`https://www.wanted.co.kr/company/{companyId}`)의 `__NEXT_DATA__` JSON에서 직원수·평균연봉을 읽는다.
2. **연도별 매출·영업이익**: 아래 API를 호출한다(비로그인 백그라운드 호출 가능, 출처 KODATA).
   ```
   https://www.wanted.co.kr/api/krs/v1/company/{companyId}/financial-report-for-wanted
   ```
   응답에서 최근 3개년 **매출액**(추이)과 최근 **영업이익**(적자=마이너스 여부)을 읽는다.
   - `javascript_tool`로 `fetch(url).then(r=>r.json())` 한 결과를 `console.log`하고 `read_console_messages`로 회수하거나, 해당 URL로 직접 navigate해 JSON 본문을 `get_page_text`로 읽는다.
3. 데이터가 없으면(미공개/신생) → ⚠️확인불가로 표기, 임의 추정 금지.

## 4. 판정·출력

- 판정 기준은 SKILL.md 본문의 "거름(❌) 기준"과 동일하다(적자+매출 정체/감소, 또는 인원 5명 이하).
- 출력 표는 본문의 표 형식을 따르되, 원티드 데이터 특성상 **평균연봉** 열을 추가할 수 있다.
- 파일은 `data/company-screening/<날짜>-wanted-bookmarks.md`로 저장한다(경로 A의 `<날짜>.md`와 구분).
- 표 상단에 출처 한 줄을 명시한다: "출처: 원티드 — 회사 페이지 `__NEXT_DATA__`(인원·연봉) + `financial-report-for-wanted` API(연도별 매출·영업이익, KODATA)".

## 에러 핸들링

- 북마크 목록 페이지 접근 실패(로그인 만료 등) → 사용자에게 보고, 경로 A 폴백.
- 특정 회사 API가 404/빈 응답 → 해당 회사만 ⚠️확인불가, 나머지 계속.
- alert/confirm 유발 요소 클릭 금지. 조회 2~3회 연속 실패 시 멈추고 보고.
