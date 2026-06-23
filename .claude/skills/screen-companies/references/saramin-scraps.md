# 사람인 스크랩(저장 공고) 수집 — 상세 절차

`screen-companies` 입력 경로 B에서 **사람인(saramin.co.kr)** 을 선택했을 때 사용한다. 사용자가 사람인에 **스크랩**한 채용공고의 회사 목록·마감일을 읽어와 스크리닝 대상으로 삼는다. 검증된 방식은 2026-06-23 실행에서 확립되었다.

## 핵심 원리

- 스크랩 **목록**은 로그인 세션에서만 보인다 → 크롬 연결(사용자가 사람인에 로그인된 상태)이 전제다.
- 사람인 스크랩 페이지는 **서버 렌더링**(AJAX 아님)이며, 회사명·공고명·마감일이 HTML에 그대로 들어 있다.
- **재무·인원은 사람인에서 뽑지 않는다.** 회사명만 수집해 SKILL.md 본문의 캐치 조회(경로 A 2단계)로 넘긴다. (캐치는 사람인 그룹 서비스라 데이터가 동일 계열.)

## 1. 크롬 연결·로그인 확인 (전제)

1. `ToolSearch`로 `mcp__claude-in-chrome__tabs_context_mcp`, `navigate`, `javascript_tool`, `tabs_create_mcp`를 한 번에 로드.
2. `tabs_context_mcp`로 연결/탭 상황 파악(없으면 `createIfEmpty:true`로 탭 그룹 생성).
3. 로그인 확인: `https://www.saramin.co.kr/zf_user/` 로 이동 후, 헤더에 "로그아웃" 링크가 있으면 로그인 상태. "로그인"만 보이면 → 자동 수집 불가, 사용자에게 알리고 경로 A(회사명 직접 붙여넣기)로 폴백. 무한 재시도 금지.

## 2. 스크랩 공고 수집

1. **전체를 한 페이지에 로드**한다(기본 20개씩이라 분할됨):
   ```
   https://www.saramin.co.kr/zf_user/persons/scrap-recruit?page=1&page_count=100
   ```
   - `page_count`는 20/50/100만 허용. 스크랩이 100개를 넘으면 `page=2`로 다음 페이지를 추가 수집.
2. `javascript_tool`로 각 `li.row`에서 추출:
   - **회사명** = 항목의 **첫 번째 `<a>` 텍스트** (예: "(주)비아"). 두 번째 `<a>`는 공고명.
   - **마감일** = `span.date` 텍스트 (형식 "~ MM/DD(요일)", 연도 없음).
   - **상태** = `span.sri_btn_immediately`(="입사지원")가 있으면 **진행중**, 없고 본문에 "접수마감"이 있으면 **마감**.
   - 추출 스니펫:
     ```js
     [...document.querySelectorAll('li.row')].map(li => {
       const a = [...li.querySelectorAll('a')].map(x=>x.textContent.replace(/\s+/g,' ').trim()).filter(Boolean);
       const d = li.querySelector('span.date');
       const open = !!li.querySelector('span.sri_btn_immediately');
       return { company: a[0]||null, title: a[1]||null,
                deadline: d?d.textContent.replace(/\s+/g,' ').trim():null,
                status: open?'진행중':'마감' };
     })
     ```
3. **마감일 연도 보정**: "~ MM/DD"에 연도가 없다. 상태가 **진행중**이면 올해(currentDate 기준, MM/DD가 오늘보다 빠르면 내년) 마감으로 본다. **마감** 상태면 지난 공고이므로 "마감"으로만 표기(임박 섹션 제외).

## 3. 회사명 정규화 → 캐치 핸드오프

- 회사 단위로 **중복 제거**(같은 회사 여러 공고 → 1행). 단 마감일은 공고 단위로 보존(진행중 공고만 임박 섹션에 사용).
- 캐치 검색 정확도를 위해 회사명을 **정규화**: 앞뒤 `㈜`·`(주)`·`주식회사` 제거, 영문 병기 괄호 `(...Co.,Ltd.)` 제거, 공백 정리. (원래 표기는 표에 그대로 쓰고, 캐치 검색어로만 정규화형 사용.)
- 정규화한 회사명 목록을 SKILL.md 본문 **"실행 절차(경로 A) 2단계(캐치 조회)"** 로 넘겨 매출 3개년·영업이익·인원을 조회한다.

## 4. 판정·출력

- 판정 기준은 SKILL.md 본문의 "거름(❌) 기준"과 동일.
- 출력은 SKILL.md "입력 경로 B 공통 출력" 규칙을 따른다(출처 열·마감일 열·마감 임박 섹션, 파일명 `<날짜>-saramin-scraps.md`).
- 평균연봉은 사람인 스크랩 목록에 없으므로 비워 둔다(캐치에 있으면 캐치 값 사용).

## 에러 핸들링

- 스크랩 페이지가 404거나 로그인 화면이면 → 로그인 만료. 사용자에게 보고, 경로 A 폴백.
- 특정 회사가 캐치 미등록 → 해당 회사만 ⚠️확인불가, 나머지 계속.
- `javascript_tool`은 쿠키·쿼리스트링·JWT가 섞인 반환을 차단한다. **raw HTML·href를 그대로 반환하지 말고**, `textContent` 등 정제된 짧은 문자열만 반환할 것. async는 IIFE로 감싸지 말고 **top-level await**로 마지막 표현식을 반환(IIFE는 Promise가 `{}`로 직렬화됨).
- 조회 2~3회 연속 실패·무응답 시 멈추고 보고. alert/confirm 유발 요소 클릭 금지.
