# 원티드 공고 발견(검색→매칭→북마크) — 상세 절차

`discover-jobs` 스킬에서 원티드(wanted.co.kr)를 능동 탐색할 때 사용한다. 마스터 이력서에서 뽑은 검색어로 공고를 검색하고, 매칭되는(높음·중간) 공고를 **북마크에 추가**한다. 검증된 방식은 2026-06-23 크롬 정찰에서 확립되었다.

저장한 북마크를 나중에 재무 스크리닝하는 절차는 `screen-companies/references/wanted-bookmarks.md`(경로 B)에 있다 — 발견은 그 앞단이다.

## 핵심 원리

- 검색·상세·북마크 모두 원티드 비공개 API(`/api/chaos/...`)를 **로그인 세션 쿠키로 `fetch`** 해서 처리한다. 페이지를 일일이 navigate하지 않는다(상태 유실·속도 문제).
- **중복 제거가 공짜다:** 검색 응답의 각 공고에 `is_bookmark` 불리언이 들어있어, 이미 북마크된 공고를 별도 조회 없이 바로 거를 수 있다.
- ⚠️ **북마크 추가는 토글이다:** `POST /api/chaos/bookmarks/v1/{jobId}` 는 상태를 뒤집는다(안 돼 있으면 추가, 돼 있으면 **해제**). 그러므로 **반드시 `is_bookmark === false` 인 공고에만 POST** 한다. 이미 북마크된 공고에 POST하면 사용자의 기존 북마크가 풀린다.

## 1. 크롬 연결 확인 (전제)

1. `ToolSearch`로 `mcp__claude-in-chrome__tabs_context_mcp`·`navigate`·`javascript_tool`·`tabs_create_mcp`를 로드.
2. `tabs_context_mcp` 호출로 연결/로그인 확인. **크롬 미연결이거나 원티드 로그인 세션이 아니면** 자동 탐색 불가 → 사용자에게 알리고 중단(폴백 없음).
3. 새 탭 생성 후 원티드 도메인을 한 번 연다(예: `https://www.wanted.co.kr`). 이후 모든 API는 이 탭에서 `fetch(..., {credentials:'include'})` 로 호출한다.

## 2. 검색 실행

### 검색 API
```
GET https://www.wanted.co.kr/api/chaos/search/v1/position
    ?query={검색어}
    &country=kr
    &years={경력하한}&years={경력상한}
    &locations=all
    &sort=job.recommend_order
    &limit=12&offset={0,12,24,...}
```
- `query`: 마스터 이력서에서 뽑은 검색어(예: 백엔드, AI 엔지니어, AI Agent, 풀스택, Python, FastAPI, LangGraph). 검색어별로 따로 호출하고 결과를 합친다(jobId로 중복 제거).
- `years`는 **두 번** 보낸다 = 경력 하한·상한. 하드 필터(주니어~경력 3년)에 맞추려면 `years=0&years=3`.
- 페이지네이션: `offset`을 limit(12)씩 늘리며 `total_count`까지. 검색어·사이트당 상한을 두고(예: offset 0~48, 최대 ~50건) 초과분은 잘라낸 건수를 리포트에 고지.

### 검색 응답 파싱
응답 = `{ total_count, data:[...], links }`. `data[]` 각 원소:
- `id` → **jobId** (북마크·상세 호출 키, 링크는 `/wd/{id}`)
- `is_bookmark` → 이미 북마크 여부(중복 제거에 사용)
- `company.id`, `company.name` → 회사(중복·거른회사 대조)
- `position` → 공고명
- `annual_from`, `annual_to` → 공고가 받는 경력 범위(년)
- `category_tag.parent_id`(518=개발), `category_tag.id`(세부 직무) → 직무 필터 보조
- `employment_type`(예: regular)

### 상세 필드(마감일·요건·기술) — 매칭·필터용
공고별로:
```
GET https://www.wanted.co.kr/api/chaos/jobs/v4/{jobId}/details
```
응답 `data.job` 에서:
- `due_time` → 마감일(ISO 문자열, **null = 상시채용**). 하드 필터(마감 지난 공고 제외)·리포트 마감일 열.
- `detail.{position, intro, main_tasks, requirements, preferred_points, benefits}` → 매칭 평가의 핵심 텍스트(주요업무·자격요건·우대).
- `skill_tags`(배열) → 요구 기술(있을 때).
- `annual_from`/`annual_to`, `company.id`, `is_bookmark` 도 여기 있음.

> 상세 호출은 공고 수만큼 발생하므로, 검색에서 1차 필터(경력·직무)로 후보를 줄인 뒤 상세를 호출한다.

## 3. 매칭·하드 필터 (스킬 본문 기준 적용)

- 경력: `annual_from`이 너무 높으면(예: 하한 4년↑) 제외. 검색의 `years=0&years=3` 으로 1차 제한됨.
- 직무: `position`·`category_tag`·`detail`로 백엔드·AI·Agent·풀스택만.
- 마감: `due_time`이 과거면 제외(null=상시는 포함).
- 매칭 등급(높음/중간/낮음)은 `detail`·`skill_tags`를 마스터 이력서와 대조해 스킬 본문 7단계 기준으로 판정.

## 4. 자동 북마크

높음·중간 등급이고 **`is_bookmark === false`** 인 공고에만:
```
POST https://www.wanted.co.kr/api/chaos/bookmarks/v1/{jobId}
     (credentials: include, 바디 불필요)
```
- 호출 후 `jobs/v4/{jobId}/details` 의 `is_bookmark` 가 `true` 인지 확인해 성공 검증.
- ⚠️ 토글이므로 `is_bookmark === true` 인 공고엔 **절대 POST 금지**(기존 북마크 해제됨).
- 실패(상태 코드 비정상)면 해당 공고 "북마크 실패"로 리포트에 표기하고 다음 진행.

## 5. 중복 제거 — 이미 북마크/거른 회사

- **이미 북마크:** 검색 응답 `is_bookmark === true` 인 공고 제외(추가 호출 불필요).
- **이미 거른 회사:** `data/company-screening/*.md` 의 ❌거름 회사명과 `company.name`(정규화 후) 대조해 제외.
- 전체 북마크 목록이 따로 필요하면 경로 B(`wanted-bookmarks.md`)의 `/profile/bookmarks` 수집 절차를 재사용.

## 에러 핸들링

- 로그인 만료(검색 API 401/리다이렉트) → 중단·안내.
- 특정 공고 상세/북마크 API 실패 → 그 공고만 건너뛰고(또는 "북마크 실패" 표기) 계속.
- alert/confirm 유발 클릭 금지(API `fetch`만 쓰면 발생 안 함). 조회 2~3회 연속 실패 시 멈추고 보고.

## 정찰 메모 (2026-06-23 확립)

- 좌표 클릭으로 카드 북마크 아이콘을 누르는 방식은 불안정 → **API `fetch`** 가 표준이다.
- `read_network_requests` 도구가 북마크 XHR을 못 잡는 경우가 있었음 → `performance.getEntriesByType('resource')` 또는 fetch/XHR 가로채기로 엔드포인트·메서드를 확인했다. 메서드 확정 결과: 추가·해제 모두 `POST`(토글).
- 정찰 시 테스트로 추가한 북마크는 같은 POST를 한 번 더 보내 즉시 원상복구했다.
