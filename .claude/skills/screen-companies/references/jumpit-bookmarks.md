# 점핏 스크랩(북마크) 포지션 수집 — 상세 절차

`screen-companies` 입력 경로 B에서 **점핏(jumpit.saramin.co.kr)** 을 선택했을 때 사용한다. 사용자가 점핏에 **스크랩**한 포지션의 회사 목록·마감일을 읽어와 스크리닝 대상으로 삼는다. 검증된 방식은 2026-06-23 실행에서 확립되었다.

## 핵심 원리

- 점핏은 **SPA(Next.js App Router/RSC)** 이고 스크랩 목록 페이지(`/scraps`)는 무한스크롤이지만 **자동 스크롤로는 30건에서 멈춘다**(IntersectionObserver가 재발화하지 않음). → DOM 긁기 대신 **백엔드 API를 직접 호출**하는 것이 정확하고 빠르다.
- 스크랩 API는 별도 서브도메인 `jumpit-api.saramin.co.kr`에 있고 **쿠키 인증**(로그인 세션)으로 동작한다 → 크롬 연결(점핏 로그인 상태)이 전제다.
- **재무·인원은 점핏에서 뽑지 않는다.** 회사명만 수집해 캐치 조회(경로 A 2단계)로 넘긴다.

## 1. 크롬 연결·로그인 확인 (전제)

1. `ToolSearch`로 `mcp__claude-in-chrome__tabs_context_mcp`, `navigate`, `javascript_tool`, `tabs_create_mcp` 로드.
2. `tabs_context_mcp`로 연결/탭 파악.
3. 새 탭에서 `https://jumpit.saramin.co.kr/` 로 이동(API fetch가 same-site 쿠키로 동작하려면 점핏 도메인 위에 있어야 한다). 헤더에 "로그인"이 보이면 미로그인 → 경로 A 폴백.

## 2. 스크랩 포지션 수집 (API 직접 호출)

점핏 도메인 페이지가 떠 있는 탭에서 `javascript_tool`로 호출한다(같은 탭, 페이지 이동 불필요):

```js
const base = 'https://jumpit-api.saramin.co.kr/api/user/scrap/list';
// size=100이면 한 번에 전부(보통 <100건). 넘으면 page=1,2.. 로 추가.
const r = await fetch(base + '?size=100', { credentials:'include', headers:{Accept:'application/json'} });
const j = await r.json();
const content = (j.result && j.result.content) || [];   // 배열
// 페이지네이션 메타: j.result.totalElements / totalPages / last
content.map(x => ({
  company: x.companyName,        // 회사명 (정제 불필요, 깔끔)
  title: x.title,                // 공고명
  positionId: x.positionId,
  closedAt: x.closedAt           // 마감일 "YYYY-MM-DD HH:mm:ss" (과거면 마감)
}))
```

- 응답 구조: `{ message, status, code, result:{ content:[{ id, companyName, logo, positionId, title, closedAt, techStacks, hiddenPosition, draft, ... }], totalElements, totalPages, last, size, number, ... } }`
- `closedAt`을 currentDate와 비교: **미래면 진행중**(남은 일수 계산 가능), **과거면 마감**.
- ⚠️ 반환 차단 회피: fetch URL에 쿼리스트링이 있으므로 **URL 문자열을 결과로 되돌려주지 말 것**(차단됨). 회사명·마감일 등 정제된 값만 반환. async는 top-level await 사용(IIFE 금지 — Promise가 `{}`로 직렬화됨).

## 3. 회사명 정규화 → 캐치 핸드오프

- 포지션 단위로 받되 회사 단위 **중복 제거**(마감일은 포지션 단위 보존, 진행중만 임박 섹션).
- `companyName`은 대체로 정제돼 있으나, `㈜`·`(주)`·`주식회사`가 붙은 경우 캐치 검색어로는 제거형을 쓴다.
- 정규화 회사명 목록을 SKILL.md "경로 A 2단계(캐치 조회)"로 넘긴다.

## 4. 판정·출력

- 판정 기준은 SKILL.md "거름(❌) 기준"과 동일.
- 출력은 "입력 경로 B 공통 출력" 규칙(출처 열·마감일 열·마감 임박 섹션). 파일명 `<날짜>-jumpit-bookmarks.md`.
- `closedAt`이 ISO라 마감일·남은 일수 계산이 가장 정확하다(점핏의 강점). 평균연봉은 미제공이라 비워 둔다.

## 에러 핸들링

- API가 401/리다이렉트/HTML 반환 → 로그인 만료 또는 도메인 밖에서 호출. 점핏 도메인 탭 위에서 재시도, 그래도 실패면 경로 A 폴백.
- `result.content`가 비면 스크랩 없음으로 보고.
- 캐치 미등록 회사 → 해당 회사만 ⚠️확인불가, 나머지 계속.
- 조회 2~3회 연속 실패·무응답 시 멈추고 보고. alert/confirm 유발 요소 클릭 금지.
