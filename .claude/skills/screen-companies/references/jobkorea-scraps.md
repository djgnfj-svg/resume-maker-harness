# 잡코리아 스크랩 공고 수집 — 상세 절차

`screen-companies` 입력 경로 B에서 **잡코리아(jobkorea.co.kr)** 를 선택했을 때 사용한다. 사용자가 잡코리아에 **스크랩**한 채용공고의 회사 목록·마감일을 읽어와 스크리닝 대상으로 삼는다. 검증된 방식은 2026-06-23 실행에서 확립되었다.

## 핵심 원리

- 스크랩 **목록**은 로그인 세션에서만 보인다 → 크롬 연결(잡코리아 로그인 상태)이 전제다.
- 잡코리아 스크랩 페이지는 **서버 렌더링**이며 회사명·공고명·마감일이 HTML에 들어 있다. 단, **긴 회사명은 서버에서 "..."로 잘려 표시**되므로 잘린 항목은 전체명 복원이 필요하다.
- **재무·인원은 잡코리아에서 뽑지 않는다.** 회사명만 수집해 캐치 조회(경로 A 2단계)로 넘긴다.

## 1. 크롬 연결·로그인 확인 (전제)

1. `ToolSearch`로 `mcp__claude-in-chrome__tabs_context_mcp`, `navigate`, `javascript_tool`, `tabs_create_mcp` 로드.
2. `tabs_context_mcp`로 연결/탭 파악.
3. `https://www.jobkorea.co.kr/User/Scrap` 로 이동. **로그인 화면(`Login_ToT.asp`)으로 리다이렉트**되면 미로그인 → 사용자에게 알리고 경로 A 폴백. 페이지 제목이 "스크랩 공고│잡코리아"면 로그인 상태.

## 2. 스크랩 공고 수집

1. **전체를 한 페이지에 로드**(기본 20개씩):
   ```
   https://www.jobkorea.co.kr/User/Scrap?Page=1&PageSize=100
   ```
   - 스크랩이 100개를 넘으면 `Page=2`로 추가 수집.
2. `javascript_tool`로 각 `.listItem`에서 추출:
   - **회사명** = 항목의 **첫 번째 `<a>` 텍스트**. 이 앵커는 잡코리아 기업정보 페이지 `/Recruit/Co_read/C/{id}`로 연결된다.
   - **마감일** = `span.deadline` 텍스트(형식 "~MM/DD", 연도 없음). 본문에 "공고마감"이 있으면 **마감** 상태.
   - 추출 스니펫:
     ```js
     [...document.querySelectorAll('.listItem')].map(r => {
       const a = r.querySelector('a');
       let path=''; try{ path=new URL(a.href,location.origin).pathname; }catch(e){}
       const dl = r.querySelector('.deadline');
       const closed = /공고마감/.test(r.textContent);
       return { company: a?a.textContent.replace(/\s+/g,' ').trim():null, path,
                deadline: dl?dl.textContent.trim():(closed?'마감':null),
                status: closed?'마감':'진행중' };
     })
     ```
3. **잘린 회사명 복원**: 회사명에 `...`/`…`가 포함된 항목은, 그 회사 앵커의 `path`(`/Recruit/Co_read/C/{id}`)를 `fetch`해 전체명을 얻는다(페이지 이동 없이):
   ```js
   const html = await fetch('https://www.jobkorea.co.kr'+path,{credentials:'include'}).then(r=>r.text());
   const doc = new DOMParser().parseFromString(html,'text/html');
   const og = (doc.querySelector('meta[property="og:title"]')||{}).content || doc.title || '';
   const full = (og.replace(/\s+/g,' ').match(/^(.*?)\s*(?:\d{4}년\s*)?기업정보/)||[])[1];
   ```
   `og:title`은 "회사명 기업정보 - …" 형식이라 "기업정보" 앞부분이 전체 회사명이다.
4. **마감일 연도 보정**: 사람인과 동일. 진행중이면 올해(빠르면 내년) 마감, 마감 상태면 "마감" 표기.

## 3. 회사명 정규화 → 캐치 핸드오프

- 회사 단위 **중복 제거**(마감일은 공고 단위 보존, 진행중만 임박 섹션).
- 정규화: `㈜`·`(주)`·`주식회사`·영문 병기 괄호 제거. (표에는 원표기, 캐치 검색어만 정규화형.)
- 정규화 회사명 목록을 SKILL.md "경로 A 2단계(캐치 조회)"로 넘긴다.

## 4. 판정·출력

- 판정 기준은 SKILL.md "거름(❌) 기준"과 동일.
- 출력은 "입력 경로 B 공통 출력" 규칙(출처 열·마감일 열·마감 임박 섹션). 파일명 `<날짜>-jobkorea-scraps.md`.
- 평균연봉 열은 비워 둔다(잡코리아 스크랩 목록 미제공, 캐치에 있으면 사용).

> 참고: 잡코리아 기업정보 페이지(`Co_read/C/{id}`)에는 직원수가 있다. 캐치가 미등록인 회사는 이 페이지의 직원수를 보조로 쓸 수 있으나, 기본은 캐치 일원화를 유지한다.

## 에러 핸들링

- 로그인 화면 리다이렉트 → 경로 A 폴백.
- 잘린 회사명 복원 fetch 실패 → 해당 회사 표기를 잘린 원문 그대로 두고 ⚠️확인불가 처리(임의 추정 금지).
- 캐치 미등록 → 해당 회사만 ⚠️확인불가, 나머지 계속.
- `javascript_tool` 반환 차단 주의: raw HTML·href·쿼리스트링을 그대로 반환 금지, 정제된 짧은 값만. async는 top-level await 사용(IIFE 금지).
- 조회 2~3회 연속 실패·무응답 시 멈추고 보고.
