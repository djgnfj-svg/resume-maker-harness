# 이력서 메이커

## 하네스: 이력서 메이커

**목표:** 한 사람의 마스터 이력서(모든 경력의 원본)를 누적 구축하고, 채용공고(JD)를 받으면 그에 맞춰 선별·강조한 맞춤 이력서를 뽑아준다.

**트리거:** 이력서 수집·정리, 마스터 이력서 구축/업데이트, JD 기반 맞춤 이력서 생성, PDF 변환 등 이력서 관련 작업 요청 시 `resume-orchestrator` 스킬을 사용하라. "이력서 기준으로 지원할 회사 찾아줘"·"잡사이트 훑어 맞는 공고 북마크" 등 능동 발견 요청은 `discover-jobs`(발견≠스크리닝), 저장한 공고를 재무로 거르는 요청은 `screen-companies`를 사용하라. 단순 질문은 직접 응답 가능.

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-05-21 | 초기 구성 (에이전트 resume-maker, 스킬 ingest/build-master/tailor/export-pdf + orchestrator) | 전체 | - |
| 2026-05-21 | 출력 규칙 추가: 제목 `[회사]포지션_이름`, 기술스택 간소화(Language/Backend/Frontend/Dev+AI), 경력에 사용기술·핵심업무 소제목 구분 | skills/tailor-resume | "스킬 너무 많음, 제목·핵심업무 구분 필요" 피드백 |
| 2026-05-21 | 기술스택은 기술명만(설명 텍스트 금지), 상세 프로젝트 최대 2건+나머지 '그외 프로젝트' 한 줄 | skills/tailor-resume | "실무사용 같은 수식 빼고, 프로젝트 2건만" 피드백 |
| 2026-05-21 | 이력서 3종(위드로봇/매드업/위시스트) 추가 병합. VoicePrep·MorningBrief 프로젝트 신규, 기술스택 확장. 전문학사 2027.02 확정, reseeall.com=MorningBrief 확정 | data/master-resume.md | 마스터 누적 업데이트 |
| 2026-05-21 | AI/Agent 직무 기준 템플릿 등록 (소크라에이아이/뉴로다임/페르소나AI에서 반복된 표준 양식) | skills/tailor-resume/assets/ai-agent-resume-template.md | 동일 형식 3회 반복 → 표준화 |
| 2026-05-21 | 주요개발 불릿 괄호 주석 금지, 상세 프로젝트·경력은 각 5~7줄 유지 규칙 추가 | skills/tailor-resume | "괄호 빼고, 프로젝트+회사업무 5~7줄" 피드백 |
| 2026-05-22 | 회사 스크리닝 스킬 추가 (도메인 목록 → 캐치 조회 → 적자+매출정체/인원5명이하 기준 ✅지원/❌거름 판정 표+파일) | skills/screen-companies | "도메인 입력하면 지원할지 정리" 요청 |
| 2026-06-16 | 스크리닝 입력 경로 B 추가: 크롬 연결 시 원티드 저장 기업(북마크) 자동 수집 → 원티드 `__NEXT_DATA__`+`financial-report-for-wanted` API로 인원·매출·영업이익 조회. 상세는 references/wanted-bookmarks.md | skills/screen-companies | "크롬 연결되면 북마크 찾아주는 기능 추가" 요청 (2026-05-22 즉흥 작업을 정식 절차화) |
| 2026-06-22 | 경로 B 절차 보정: 북마크 진입을 `/profile/bookmarks`로 직접 이동 명시, 무한스크롤 끝까지 로드, companyId는 공고 상세 API(`jobs/v4/{jobId}/details`)에서 추출, **재무 API는 companyId가 아니라 regNoHash(회사페이지 `__NEXT_DATA__`) 필요**(companyId로는 404), navigate 시 window 상태 유실 → 한 페이지에서 fetch로 일괄 처리 | skills/screen-companies/references/wanted-bookmarks.md | "북마크 말고 profile/bookmarks로 바로 가게 해줘" 요청 + 실행 중 API 스펙 변경 확인 |
| 2026-06-22 | 경로 B에 **마감일 자동 수집** 추가: 공고 상세 API `due_time`(null=상시채용)을 companyId와 함께 수집, 출력에 마감일 열 + "마감 임박 공고" 섹션(오름차순·남은 일수·🔴 강조) 추가 | skills/screen-companies (SKILL.md + references/wanted-bookmarks.md) | "앞으로 공고 마감도 자동으로 수집하게 업데이트" 요청 |
| 2026-06-23 | **경로 B를 다중 잡사이트로 일반화**(원티드 외 사람인·잡코리아·점핏 추가). 사이트 라우팅 표 + 사이트별 reference 3종 신규. 새 3사는 회사명·마감일만 수집→캐치 조회로 일원화(원티드만 자체 데이터 유지). 여러 사이트 동시 요청 시 통합 파일(`-saved-jobs.md`, 출처 열). 크롬 정찰로 확립: 사람인 `scrap-recruit?page_count=100`(li.row 첫 앵커·span.date), 잡코리아 `/User/Scrap?PageSize=100`(잘린 회사명은 `Co_read/C/{id}` og:title로 복원), 점핏 `jumpit-api…/api/user/scrap/list?size=100`(쿠키 인증·companyName·closedAt) | skills/screen-companies (SKILL.md + references/{saramin-scraps,jobkorea-scraps,jumpit-bookmarks}.md) | "사람인·잡코리아·점핏도 동작하게" 요청 |
| 2026-06-23 | **회사 발견 스킬 `discover-jobs` 신규**(브레인스토밍→스펙→플랜→SDD). 마스터 이력서에서 검색어 자동 추출→잡사이트 검색→하드필터(경력 주니어~3년·직무 백엔드/AI/풀스택·마감)→중복제거(이미 북마크/스크랩+과거 ❌거른 회사)→에이전트 매칭 등급(높음/중간/낮음)→**높음·중간만 그 사이트 북마크까지** 하고 멈춤(재무는 screen-companies 별도, 발견≠스크리닝). 1차=원티드·점핏. 크롬 정찰로 확립: 원티드 검색 `search/v1/position?years=0&years=3`+상세 `jobs/v4/{id}/details`+북마크 `POST chaos/bookmarks/v1/{id}`(토글, is_bookmark=false에만), 점핏 검색 `jumpit-api/api/positions?keyword=`(scraped 플래그 포함)+스크랩 `POST .../scrap`·해제 `.../unscrap`(로그인 필수). 1차 통합 검증서 16건 실제 북마크. 사람인·잡코리아는 2차 | skills/discover-jobs (SKILL.md + references/{wanted,jumpit}-discovery.md), resume-orchestrator(흐름 E) | "이력서 기준 지원할 회사 능동 발견" 기능 요청 |
