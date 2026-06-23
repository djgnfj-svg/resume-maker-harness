# 점핏 공고 발견(검색→매칭→스크랩) — 상세 절차

`discover-jobs` 스킬에서 점핏(jumpit.saramin.co.kr)을 능동 탐색할 때 사용한다. 마스터 이력서에서 뽑은 검색어로 공고를 검색하고, 매칭되는(높음·중간) 공고를 **스크랩(=북마크)에 추가**한다. 검증된 방식은 2026-06-23 크롬 정찰에서 확립되었다.

저장한 스크랩을 나중에 재무 스크리닝하는 절차는 `screen-companies/references/jumpit-bookmarks.md`(경로 B)에 있다 — 발견은 그 앞단이다.

## 핵심 원리

- API 도메인은 **`jumpit-api.saramin.co.kr`** 이다(페이지는 `jumpit.saramin.co.kr`). 모두 로그인 세션 쿠키로 `fetch(..., {credentials:'include', headers:{'Accept':'application/json'}})` 호출.
- **검색 한 번으로 매칭·필터에 필요한 필드를 다 얻는다** — 회사명·기술스택·경력·마감일까지 검색 응답에 있어, 원티드와 달리 공고별 상세 호출이 필수가 아니다.
- **중복 제거가 공짜다(로그인 시):** 검색 응답 각 공고에 `scraped`(이미 스크랩) · `applied`(이미 지원) 불리언이 있어, 이미 스크랩한 공고를 별도 조회 없이 거른다.
- **스크랩은 토글이 아니다:** 추가는 `POST .../scrap`, 해제는 `POST .../unscrap` 로 **경로가 다르다**(DELETE는 405). 추가는 멱등이라 이미 스크랩된 공고에 다시 POST해도 안전하지만, 불필요하니 `scraped === false` 인 공고에만 호출한다.

## 1. 크롬 연결·로그인 확인 (전제)

1. `ToolSearch`로 `mcp__claude-in-chrome__tabs_context_mcp`·`navigate`·`javascript_tool`·`read_network_requests`·`tabs_create_mcp`를 로드.
2. `tabs_context_mcp`로 연결 확인. 새 탭에서 `https://jumpit.saramin.co.kr` 를 한 번 연다(이후 API는 이 탭에서 `fetch`).
3. **로그인 확인(필수):** 검색은 비로그인도 되지만 **스크랩은 로그인 필수**다.
   ```
   GET https://jumpit.saramin.co.kr/api/auth/verified-status
   ```
   `200` 이면 로그인됨. `401`(Unauthorized)이면 → 사용자에게 점핏 로그인을 요청하고, 로그인 전까지 점핏은 건너뛴다(자격증명은 사용자가 직접 입력).

## 2. 검색 실행

### 검색 API
```
GET https://jumpit-api.saramin.co.kr/api/positions
    ?keyword={검색어}
    &sort=relation
    &highlight=false
    &page={1,2,3,...}
```
- `keyword`: 마스터 이력서에서 뽑은 검색어(백엔드, AI 엔지니어, AI Agent, 풀스택, Python, FastAPI, Django, LangGraph 등). 검색어별로 호출하고 `id`로 중복 제거해 합친다.
- `sort`: `relation`(관련도) 또는 `popular`/`recent`. `jobCategory`(예: 1=개발 계열, 15 등)로 직무 카테고리 필터 가능.
- `highlight=false`: 결과 텍스트의 `<span>` 강조 태그를 빼고 싶을 때. (true면 `title`·`jobCategory`에 `<span>…</span>` 가 섞이니 파싱 시 태그 제거.)
- 페이지네이션: `page`를 1부터 증가시키며 `result.totalCount`까지. 검색어·사이트당 상한을 두고, 잘라낸 건수는 리포트에 고지(SKILL.md 9단계 "조용한 누락 금지").

### 검색 응답 파싱
응답 = `{ message, status, code, result }`. `result.{ totalCount, page, positions[] }`. `positions[]` 각 원소:
- `id` → **positionId** (스크랩·상세·링크 `/position/{id}` 키)
- `title` → 공고명 (`highlight=false`면 평문, 아니면 `<span>` 제거)
- `companyName` → 회사명 (중복·거른회사 대조)
- `techStacks[]` → 요구 기술(매칭 핵심)
- `minCareer`, `maxCareer` → 경력 범위(년). 하드 필터(주니어~3년).
- `closedAt` → 마감일(ISO). `alwaysOpen`(true=상시채용). 하드 필터(마감 지난 공고 제외).
- `locations[]` → 근무지(참고; 지역은 하드 필터 아님)
- `jobCategory` → 직무 카테고리(직무 필터 보조)
- `scraped` → 이미 스크랩 여부(중복 제거; **로그인 시에만 채워짐**)
- `applied` → 이미 지원 여부, `newcomer` → 신입 채용 여부

### 상세 API (선택 — 요건 텍스트가 더 필요할 때)
```
GET https://jumpit-api.saramin.co.kr/api/position/{id}
```
`result.{ scrap, responsibility(주요업무), qualifications(자격요건), preferredRequirements(우대), welfares, minCareer, maxCareer, closedAt, alwaysOpen, ... }`. `scrap` 불리언으로 단건 스크랩 상태 확인 가능. 검색 응답으로 매칭이 충분하면 호출 생략.

## 3. 매칭·하드 필터 (스킬 본문 기준 적용)

- 경력: `minCareer`가 너무 높으면(예: 4년↑) 제외. 주니어~경력 3년 범위.
- 직무: `title`·`jobCategory`·`techStacks`로 백엔드·AI·Agent·풀스택만. **제외**: 프론트/모바일 전용, 데이터엔지니어, QA, 인프라·클라우드만.
- 마감: `alwaysOpen===true`는 상시(포함), 아니면 `closedAt`이 과거면 제외.
- 매칭 등급(높음/중간/낮음)은 `techStacks`·`title`(+필요시 상세 `responsibility`·`qualifications`)을 마스터 이력서와 대조해 스킬 본문 7단계 기준으로 판정.

## 4. 자동 스크랩

높음·중간 등급이고 **`scraped === false`** 인 공고에만:
```
POST https://jumpit-api.saramin.co.kr/api/position/{id}/scrap
     (credentials: include, 바디 불필요)
```
- 호출 후 검증: `GET /api/position/{id}` 의 `result.scrap` 이 `true` 인지 확인(스크랩한 공고당 추가 호출 1회).
- 추가는 멱등(이미 스크랩돼 있어도 200·상태 유지)이라 토글 사고는 없지만, 불필요한 호출을 피하려 `scraped===false`에만 호출.
- 해제(정찰·복구용): `POST .../unscrap`. **`DELETE`는 405라 쓰지 않는다.**
- 실패(비정상 상태 코드)면 해당 공고 "스크랩 실패"로 리포트에 표기하고 다음 진행.

## 5. 중복 제거 — 이미 스크랩/거른 회사

- **이미 스크랩:** 검색 응답 `scraped === true` 인 공고 제외(로그인 시 추가 호출 불필요).
- **이미 거른 회사:** `companyName`을 정규화(`㈜`·`(주)`·`주식회사`·영문 병기 괄호 제거) 후 `data/company-screening/*.md` 의 ❌거름 회사 목록과 대조해 일치 시 제외.
- 전체 스크랩 목록이 따로 필요하면(예: `scraped` 누락 대비) 경로 B 재사용:
  ```
  GET https://jumpit-api.saramin.co.kr/api/user/scrap/list?page=1&size=100
  ```
  응답 `result.content[]`(Spring Page: `totalElements`·`last`·`number` 등). 로그인 필수.

## 에러 핸들링

- `verified-status` 401(로그인 만료) → 점핏 건너뛰고 사용자에게 로그인 요청.
- 특정 공고 상세/스크랩 API 실패 → 그 공고만 건너뛰고(또는 "스크랩 실패") 계속.
- alert/confirm 유발 클릭 금지(API `fetch`만 쓰면 발생 안 함). 조회 2~3회 연속 실패 시 멈추고 보고.

## 정찰 메모 (2026-06-23 확립)

- 점핏은 Next.js라 목록 페이지가 RSC로 렌더되지만, 데이터는 `jumpit-api.saramin.co.kr/api/positions` JSON API로 받는다(좌표 클릭/DOM 파싱 불필요).
- 스크랩 추가/해제 메서드는 fetch/XHR 가로채기로 확인: 추가 `POST .../scrap`, 해제 `POST .../unscrap`. `DELETE .../scrap` 는 405. POST 추가는 토글이 아니라 멱등.
- 정찰 시 테스트 스크랩은 `unscrap` 으로 즉시 원상복구했다(최종 `scrap:false` 확인).
