# 사람인 공고 발견(검색→매칭→스크랩) — 상세 절차

`discover-jobs` 스킬에서 사람인(saramin.co.kr)을 능동 탐색할 때 사용한다. 마스터 이력서 검색어로 공고를 검색하고, 매칭되는(높음·중간) 공고를 **스크랩에 추가**한다. 검증된 방식은 2026-06-23 크롬 정찰에서 확립되었다.

저장한 스크랩의 회사 목록 수집(재무 스크리닝용)은 `screen-companies/references/saramin-scraps.md`(경로 B)에 있다 — 발견은 그 앞단이다.

## 핵심 원리

- 사람인은 **서버 렌더 HTML** 사이트다(원티드·점핏 같은 JSON 검색 API 아님). 검색 결과는 페이지 DOM을 파싱한다.
- 스크랩 추가/해제는 **AJAX `fetch`** 로 된다(쿠키 인증). 추가 시 응답 `scrap_cnt`(전체 스크랩 수)로 성공 검증 가능.
- **로그인 필수**(스크랩). 로그인 시 검색 페이지가 "○○님" 개인화로 보인다.
- 사람인 검색 목록은 **공고에 스크랩 여부 플래그를 주지 않는다** → 중복 제거는 기존 스크랩 목록(경로 B)과 rec_idx로 대조한다.
- 기술스택을 목록에 거의 노출하지 않으므로, 매칭은 **공고명 + 직무 분야(job_sector)** 중심으로 한다.

## 1. 크롬 연결·로그인 확인 (전제)

1. 브라우저 도구 로드 후 `tabs_context_mcp`로 연결 확인. 새 탭에서 `https://www.saramin.co.kr/zf_user/` 를 열고, 페이지에 사용자 이름 개인화("○○님…")가 보이면 로그인됨. 안 되면 사람인 로그인 요청 후 건너뜀.

## 2. 검색 실행

### 검색 URL (HTML)
```
https://www.saramin.co.kr/zf_user/search/recruit?searchword={검색어}&recruitSort=relation&recruitPageCount=40&page={1,2,...}
```
- `searchword`: 마스터 이력서 검색어(백엔드, AI, AI Agent, 풀스택, Python, FastAPI, Django 등).
- `recruitSort=relation`(관련도), `recruitPageCount`(페이지당 건수), `page`(페이지네이션). 경력은 URL 파라미터(`exp_cd`/`exp_min`/`exp_max`)로도 걸 수 있으나, 목록의 `.job_condition` 경력 텍스트로 파싱·필터하는 게 단순·안정적이다.

### 결과 파싱 (DOM)
`get_page_text`가 아니라 `javascript_tool`로 `document.querySelectorAll('.item_recruit')`(페이지당 40개) 순회. 각 항목에서:
- `el.getAttribute('value')` → **rec_idx** (공고 id; 스크랩·중복 키)
- `.job_tit a` 텍스트 → 공고명
- `.corp_name a` 텍스트 → 회사명
- `.job_condition` 텍스트 → 지역 / **경력**(예: "경력 3~20년") / 학력 / 고용형태 (경력 하드 필터에 사용)
- `.job_date .date` 텍스트 → 마감일(예: "~ 07/10(금)"; "상시"·"채용시"는 상시)
- `.job_sector` 텍스트 → 직무 분야(예: "백엔드/서버개발, 웹개발") — 직무 필터·매칭 보조
- ⚠️ 출력에 URL/쿼리스트링을 그대로 담으면 도구가 [BLOCKED] 처리하므로, **텍스트 필드만** 추출해 반환한다(숫자 id는 4자리+를 `#` 등으로 치환해도 됨).

## 3. 매칭·하드 필터

- 경력: `.job_condition`의 경력 하한이 3년 초과면 제외(주니어~3년). "경력무관"·"신입"은 포함.
- 직무: 공고명·`job_sector`로 백엔드·AI·Agent·풀스택만. 제외: 프론트 전용·데이터엔지니어·QA·기획·영업 등.
- 마감: `job_date`가 과거면 제외("상시"·"채용시"는 포함).
- 등급(높음/중간/낮음)은 공고명+직무분야를 마스터 이력서와 대조(기술스택 미노출이라 제목·분야 기반).

## 4. 자동 스크랩

높음·중간이고 아직 스크랩 안 된 rec_idx에만, `javascript_tool`에서 `fetch`:
```
// 추가
POST https://www.saramin.co.kr/zf_user/recruit/person-recruit-scrap-ajax
  headers: { 'Content-Type':'application/x-www-form-urlencoded; charset=UTF-8', 'X-Requested-With':'XMLHttpRequest' }
  credentials: 'include'
  body: 'rec_idx={rec_idx}&status=scrap'
  → 응답 JSON { error_code:0, scrap_cnt:N }  (error_code 0 = 성공)
```
```
// 해제(정찰·복구용)
POST .../zf_user/recruit/person-recruit-scrap-unset-ajax   body: 'rec_idx={rec_idx}&status=cancel'
```
- `scrap_cnt`(전체 스크랩 수) 증가로 성공 확인. `error_code`가 0이 아니면 "스크랩 실패" 표기 후 계속.
- 좌표로 `.icon_scrap_star` 클릭도 되지만 리스트 리플로우로 불안정 → **`fetch` 권장**.

## 5. 중복 제거 — 이미 스크랩/거른 회사

- **이미 스크랩:** 사람인 검색 목록엔 스크랩 플래그가 없으므로, 경로 B(`saramin-scraps.md`)의 스크랩 목록 페이지
  `https://www.saramin.co.kr/zf_user/mypage/scrap/scrap-recruit?page_count=100`
  에서 기존 rec_idx 집합을 먼저 모아, 검색 결과와 대조해 제외한다.
- **이미 거른 회사:** 회사명 정규화(`㈜`·`(주)`·`주식회사`·영문 병기 괄호 제거) 후 `data/company-screening/*.md` ❌거름 목록과 대조해 제외.

## 에러 핸들링

- 로그인 안 됨(개인화 미표시/리다이렉트) → 사람인 건너뛰고 사용자에게 로그인 요청.
- 특정 공고 스크랩 `error_code≠0` → 그 공고만 "스크랩 실패" 표기, 계속.
- alert/confirm 유발 클릭 금지(`fetch`만 쓰면 발생 안 함). 2~3회 연속 실패 시 멈추고 보고.

## 정찰 메모 (2026-06-23 확립)

- 스크랩 메서드는 fetch/XHR 가로채기로 확인: 추가 `person-recruit-scrap-ajax`(`status=scrap`), 해제 `person-recruit-scrap-unset-ajax`(`status=cancel`). 응답에 `scrap_cnt` 포함.
- `fetch` 직접 호출(폼 인코딩 + `X-Requested-With`)로 add→remove 검증 성공(scrap_cnt 15→16→15)했고, 테스트 스크랩은 즉시 원상복구.
- `javascript_tool` 출력이 쿠키/쿼리스트링 포함 시 [BLOCKED] 되니, DOM·응답에서 **텍스트/필드명만** 반환할 것.
