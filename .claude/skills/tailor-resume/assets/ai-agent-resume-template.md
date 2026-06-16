# [{회사명}]{포지션명}_송영재

이름 : 송영재

**Github : https://github.com/djgnfj-svg**

**Email : djgnfj3795@gmail.com**

---

## 소개

저는 AI Agent를 활용하여 2개의 서비스를 운영하고 있는 개발자입니다. 현재는 한국로봇융합연구원에서 로봇 관제 대시보드를 개발·운영하고 있습니다.

VoicePrep 프로젝트에서 LangGraph Agent · RAG · 평가 파이프라인을 설계하고, LLM 비용 11% 절감, TTFA 64% 개선 했습니다. 그 외로 크몽 외주에서 64건의 데이터 자동화 프로젝트를 모두 만점으로 완성한 경험이 있습니다.

{회사명 + 지원동기 한 단락 — JD에 맞춰 작성}

### 기술 스택

- **Language** : Python
- **Backend** : FastAPI, DRF
- **AI/Agent** : LangGraph, LangChain, MCP
- **DB** : PostgreSQL, Redis
- **DevOps** : Docker

> 기술 스택은 JD에 맞춰 분류·항목을 조정한다. AI 직무이므로 AI/Agent를 상단에 둔다.

# 프로젝트

---

### VoicePrep | AI 음성 기술면접 코치 | 1인 프로젝트

이력서·JD 기반으로 실제 면접관처럼 질문하고 평가하는 **멀티 에이전트 서비스**

Link : https://jachana.com/

- Tech
    - AI / Agent: LangGraph, OpenAI, pgvector
    - Backend: FastAPI, SSE
    - Frontend: Next.js 15
    - Test: Playwright

### 주요 개발

- 단일 프롬프트로 질문을 일괄 생성하던 구조를 LangGraph 기반 Planner-Executor 2단계 에이전트로 재설계해 자연스러운 면접 구현
- 이력서를 4가지 청크 타입으로 분할 임베딩하고 pgvector top-3을 질문 생성에 주입해 이력서·공고 기반 개인화 질문 구현
- 면접 세션 종료 시 답변에서 강점·약점·패턴을 추출해 pgvector 기반 사용자 메모리에 누적하고 다음 세션 프롬프트에 주입해 세션 간 맥락 단절 문제 해결
- 채점을 한 번에 LLM에 맡기던 구조를 결정적 가중 합산 + 저품질 답변 guardrail로 전환해 채점 신뢰성 확보
- Prompt caching 적용을 위해 system 프롬프트의 정적/동적 영역을 분리해 1세션 LLM 비용 11% 절감 (gpt-4o-mini 기준) 및 TTS 청크 스트리밍 + 폴백 엔진으로 TTFA 64% 개선

### Resee | AI 기반 스마트 복습 자동화 플랫폼 | 1인 프로젝트

사용자가 작성한 글을 에빙하우스 망각곡선 기반으로 자동 복습할 수 있는 플랫폼

- **GitHub**: https://github.com/djgnfj-svg/Resee-project
- **Tech**
    - **Backend**: DRF, Celery, PostgreSQL, Redis
    - **AI**: LangGraph, LangChain
    - **Dev** : GitHub Actions, AWS ECS Fargate

### 주요 개발

- 복습 리스트 조회 시 발생한 N+1 쿼리로 인한 지연 문제를 Eager Loading으로 해결하여 API 응답 속도 70% 개선
- AI 분석 및 이메일 발송으로 인한 API Timeout 문제를 Celery + Redis 비동기 큐로 분리하여 즉시 응답 제공 및 백그라운드 병렬 처리 구현
- 대용량 기록 조회 시 전체 테이블 스캔으로 인한 성능 저하를 복합 인덱스 설계로 해결하여 복습 기록 페이지 응답 속도 65% 개선
- 수동 배포시 휴먼 에러 및 서비스 중단 문제를 AWS ECS Fargate + GitHub Actions CI/CD 파이프라인 구축해 무중단 배포 및 배포 시간 80% 단축
- 단일 LLM 프롬프트 호출의 낮은 답변 품질 문제를 LangGraph 기반 State Machine 워크플로우 설계로 해결하여 검색-분석-생성 다단계 파이프라인 구축
- DRF 기반 RESTful API 설계 및 Swagger 자동 문서화 | 테스트 커버리지 80% 이상 유지로 보다 안정적인 개발환경 구축

### 기타 프로젝트

- MorningBrief | 여러 LLM 페르소나가 토론·합의해 매일 아침 주식 추천 메일을 전송하는 LangGraph 멀티에이전트 서비스
    - https://www.reseeall.com/
- **Crawling 외주 :** 6개월간 64건의 외주 완수(평점 5.0)
    - **Profile**: https://kmong.com/@개발자작하

# 경력

자동화 업무로 시작해서 현재는 풀스택 개발자로 비동기 로봇 관제 대시보드를 개발, 운영하고 있습니다.

---

### 한국로봇융합연구원 | WEB Full Stack | 2026.01 ~ 재직 중

현장 실무자용 로봇 관제 시스템을 PyQt 로컬 환경에서 웹 기반으로 전환하는 프로젝트를 담당하고 있습니다.

사용기술 : FastAPI, React, Nginx

**핵심업무**

- PyQt 로컬 관제 시스템을 React + FastAPI 웹 대시보드로 전환해 원격 관제 실현
- MQTT 바이너리 데이터를 파서 기반으로 자동 변환·저장하는 실시간 데이터 수집 파이프라인 구축
- 서브도메인 기반 멀티테넌시로 단일 서버에서 복수 고객사 분리 운영
- 세션 잠금 + 하트비트 + 비활성 타임아웃으로 다수 사용자 동시 접속 시 로봇 제어 충돌 해결
- Dropbox 동기화 + systemd 파일 감시 기반 자동 배포 구축
- 포트포워딩 직접 노출 구조를 Cloudflare Tunnel + Nginx 리버스 프록시로 전환해 외부 노출 포트 제거

### Chips&Media | QA Team | 1년 (2024.07 ~ 2025.07)

반도체 IP 설계 기업에서 QA팀의 수기 검증·리포팅 업무를 Python, Shell script 기반으로 자동화했습니다.

사용 기술 : Python, Shell

**핵심 업무**

- 검증 내역 12배 증가로 9시간 걸리던 Python script 동작을 I/O bound 부분 멀티스레딩으로 전환해 처리 속도 95% 개선
- SVN 리비전 트레킹 Shell Script를 순차 탐색에서 이진 탐색으로 개선해 성능 40% 향상
- 테스트 결과 수기 취합하는 기존 pipeline에 비주얼 라이징을 추가해 자동 리포트를 구현해 엔지니어 리소스 주당 30시간 절감, 휴먼 에러 제거
- 제품별 체크리스트 수기 작성 작업을 Jenkins pipeline에 검증 로직 추가해 처리 시간 80% 단축(4시간 →30분) & 휴먼 에러 감소
- WIKI에 올라오는 Error report 과정을 Confluence API와 Python script를 활용해 자동전달 구현, 휴먼 에러 제거 & 일일 작업 2시간 절감

---

## 교육

컴퓨터네트워크과(학점 은행제) | 2025.11 ~ 진행 중(2027.02 전문학사 취득 예정)

**42 Seoul** | Common Core (2022.01 - 2023.01) - C언어 기반 시스템 프로그래밍 및 자료구조, 알고리즘 학습
