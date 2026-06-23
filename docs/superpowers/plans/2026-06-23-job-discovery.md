# discover-jobs 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 잡사이트를 능동 탐색해 마스터 이력서와 매칭되는 공고를 발견·북마크하는 새 스킬 `discover-jobs`를 만든다.

**Architecture:** 새 스킬 `.claude/skills/discover-jobs/SKILL.md`(4사 공통 파이프라인) + 사이트별 어댑터 reference. 매칭은 에이전트 정성 등급(높음/중간/낮음). 높음·중간만 그 사이트 북마크에 추가하고 멈춘다(재무 스크리닝은 기존 `screen-companies` 경로 B가 별도 처리).

**Tech Stack:** Claude Code 스킬(markdown), `mcp__claude-in-chrome__*` 브라우저 자동화(라이브 정찰), 마스터 이력서(`data/master-resume.md`).

## 이 계획의 특수성 (꼭 읽을 것)

- **단위 테스트 하네스가 없다.** 이 레포의 "구현"은 markdown 스킬/문서 작성이다. 따라서 각 태스크의 "검증"은 pytest가 아니라 **① 문서 일관성 재독 ② 크롬 라이브 동작 확인**이다.
- **어댑터(Task 2·3·5·6)는 라이브 정찰이 본질이다.** 사이트의 검색 API 엔드포인트·DOM 셀렉터·북마크 버튼 동작은 **사전에 알 수 없고 정찰로 알아내야 한다**(경로 B reference들이 정확히 이렇게 만들어짐). 그래서 이 태스크들의 "코드"는 *최종 API 스펙*이 아니라 **정찰 절차 + 결과 문서가 채워야 할 구조**다. 이는 플레이스홀더가 아니라 의도된 설계다.
- **사용자 크롬 세션 의존:** Task 2·3·5·6과 통합 검증(Task 4·7)은 **사용자가 해당 사이트에 로그인된 크롬이 연결돼 있어야** 실행 가능하다. 구현 착수 전 크롬 연결을 확인하라.

## Global Constraints

- 스킬·문서·보고는 모두 **한국어**.
- 매칭 이유에 **마스터 이력서에 없는 내용을 지어내지 않는다**(tailor-resume 원칙).
- **alert/confirm 유발 요소 클릭 금지.** 조회/클릭 2~3회 연속 실패 시 멈추고 보고(무한 재시도 금지).
- 회사명 정규화 규칙(중복 대조용): `㈜`·`(주)`·`주식회사`·영문 병기 괄호 제거.
- 북마크 대상 등급 = **높음 + 중간**. 낮음은 북마크 안 함(리포트엔 남김).
- 결과 파일 경로 = `data/job-discovery/<날짜>-discovered.md` (`<날짜>`=YYYY-MM-DD).
- 설계 근거: `docs/superpowers/specs/2026-06-23-job-discovery-design.md`.

## 파일 구조

- Create: `.claude/skills/discover-jobs/SKILL.md` — 4사 공통 파이프라인(검색 seed·필터·중복·매칭·북마크·리포트·에러). 사이트 라우팅 표.
- Create: `.claude/skills/discover-jobs/references/wanted-discovery.md` — 원티드 어댑터(1차).
- Create: `.claude/skills/discover-jobs/references/jumpit-discovery.md` — 점핏 어댑터(1차).
- Create: `.claude/skills/discover-jobs/references/saramin-discovery.md` — 사람인 어댑터(2차).
- Create: `.claude/skills/discover-jobs/references/jobkorea-discovery.md` — 잡코리아 어댑터(2차).
- Create: `data/job-discovery/.gitkeep` — 출력 디렉토리.
- Modify: `.claude/skills/resume-orchestrator/SKILL.md` — 흐름 분류표에 "E. 회사 발견" 추가.
- Modify: `CLAUDE.md` — 변경 이력 1줄, 트리거 안내.

---

# Phase 1 — 1차(원티드·점핏): 그 자체로 동작하는 기능

## Task 1: 스킬 골격 `SKILL.md` + 출력 디렉토리

**Files:**
- Create: `.claude/skills/discover-jobs/SKILL.md`
- Create: `data/job-discovery/.gitkeep`

**Interfaces:**
- Produces: 스킬 `discover-jobs`. 파이프라인 9단계, 사이트 라우팅 표(어댑터 경로 4종), 결과 파일 규약 `data/job-discovery/<날짜>-discovered.md`. 이후 어댑터 태스크는 라우팅 표의 자기 행과 파이프라인 4·8단계(검색·북마크)가 부르는 어댑터 인터페이스를 구현한다.

- [ ] **Step 1: 출력 디렉토리 placeholder 생성**

`data/job-discovery/.gitkeep` 빈 파일 생성(디렉토리를 git에 남기기 위함).

- [ ] **Step 2: `SKILL.md` 작성**

아래 내용 그대로 `.claude/skills/discover-jobs/SKILL.md`에 작성한다:

````markdown
---
name: discover-jobs
description: 사용자가 "이력서 기준으로 지원할 회사 찾아줘", "잡사이트 훑어서 나한테 맞는 공고 북마크해줘", "원티드/점핏/사람인/잡코리아에서 매칭되는 공고 찾아줘" 등 잡사이트를 능동 탐색해 마스터 이력서와 맞는 새 공고를 발견·북마크해 달라고 하면 이 스킬을 사용한다. (이미 저장한 공고를 재무 기준으로 거르는 것은 screen-companies다 — 발견≠스크리닝.) 크롬 연결이 전제다.
---

# discover-jobs — 이력서 기반 회사 발견

마스터 이력서(`data/master-resume.md`)에서 검색어를 자동 추출해 잡사이트를 검색하고, 마스터와 매칭되는 공고를 그 사이트의 **북마크/스크랩에 추가**한다. 재무 스크리닝은 하지 않는다 — 북마크가 쌓이면 사용자가 `screen-companies` 경로 B를 별도 실행한다.

**screen-companies와의 구분:** 발견 = 새 공고를 찾아 북마크 / 스크리닝 = 저장된 공고 재무 판정.

## 전제
- **크롬 연결 필수**(해당 사이트 로그인 세션). 미연결/미로그인이면 자동 탐색 불가 → 사용자에게 알리고 중단. 폴백 없음.
- `data/master-resume.md` 존재.

## 실행 파이프라인

### 1. 검색 seed 자동 추출
`data/master-resume.md`를 읽어 검색어 세트를 만든다:
- **직무 키워드**: 백엔드, AI 엔지니어, AI Agent, 풀스택 (마스터의 직무·한 줄 소개 기준)
- **핵심 기술**: Python, FastAPI, Django, LangGraph 등 마스터 스킬 상위 항목
사용자가 직무를 좁혀 지정하면(예: "AI 직무만") 그 범위로 제한한다.

### 2. 크롬 연결·로그인 확인
`ToolSearch`로 `tabs_context_mcp`·`navigate`·`get_page_text`·`javascript_tool`·`read_network_requests`·`tabs_create_mcp`를 한 번에 로드 → `tabs_context_mcp`로 연결 확인. 미연결/미로그인이면 중단·안내.

### 3. 제외 목록 로드
- **이미 북마크된 공고**: 각 사이트 어댑터의 "기존 북마크 조회"로 확보(경로 B reference 재사용).
- **이미 거른 회사**: `data/company-screening/*.md`를 읽어 ❌거름 판정 회사명을 수집. 회사명 정규화(`㈜`·`(주)`·`주식회사`·영문 병기 제거) 후 대조.

### 4. 사이트별 검색
요청에서 사이트를 판단(특정 안 하면 로그인된 사이트 전부). 각 사이트의 어댑터 reference를 읽고 검색 실행 → 공고 후보 수집(공고명·회사명·링크·마감일·경력요건·기술스택).

| 사이트 | 어댑터 | 도입 |
|--------|--------|------|
| 원티드 | `references/wanted-discovery.md` | 1차 |
| 점핏 | `references/jumpit-discovery.md` | 1차 |
| 사람인 | `references/saramin-discovery.md` | 2차 |
| 잡코리아 | `references/jobkorea-discovery.md` | 2차 |

### 5. 하드 필터
- **경력**: 주니어~경력 3년 이하만. "경력 7년+ 필수" 등 제외.
- **직무**: 백엔드·AI·Agent·풀스택만. 프론트 전용·데이터엔지니어·QA 등 부합 낮은 직무 제외.
- **마감**: 이미 마감된 공고 제외(상시채용은 포함).

### 6. 중복 제거
3단계 제외 목록 적용 — 이미 북마크된 공고, 이미 거른 회사 빼기.

### 7. 매칭 평가
각 후보 공고를 `data/master-resume.md`와 대조해 등급+이유 산출:
- **높음**: 핵심 직무 일치 + 요구 기술 다수가 마스터에 있음 + 연차 적합.
- **중간**: 직무 인접 또는 기술 일부 일치(갭 일부).
- **낮음**: 직무/기술 갭 큼.
이유는 어떤 경력·프로젝트·기술이 맞는지 1~2줄. **마스터에 없는 건 지어내지 않는다.**

### 8. 북마크
**높음+중간** 등급 공고를 해당 사이트의 저장/스크랩에 추가(어댑터의 "자동 북마크"). 낮음은 북마크 안 함. 클릭/요청 실패 시 "북마크 실패"로 표기하고 다음 진행.

### 9. 리포트
채팅 표 + `data/job-discovery/<날짜>-discovered.md`에 동일 저장:

| 사이트 | 회사 | 공고명 | 경력요건 | 마감일 | 매칭 | 매칭 이유 | 북마크 |
|--------|------|--------|----------|--------|------|-----------|--------|

요약 한 줄: 발견 N건 / 북마크 M건(높음 X·중간 Y) / 제외 Z건(이미 거름·중복). 사이트·쿼리당 상한을 적용했으면 잘라낸 건수를 고지(조용한 누락 금지).

→ 여기서 멈춘다. 재무 스크리닝은 사용자가 `screen-companies`를 별도 실행한다.

## 에러 핸들링
- 크롬 미연결/미로그인 → 중단·안내, 폴백 없음.
- 특정 사이트 검색 실패 → 그 사이트만 스킵, 나머지 계속.
- 자동 북마크 실패 → 해당 공고 "북마크 실패" 표기, 계속.
- alert/confirm 유발 클릭 금지. 2~3회 연속 실패 시 멈추고 보고.
- 마스터 이력서·검색 결과가 비면 사유 안내.

## 테스트 시나리오
**정상(1차)**: "원티드·점핏에서 나한테 맞는 공고 찾아 북마크해줘" → 크롬 확인 → seed 추출 → 두 사이트 검색 → 필터·중복 제거 → 매칭 등급 → 높음+중간 북마크 → 표 + `data/job-discovery/<날짜>-discovered.md` + 요약.
**에러**: 크롬 미연결 → 중단·안내.
````

- [ ] **Step 3: 검증 — 트리거·구조 재독**

`SKILL.md`를 다시 읽고 확인:
- description에 발견 트리거 문구 3종과 "발견≠스크리닝" 구분이 있는가.
- 파이프라인 1~9단계가 spec §4와 일치하는가.
- 라우팅 표가 어댑터 4개 경로를 정확히 가리키는가(파일명 오타 없음).
Expected: 모두 충족. 불일치 시 수정.

- [ ] **Step 4: Commit**

```bash
git add .claude/skills/discover-jobs/SKILL.md data/job-discovery/.gitkeep
git commit -m "feat(discover-jobs): 스킬 골격 + 4사 공통 파이프라인"
```

---

## Task 2: 원티드 어댑터 정찰 + `wanted-discovery.md`

**전제: 사용자 크롬이 원티드(wanted.co.kr)에 로그인 연결돼 있어야 함.**

**Files:**
- Create: `.claude/skills/discover-jobs/references/wanted-discovery.md`

**Interfaces:**
- Consumes: SKILL.md 파이프라인 4단계(검색)·8단계(북마크).
- Produces: 원티드 어댑터 4기능 — ① 검색(직무·경력 필터) ② 결과 파싱(공고명·회사·링크·마감일·경력·기술) ③ 자동 북마크 ④ 기존 북마크 조회.

- [ ] **Step 1: 정찰 도구 로드 + 크롬 연결 확인**

`ToolSearch`로 `mcp__claude-in-chrome__tabs_context_mcp,navigate,get_page_text,javascript_tool,read_network_requests,computer,tabs_create_mcp` 로드 → `tabs_context_mcp` 호출.
Expected: 원티드 로그인 세션 확인. 미연결이면 사용자에게 크롬 연결 요청 후 대기.

- [ ] **Step 2: 검색 경로 정찰**

새 탭에서 원티드 개발 직군 검색을 수행한다(예: 백엔드 직무 + 검색어 "Python"). 검색 중 `read_network_requests`로 검색 결과를 싣는 **API 엔드포인트**(URL·쿼리파라미터: 직무 카테고리 id, 경력 필터, 키워드, 페이지네이션)를 식별한다. 응답 JSON에서 공고별 `jobId`·공고명·회사명·회사id를 어디서 읽는지 확인한다.
Expected: 검색 API URL + 파라미터 + 응답 필드 경로를 확보.

- [ ] **Step 3: 상세 필드(마감일·경력·기술) 경로 정찰**

공고 상세 API(경로 B에서 쓰던 `https://www.wanted.co.kr/api/chaos/jobs/v4/{jobId}/details` 재사용 가능)에서 `due_time`(마감일/null=상시), 경력요건, 기술스택/자격요건 텍스트를 읽는 경로를 확인한다.
Expected: 필터(5단계: 경력·마감)와 매칭(7단계)에 필요한 필드 경로 확보.

- [ ] **Step 4: 자동 북마크 정찰 (핵심·불확실)**

공고 1건을 골라 원티드 "북마크(저장)" 동작을 수행하고, `read_network_requests`로 북마크를 거는 **API 요청**(메서드·URL·바디·인증)을 식별한다. API가 불명확하면 DOM의 북마크 버튼 셀렉터를 `computer`/`javascript_tool`로 식별해 클릭 방식을 확정한다. 직후 `/profile/bookmarks`에서 실제로 추가됐는지 확인한다.
Expected: "공고 1건 북마크 → 목록에 반영" 동작을 재현 가능한 형태로 확정. (실패 시 멈추고 사용자 보고 — 이 사이트 북마크 자동화 가능 여부가 1차 검증의 핵심.)

- [ ] **Step 5: 기존 북마크 조회 경로 확정**

경로 B reference(`screen-companies/references/wanted-bookmarks.md`)의 `/profile/bookmarks` 수집 절차를 재사용해, 중복 제거(6단계)용 "이미 북마크된 jobId 목록" 확보 방법을 reference에 링크/요약한다.
Expected: 기존 북마크 jobId 집합 확보 방법 명시.

- [ ] **Step 6: `wanted-discovery.md` 작성**

`screen-companies/references/wanted-bookmarks.md` 문서 스타일을 따라 아래 섹션으로 작성한다(각 섹션은 Step 2~5에서 **실제 확인한** URL·파라미터·필드 경로·셀렉터로 채운다):

```
# 원티드 공고 발견(검색→매칭→북마크) — 상세 절차
## 핵심 원리
## 1. 크롬 연결 확인 (전제)
## 2. 검색 실행            ← Step 2·3: 검색 API + 직무/경력 필터 + 결과·상세 필드 파싱
## 3. 자동 북마크          ← Step 4: 북마크 API 또는 버튼 클릭 방식
## 4. 기존 북마크 조회     ← Step 5: 중복 제거용(경로 B 재사용)
## 에러 핸들링
```

- [ ] **Step 7: 검증 — 단일 사이트 라이브 드라이런**

원티드만 대상으로 SKILL.md 파이프라인을 처음부터 1회 실행: seed 추출 → 검색 → 필터 → 매칭 → 높음·중간 1건 실제 북마크 → `/profile/bookmarks`에서 반영 확인.
Expected: 최소 1건이 실제로 원티드 북마크에 추가됨. 안 되면 Step 4로 돌아가 수정.

- [ ] **Step 8: Commit**

```bash
git add .claude/skills/discover-jobs/references/wanted-discovery.md
git commit -m "feat(discover-jobs): 원티드 어댑터(검색·매칭·자동 북마크) 정찰·확립"
```

---

## Task 3: 점핏 어댑터 정찰 + `jumpit-discovery.md`

**전제: 사용자 크롬이 점핏(jumpit.saramin.co.kr)에 로그인 연결돼 있어야 함.**

**Files:**
- Create: `.claude/skills/discover-jobs/references/jumpit-discovery.md`

**Interfaces:**
- Consumes: SKILL.md 파이프라인 4·8단계.
- Produces: 점핏 어댑터 4기능(검색·파싱·자동 북마크·기존 북마크 조회). 점핏은 기술스택 태그 기반 검색이 마스터 이력서와 직결.

- [ ] **Step 1: 정찰 도구 로드 + 크롬 연결 확인**

Task 2 Step 1과 동일 도구. `tabs_context_mcp`로 점핏 로그인 세션 확인.
Expected: 점핏 로그인 확인. 미연결이면 요청 후 대기.

- [ ] **Step 2: 검색 경로 정찰 (기술스택 태그)**

점핏에서 기술스택/직무 검색을 수행하고(예: "Python", "백엔드"), `read_network_requests`로 검색 결과 API(경로 B의 `jumpit-api…/api/...` 계열일 가능성)와 쿼리 파라미터(직무·기술 태그·경력 필터·size)를 식별한다. 응답에서 공고id·`companyName`·공고명·`closedAt`(마감)·경력요건·기술태그 경로를 확인한다.
Expected: 검색 API URL + 파라미터 + 응답 필드 경로 확보.

- [ ] **Step 3: 자동 북마크(스크랩) 정찰 (핵심·불확실)**

공고 1건을 스크랩하고 `read_network_requests`로 스크랩 API(메서드·URL·바디·쿠키 인증; 경로 B에서 `scrap/list`를 봤으니 추가용 엔드포인트 존재 가능)를 식별한다. 불명확하면 DOM 스크랩 버튼 셀렉터로 클릭 방식 확정. 직후 스크랩 목록에서 반영 확인.
Expected: "공고 1건 스크랩 → 목록 반영" 재현 가능 확정.

- [ ] **Step 4: 기존 스크랩 조회 경로 확정**

경로 B reference(`jumpit-bookmarks.md`)의 `scrap/list?size=100` 절차를 재사용해 중복 제거용 기존 스크랩 목록 확보 방법을 명시.
Expected: 기존 스크랩 공고 집합 확보 방법 확정.

- [ ] **Step 5: `jumpit-discovery.md` 작성**

Task 2 Step 6과 동일 섹션 구조로, Step 2~4에서 실제 확인한 값으로 작성.

- [ ] **Step 6: 검증 — 점핏 단일 라이브 드라이런**

점핏만 대상으로 파이프라인 1회 실행 → 높음·중간 1건 실제 스크랩 → 목록 반영 확인.
Expected: 최소 1건 실제 스크랩됨. 안 되면 Step 3로.

- [ ] **Step 7: Commit**

```bash
git add .claude/skills/discover-jobs/references/jumpit-discovery.md
git commit -m "feat(discover-jobs): 점핏 어댑터(검색·매칭·자동 스크랩) 정찰·확립"
```

---

## Task 4: 1차 통합 검증 + 오케스트레이터·CLAUDE.md 등록

**Files:**
- Modify: `.claude/skills/resume-orchestrator/SKILL.md`
- Modify: `CLAUDE.md`

**Interfaces:**
- Consumes: Task 1~3 산출물(스킬 + 원티드·점핏 어댑터).
- Produces: 사용자에게 노출되는 트리거(오케스트레이터 흐름 E) + CLAUDE.md 변경 이력.

- [ ] **Step 1: 1차 통합 라이브 검증 (원티드+점핏 동시)**

"원티드·점핏에서 나한테 맞는 공고 찾아 북마크해줘"로 전체 파이프라인 실행. 확인 항목:
- 두 사이트 모두 검색·필터·매칭 동작.
- 중복 제거: 과거 `data/company-screening/*.md`의 ❌회사가 결과에서 빠지는가(해당 회사가 검색에 잡힐 때).
- 높음·중간만 북마크되고 낮음은 안 되는가.
- 결과 표 + `data/job-discovery/<오늘>-discovered.md` 저장 + 요약(발견/북마크/제외 건수).
Expected: 위 전부 충족. 불충족 항목은 해당 Task로 돌아가 수정.

- [ ] **Step 2: 오케스트레이터에 흐름 E 추가**

`.claude/skills/resume-orchestrator/SKILL.md` "Phase 1 컨텍스트 확인" 분류표에 행 추가:

```
| "지원할 회사 찾아줘", "잡사이트 훑어 맞는 공고 북마크" | **E. 회사 발견** |
```

그리고 "Phase 2 흐름별 실행"에 블록 추가:

```
**E. 회사 발견**
1. 크롬 연결 확인 (없으면 안내·중단)
2. `discover-jobs` — 마스터 이력서 기반 잡사이트 탐색 → 높음·중간 매칭 공고 북마크
3. 발견/북마크/제외 요약 보고 (재무 스크리닝이 필요하면 screen-companies 별도 안내)
```

- [ ] **Step 3: CLAUDE.md 변경 이력 + 트리거 갱신**

`CLAUDE.md` 변경 이력 표에 행 추가:

```
| 2026-06-23 | 회사 발견 스킬 `discover-jobs` 신규 — 마스터 이력서에서 검색어 자동 추출→잡사이트(원티드·점핏 1차) 검색→하드필터(경력 주니어~3년·직무 백엔드/AI/풀스택·마감)→중복제거(이미 북마크+과거 ❌회사)→에이전트 매칭 등급(높음/중간/낮음)→높음·중간 북마크까지만. 재무는 screen-companies가 별도. 사람인·잡코리아는 2차 | skills/discover-jobs | "이력서 기준 지원할 회사 발견" 기능 요청 |
```

`CLAUDE.md` 트리거 문단에 발견 스킬 한 줄을 추가(이력서 관련 작업에 `discover-jobs` 포함).

- [ ] **Step 4: 검증 — 등록 정합성**

`resume-orchestrator` 분류표와 `discover-jobs` description의 트리거 문구가 충돌·중복되지 않는지, screen-companies와 구분되는지 재독.
Expected: 발견과 스크리닝이 서로 다른 흐름으로 라우팅됨.

- [ ] **Step 5: Commit**

```bash
git add .claude/skills/resume-orchestrator/SKILL.md CLAUDE.md
git commit -m "feat(discover-jobs): 오케스트레이터 흐름 E 등록 + CLAUDE.md 이력"
```

---

# Phase 2 — 2차(사람인·잡코리아): 1차 검증 후 진행

> Phase 1이 라이브로 검증된 뒤 착수한다. 골격·매칭·필터·중복·리포트는 그대로 두고 어댑터만 추가한다.

## Task 5: 사람인 어댑터 정찰 + `saramin-discovery.md`

**전제: 사용자 크롬이 사람인(saramin.co.kr)에 로그인 연결돼 있어야 함.**

**Files:**
- Create: `.claude/skills/discover-jobs/references/saramin-discovery.md`

**Interfaces:**
- Consumes: SKILL.md 파이프라인 4·8단계.
- Produces: 사람인 어댑터 4기능(검색·파싱·자동 스크랩·기존 스크랩 조회).

- [ ] **Step 1: 도구 로드 + 사람인 로그인 확인**

Task 2 Step 1 도구. `tabs_context_mcp`로 사람인 세션 확인.

- [ ] **Step 2: 검색 경로 정찰**

사람인에서 직무·키워드·경력 필터로 검색하고 `read_network_requests`/DOM으로 결과 목록(공고명·회사명·링크·마감일·경력요건) 추출 경로를 식별한다. 경로 B reference(`saramin-scraps.md`)의 `li.row 첫 앵커·span.date` 파싱 패턴을 참고한다.
Expected: 검색 결과 파싱 경로 확보.

- [ ] **Step 3: 자동 스크랩 정찰**

공고 1건 스크랩 → 스크랩 거는 요청/버튼 셀렉터 식별 → 스크랩 목록(`scrap-recruit`) 반영 확인.
Expected: 스크랩 추가 재현 가능.

- [ ] **Step 4: 기존 스크랩 조회 경로 확정**

`saramin-scraps.md`의 `scrap-recruit?page_count=100` 절차 재사용으로 중복 제거용 기존 스크랩 목록 확보 방법 명시.

- [ ] **Step 5: `saramin-discovery.md` 작성**

Task 2 Step 6 섹션 구조로 실제 확인 값으로 작성.

- [ ] **Step 6: 검증 — 사람인 단일 라이브 드라이런**

사람인만 대상 1회 실행 → 높음·중간 1건 실제 스크랩 → 반영 확인.

- [ ] **Step 7: Commit**

```bash
git add .claude/skills/discover-jobs/references/saramin-discovery.md
git commit -m "feat(discover-jobs): 사람인 어댑터 정찰·확립(2차)"
```

---

## Task 6: 잡코리아 어댑터 정찰 + `jobkorea-discovery.md`

**전제: 사용자 크롬이 잡코리아(jobkorea.co.kr)에 로그인 연결돼 있어야 함.**

**Files:**
- Create: `.claude/skills/discover-jobs/references/jobkorea-discovery.md`

**Interfaces:**
- Consumes: SKILL.md 파이프라인 4·8단계.
- Produces: 잡코리아 어댑터 4기능. 잘린 회사명 복원 주의(경로 B 기지식).

- [ ] **Step 1: 도구 로드 + 잡코리아 로그인 확인**

Task 2 Step 1 도구. `tabs_context_mcp`로 잡코리아 세션 확인.

- [ ] **Step 2: 검색 경로 정찰**

직무·키워드·경력 필터 검색 → 결과 목록 파싱(공고명·회사명·링크·마감일·경력). 회사명이 잘려 보이면 경로 B의 `Co_read/C/{id}` og:title 복원 패턴(`jobkorea-scraps.md`) 적용.
Expected: 검색 결과 파싱(회사명 복원 포함) 경로 확보.

- [ ] **Step 3: 자동 스크랩 정찰**

공고 1건 스크랩 → 요청/버튼 식별 → `/User/Scrap` 목록 반영 확인.
Expected: 스크랩 추가 재현 가능.

- [ ] **Step 4: 기존 스크랩 조회 경로 확정**

`jobkorea-scraps.md`의 `/User/Scrap?PageSize=100` 절차 재사용으로 중복 제거용 기존 스크랩 목록 확보 방법 명시.

- [ ] **Step 5: `jobkorea-discovery.md` 작성**

Task 2 Step 6 섹션 구조로 실제 확인 값으로 작성. 회사명 복원 주의 명시.

- [ ] **Step 6: 검증 — 잡코리아 단일 라이브 드라이런**

잡코리아만 대상 1회 실행 → 높음·중간 1건 실제 스크랩 → 반영 확인.

- [ ] **Step 7: Commit**

```bash
git add .claude/skills/discover-jobs/references/jobkorea-discovery.md
git commit -m "feat(discover-jobs): 잡코리아 어댑터 정찰·확립(2차)"
```

---

## Task 7: 4사 통합 검증 + 문서 갱신

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: 4사 통합 라이브 검증**

"내 잡사이트 다 훑어서 맞는 공고 북마크해줘"로 전체 실행(로그인된 사이트 전부 순회). 통합 결과 표(사이트 열 포함) + `data/job-discovery/<오늘>-discovered.md` + 통합 요약 확인. 중복 제거가 사이트 간에도 동작(같은 회사 다른 사이트)하는지 확인.
Expected: 4사 순회·통합 출력·중복 제거 정상.

- [ ] **Step 2: CLAUDE.md 변경 이력 갱신**

```
| 2026-06-23 | discover-jobs 2차 — 사람인·잡코리아 어댑터 추가, 4사 통합 발견 동작 | skills/discover-jobs/references | 1차 검증 후 확장 |
```

- [ ] **Step 3: Commit**

```bash
git add CLAUDE.md
git commit -m "feat(discover-jobs): 사람인·잡코리아 추가 후 4사 통합 검증 완료"
```

---

## Self-Review (작성자 점검 결과)

**1. Spec coverage**
- §2 확정 요구사항(seed 자동·북마크까지·4사·등급 매칭·필터·높음+중간·멈춤·중복방지·파일저장) → Task 1 SKILL.md에 전부 반영. ✅
- §3 단계적 도입(원티드·점핏 1차 → 사람인·잡코리아 2차) → Phase 1/2 분리. ✅
- §6 어댑터 4정찰 항목(검색·파싱·자동북마크·기존북마크) → Task 2·3·5·6 각 Step. ✅
- §7 출력 표·요약·상한 고지 → SKILL.md 9단계. ✅
- §8 에러 핸들링 → SKILL.md + Global Constraints. ✅
- §9 트리거·등록 → Task 4. ✅

**2. Placeholder scan**
- 어댑터 태스크의 "정찰로 확정" 표현은 플레이스홀더가 아니라 의도된 라이브 정찰(문서 상단에 명시). 그 외 TBD/TODO 없음. ✅

**3. Type consistency**
- 어댑터 4기능 명칭(검색·결과 파싱·자동 북마크/스크랩·기존 북마크/스크랩 조회)을 Task 2·3·5·6에서 동일하게 사용. SKILL.md 라우팅 표의 reference 경로와 각 Task의 Create 경로 일치. 결과 파일 경로 `data/job-discovery/<날짜>-discovered.md`로 전 구간 통일. ✅
