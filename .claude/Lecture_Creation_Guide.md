# Lecture Creation Guide - 강의 제작 파이프라인 가이드

Lecture Factory 프로젝트의 전체 강의 제작 워크플로우를 정의합니다.

## 전체 파이프라인 개요

```
/lecture-outline  →  /lecture-script   →  /slide-planning    →  /slide-generation
  강의구성안             강의교안            슬라이드 기획         슬라이드 생성 프롬프트
  (7단계)               (7단계)             (5단계)              (3단계)
```

각 워크플로우는 독립 실행 가능하며, 이전 단계의 산출물을 입력으로 참조합니다.

---

## 아키텍처

### Skill 오케스트레이터 패턴

Skill이 직접 오케스트레이터 역할을 하며, `Agent` 도구로 공통 에이전트를 순차 호출합니다.

```
/lecture-outline (Skill = 오케스트레이터)
     ├── Phase 1: Agent → input-agent
     ├── Phase 2: Agent → research-agent (탐색적 리서치)
     ├── Phase 3: Agent → brainstorm-agent
     ├── Phase 4: Agent → research-agent (심화 리서치)
     ├── Phase 5: Agent → architecture-agent
     ├── Phase 6: Agent → writer-agent
     └── Phase 7: Agent → review-agent
```

- Skill → Agent 호출 (허용)
- Agent → Agent 중첩 (금지)

### 6개 공통 에이전트

| 에이전트 | 역할 | 사용 워크플로우 |
|---------|------|--------------|
| **input-agent** | 사용자 입력 수집 + 이전 산출물 로드 + 컨텍스트 구성 | 구성안, 교안, 슬기획, 슬생성 |
| **brainstorm-agent** | 아이디어 확장, 구체화, 우선순위 분류 | 구성안, 교안, 슬기획 |
| **research-agent** | 인터넷 리서치, 트렌드 분석, 자료 수집 (2-Pass: 탐색적 + 심화) | 구성안, 교안 |
| **architecture-agent** | 구조 설계, 정렬 맵, Backward Design 적용 | 구성안, 교안, 슬기획 |
| **writer-agent** | 최종 문서/콘텐츠 생성 (템플릿 기반) | 구성안, 교안, 슬기획, 슬생성 |
| **review-agent** | 품질 검증, 체크리스트 평가, 피드백 생성 | 구성안, 교안, 슬기획, 슬생성 |

에이전트는 Skill이 전달하는 컨텍스트(지시사항 + 템플릿)에 따라 워크플로우별로 다르게 동작합니다.

---

## 워크플로우 상세

### 워크플로우 1: 강의구성안 (`/lecture-outline`)

**목적**: 강의의 전체 설계도를 작성합니다.

| Phase | 단계 | 에이전트 | 핵심 작업 |
|-------|------|---------|----------|
| 1 | 입력 수집 | input-agent | 필수 6개(주제, 학습자, 목표, 형태, 시간, 키워드) + 선택 7개 → input_data.json |
| 2 | 탐색적 리서치 | research-agent | 참고자료 분석(로컬 폴더 + NotebookLM) + 트렌드, 유사 강의 (문제 공간 이해) |
| 3 | 브레인스토밍 | brainstorm-agent | 6가지 발산 기법(교차영역 비유, 가정 역전, 규모 전환, 제약 변경, 학제간 융합, SCAMPER) → Bloom's 매핑 → 3역할 검증(회의론자·학습자대리인·중재자) |
| 4 | 심화 리서치 | research-agent | 브레인스토밍 결과 기반 사례 수집, 참고자료 심화 분석, 참고문헌, 보충 콘텐츠 |
| 5 | 아키텍처 설계 | architecture-agent | Backward Design(학습결과→평가→학습경험), 4단계 매크로 구조(A환경/B핵심/C심화/D통합), 정렬 맵, 시간 배분, 3중 검증(정렬·시간·인지부하) |
| 6 | 구성안 작성 | writer-agent | outline-template.md 기반, GAIDE 5단계 + BOPPPS + 자동 분할 작성 + §시간표 생성 |
| 7 | 품질 검토 | review-agent | QM Rubric 8기준 + 5영역 가중치 체크리스트(25/25/15/15/20), 4단계 판정 |

**적용 프레임워크**: Backward Design + GAIDE + BOPPPS + 2-Pass Research + 프롬프트 체이닝

#### Phase 1: 입력 수집 상세

**필수 질문 (6개)** — 답변 필수, 없으면 진행 불가

| # | 카테고리 | 질문 | 입력 형태 |
|---|---------|------|----------|
| Q1 | 핵심 주제 | 무엇을 가르치나요? | 자유 텍스트 |
| Q2 | 대상 학습자 | 누구를 가르치나요? | 자유 텍스트 + 수준(입문/중급/고급) |
| Q3 | 학습 목표 | 강의 후 학습자가 무엇을 할 수 있어야 하나요? | 자유 텍스트 (측정 가능 행동 동사) |
| Q4 | 강의 형태 | 어떤 형태로 진행하나요? | 선택지 (기본값: 강의/오프라인) |
| Q5 | 시간·차시 | 시간 구성은? | 프리셋 (기본값: 집중 워크숍 5일×8h, 50분+10분) |
| Q6 | 핵심 키워드 | 반드시 다뤄야 할 주제나 기술은? | 자유 텍스트 |

**선택 질문 (9개)** — 기본값 존재, 미입력 시 자동 적용

| # | 카테고리 | 기본값 |
|---|---------|--------|
| Q7 | 선수 지식 | 없음 (초급부터) |
| Q8 | 제외 범위 | 없음 |
| Q9 | 평가 방식 | 형성평가 (퀴즈+실습) |
| Q10a | 교수 전략 | PBL + AI-first, 실습 50%+ |
| Q10b | 톤·스타일 | 비유 중심 설명 + 메타포 목록 |
| Q11 | 참고 자료 | 없음 (로컬 폴더 경로 / NotebookLM URL, 수집만) |
| Q12 | 맥락 | 독립 강의 |
| Q13 | 산출 범위 | 없음 (강의 산출물 범위 정의) |
| Q14 | 실습 환경 | 없음 (OS, 도구, 런타임, 버전 등 환경 제약) |

Q11 참고 자료는 **경로/URL만 수집**하며, 실제 분석은 Phase 2 탐색적 리서치에서 수행:
- 로컬 폴더 → research-agent가 Glob+Read로 스캔·분석
- NotebookLM → research-agent가 NBLM 스킬로 소스 쿼리

스키마: `.claude/templates/input-schema.json` 참조

#### 2-Pass Research 설계 근거

- ADDIE, SAM, Double Diamond 등 모든 주요 교수설계 프레임워크가 분석/리서치를 아이디어 생성 이전에 배치
- Minas et al.(2018) — 사전 프라이밍이 아이디어의 수량, 참신성, 실현가능성, 관련성을 동시에 향상
- IdeaSynth(2024) — 2-pass 구조에서 대안적 아이디어 탐색이 유의미하게 증가 (5.40 vs 3.65)
- 1차 리서치는 **방향 제시형(orientation)** — 문제 공간 이해 수준으로 제한하여 고착 효과(fixation) 방지
- 2차 리서치는 **심화형(deep dive)** — 브레인스토밍에서 구체화된 아이디어를 검증하고 자료 보강

#### Phase 2 탐색적 리서치 — web-research 패턴 적용

- 패턴 출처: `langchain-ai/deepagents@web-research` (참조용 설치)
- 5자료원 통합: 01_input_data.json + 로컬 참고자료 + NotebookLM + Context7 공식문서(조건부) + 인터넷 리서치
- 5단계 통합 알고리즘: 주제 축 추출 → 축별 배정 → 교차검증(삼각측량) → 고착효과 필터 → 구조화 작성
- 상세 워크플로우: `.claude/agents/research-agent/AGENT.md` 참조

#### Phase 4 심화 리서치 — deep-research 스킬 적용

- 스킬: `199-biotechnologies/claude-deep-research-skill@deep-research` (설치 완료, `--copy`)
- research-agent가 `.claude/skills/deep-research/SKILL.md`의 8단계 파이프라인을 방법론 레퍼런스로 따름
- 8단계: Scope → Plan → Retrieve → Triangulate → Outline Refinement → Synthesize → Critique → Refine → Package
- 브레인스토밍 결과(`03_brainstorm_result.md` §8)를 입력으로 구체적 사례·참고문헌·검증 데이터 수집
- 인용 기반 보고서 생성 (anti-hallucination protocol: 모든 팩트에 즉시 인용 `[N]` 부여)
- 비판적 자기검증 (Critique): 수집 편향, 최신성, 다양성, 커버리지 점검
- 검증 스크립트: `verify_citations.py`, `validate_report.py` 활용
- 모드 선택: Standard(5-10분, 기본) 또는 Deep(10-20분, 철저 검증)
- 상세 워크플로우: `.claude/agents/research-agent/AGENT.md`의 "강의구성안 심화 리서치" 섹션 참조

**데이터 흐름**:
```
사용자 입력 → 01_input_data.json → 02_explore_research.md → 03_brainstorm_result.md
→ 04_deep_research.md → 05_arch_architecture.md → 06_write_lecture_outline.md → 07_review_quality.md
```

**산출물**: `lectures/YYYY-MM-DD_{강의명}/01_outline/06_write_lecture_outline.md`

#### Phase 5: 아키텍처 설계 상세

**4단계 매크로 구조 (Compositional Stacking)**:

```
Phase A: 환경/기초 정립 ─── ~15%  │ 이론:실습 = 70:30  │ I Do 중심
Phase B: 핵심 개념 습득 ─── ~35%  │ 이론:실습 = 50:50  │ I Do → We Do
Phase C: 심화 실습 ─────── ~30%  │ 이론:실습 = 30:70  │ We Do → You Do
Phase D: 통합/산출물 ────── ~20%  │ 이론:실습 = 10:90  │ You Do 중심
```

비율은 `schedule.days`, `pedagogy`, `target_learner.level`에 따라 보정.

**3중 검증** (`05_arch_architecture.md` 완성 전 필수):

| 검증 | 기준 | FAIL 시 처리 |
|------|------|------------|
| 정렬 검증 | 모든 학습 목표가 활동 + 평가에 매핑 | 빈 행에 활동/평가 추가 |
| 시간 예산 검증 | 배정 ≤ 가용×1.05, 배정 ≥ 가용×0.85 | 주제 보충자료 전환 또는 실습 확대 |
| 인지 부하 검증 | 차시당 개념 ≤ 5, 연속 설명 ≤ 20분 | 차시 분할 또는 중간 활동 삽입 |

#### Phase 6: 구성안 작성 상세

**GAIDE 5단계 작성**: Setup → Rough Draft → Macro Refinement → Micro Refinement → Consolidation

**BOPPPS 차시 내부 구조** (50분 수업 + 10분 휴식 = 60분):

```
B  Bridge In        5분   │ Hook, 동기유발
O  Outcomes         2분   │ 학습 목표 제시
P  Pre-assessment   5분   │ 선수지식 확인
P  Participatory   25분   │ 핵심 학습 활동 (50% 보호)
P  Post-assessment  5분   │ 이해도 확인
S  Summary          5분   │ 핵심 정리, 차시 예고
── Break           10분   │ 휴식
```

**분할 작업 (Chunk Writing)** — 대규모 강의 자동 분할:

writer-agent가 총 교시 수에 따라 단일/분할 작성을 자동 판단합니다.

| 총 교시 수 | 작성 모드 | 동작 |
|-----------|---------|------|
| ≤ 20교시 | **단일 패스** | 전체 구성안을 한 번에 작성 |
| 21~40교시 | **2분할** | Day 그룹 2개로 나눠 순차 작성 → 병합 |
| 41교시+ | **3분할** | Day 그룹 3개로 나눠 순차 작성 → 병합 |

분할 작성 절차:
1. `05_arch_architecture.md`의 매크로 구조(Phase A~D)를 기준으로 Day 그룹 경계 결정
2. 각 청크(chunk)별로 GAIDE 5단계를 독립 실행 (이전 청크의 마지막 차시를 컨텍스트로 전달)
3. 전체 청크 작성 완료 후 단일 `06_write_lecture_outline.md`로 병합 (중간 청크: `06_write_outline_chunk_N.md`)
4. 병합 후 §시간표, 정렬 매트릭스, 부록 등 코스 레벨 섹션을 통합 정제

**시간표 섹션 (§시간표)** — outline 내부 포함:

writer-agent가 `06_write_lecture_outline.md` 내부에 `§시간표` 섹션을 생성합니다. 일별 × 교시별 그리드 형태로 전체 일정을 한눈에 조망합니다.

```
§시간표 형식 예시:

| 일차 | 교시 | 시간 | 주제 | 핵심 활동 | Phase |
|------|------|------|------|---------|-------|
| Day 1 | 1교시 | 09:00-09:50 | 환경 설정 | Colab GPU 실습 | A |
| Day 1 | 2교시 | 10:00-10:50 | 딥러닝 개요 | 개념 설명 + 퀴즈 | A |
| ... | ... | ... | ... | ... | ... |
```

- 배치 위치: §5 로드맵 바로 앞 (§4-3과 §5 사이)
- 시간대: `01_input_data.json`의 `schedule` 기반 자동 산출
- Phase 열: 매크로 구조(A/B/C/D) 표기로 전체 흐름 시각화

---

### 워크플로우 2: 강의교안 (`/lecture-script`)

**목적**: 구성안을 기반으로 실제 강의에 사용하는 상세 교안을 작성합니다.

| Phase | 단계 | 에이전트 | 핵심 작업 |
|-------|------|---------|----------|
| 1 | 입력 수집 | input-agent | 구성안 자동 로드 + 교수 모델·활동 전략·형성평가·시간 비율 등 6개 필수질문 수집 |
| 2 | 탐색적 리서치 | research-agent | 교수법 사례, Gagne/Hunter 적용 패턴, 발문·실습·형성평가 도구 사례 탐색 |
| 3 | 브레인스토밍 | brainstorm-agent | 조건부 3~5축 발산(발문·활동·사례·PBL시나리오·평가), Bloom's 발문 배치, Gagne-GRR 매핑, SLO 정렬, 페르소나 시뮬레이션, 다관점 검증 |
| 4 | 심화 리서치 | research-agent | 브레인스토밍 §11 기반 4대 영역(교수법 사례·발문 뱅크·활동 자료·형성평가 도구) 심화 수집, SLO-활동-평가 삼각 정렬 검증 |
| 5 | 교안 구조 설계 | architecture-agent | 레슨 레벨 Backward Design, 교수 모델별 도입-전개-정리 구조, Gagné 사태·발문·GRR 배치, 분 단위 시간 설계, 3중 검증(시간합산+SLO정렬+Gagné순차) |
| 6a | 차시별 교안 작성 | writer-agent | 1차시 단위 순차 작성, Running Summary 기반 일관성 유지, script-template.md 차시 형식 준수 |
| 6b | 차시별 경량 검토 | review-agent | 시간합산·SLO커버리지·Bloom's점진 3항목 즉시 검증, FAIL 시 1회 재작성 |
| 6c | 4시간 모듈 통합 | writer-agent | 반일(4시간) 차시 병합, 일관성 패치, Synthesizer 삽입, Running Summary 리셋 |
| 6d | 모듈 수준 검토 | review-agent | 차시 간 연결성·톤 일관성·난이도 점진·모듈 SLO 커버리지 4항목 검증 |
| 7a | 최종 전체 통합 | writer-agent | 모든 모듈 병합 → 최종 교안, 코스 레벨 섹션(메타·시간표·정렬 매트릭스·부록) 추가 |
| 7b | 최종 품질 검토 | review-agent | QM Rubric 기준 3·5 중심, 5영역 25항목 전체 검토, 교차 모듈 정합성 집중, 4단계 판정 |

**적용 프레임워크**:
- Madeline Hunter 6단계 (직접교수법)
- PBL 6단계 (문제기반학습)
- Before/During/After (플립러닝)
- Gagne의 9가지 수업사태 (전체 9사태 / 핵심 5사태 / 라벨 없음 선택 가능)
- Gradual Release of Responsibility (I Do → We Do → You Do)

#### Phase 1: 입력 수집 상세

**구성안 자동 로드 (사용자 개입 없음)**:
`01_outline/06_write_lecture_outline.md` + `01_outline/01_input_data.json`에서 14개 항목 자동 파싱
(topic, target_learner, format, schedule, pedagogy, tone, learning_goals, essential_questions, assessment, course_structure, sessions, prerequisites, exclusions, keywords)

**필수 질문 (6개)** — AskUserQuestion 2회 묶음 호출 (3+3)

| # | 카테고리 | 질문 | 입력 형태 |
|---|---------|------|----------|
| SQ1 | 교안 작성 범위 | 전체 차시 vs 특정 차시 선택 | 선택지 (기본: 전체) |
| SQ1a | 교수 모델 | 직접교수법 / PBL / 플립러닝 / 혼합 | 선택지 (pedagogy에서 자동 추론 후 추천) |
| SQ1b | 활동 전략 | 개인실습 / 그룹활동 / 토론·발문 / 프로젝트 | 복수 선택 (pedagogy에서 자동 추론) |
| SQ2 | 스크립트 상세도 | 완전 스크립트 / 반구조화 / 불릿 노트 | 선택지 (기본: 반구조화) |
| SQ3 | 형성평가 유형 | 섹션별 체크 / Exit Ticket / 실습 통합 / 없음 | 선택지 (기본: 섹션별 체크) |
| SQ4 | 시간 비율 | 교수 모델 기반 자동 / 직접 입력 | 선택지 (기본: 자동) |

**선택 질문 (3개)** — 게이팅 방식. "기본값으로 진행" 선택 시 전체 스킵

| # | 카테고리 | 기본값 |
|---|---------|--------|
| SQ5 | Gagne 9사태 적용 수준 | 핵심 5사태만 (1·2·3·6·9) |
| SQ6 | 발문 설계 포함 여부 | 포함 — 발문만 (예상 답변 제외), 차시당 4개 |
| SQ7 | 실습 가이드 상세도 | 단계 목록 + 핵심 지시 |

**교수 모델 자동 추론**: `pedagogy` 텍스트에서 키워드 기반 추론 → SQ1a 추천값 생성
- "PBL", "프로젝트 기반" → `pbl` / "플립", "flipped" → `flipped` / "직접교수", "강의식" → `direct_instruction`
- 매칭 없음 → `direct_instruction` (low confidence, 경고 기록)

스키마: `.claude/temp/design_script_phase1.md` B섹션 참조

#### Phase 2 탐색적 리서치 — 교수법 중심 web-research 패턴 적용

- 강의구성안 Phase 2와 동일 골격 (Step 0~4) 재활용, 리서치 초점을 교수법으로 전환
- 6자료원 통합: 01_input_data.json + 구성안 02_explore_research.md(상속) + 로컬 참고자료 + NotebookLM + Context7 공식문서(조건부) + 인터넷 리서치
- 구성안에서 이미 수집된 주제 개요·트렌드·학습자 분석은 **상속** (중복 수집 방지)
- 교수 모델(`teaching_model`)에 따라 리서치 질문 분기: 직접교수법(Hunter) / PBL(시나리오) / 플립러닝(Before-During) / 혼합
- Gagne 사태 적용 사례를 공통 리서치 질문에 포함, `gagne_display` 값(전체 9/핵심 5/라벨 없음)에 따라 탐색 깊이 분기
- 5축 통합 알고리즘 (교안 버전): 교수법 패턴 → 발문 설계 → 학습활동 → 형성평가 도구 → 실생활 사례
- 서브토픽 검색 예산 조건부 조정: `questioning_design` "제외" 시 발문 예산 축소→활동·평가 재배분, `gagne_display` "라벨 없음" 시 Gagne 검색 생략
- 고착 효과 필터 (교안 버전): 다른 강의의 교안 스크립트 전사 금지, 교수법 패턴·활동 유형·발문 패턴은 유지
- §7 리서치 인사이트에 Bloom's 수준 기초 마킹 추가 (Phase 3 발문 설계 브레인스토밍용)
- 상세 워크플로우: `.claude/agents/research-agent/AGENT.md`의 "강의교안 탐색적 리서치" 섹션 참조

#### Phase 3: 브레인스토밍 상세

**핵심 전환**: 구성안 "무엇을 가르칠까" → 교안 **"어떻게 가르칠까"**

**입력 4개**: `02_script/01_input_data.json` + `02_script/02_explore_research.md` + `01_outline/03_brainstorm_result.md`(페르소나 상속) + `01_outline/06_write_lecture_outline.md`(차시·SLO)

**조건부 브레인스토밍 축 (3~5개)**:

| 축 | 내용 | 생성 조건 |
|----|------|----------|
| 1 | **발문 설계** — Bloom's L1~L6별 발문 풀 | `questioning_design.include = true` |
| 2 | **학습활동 아이디어** — GRR×Gagne 기반 | 항상 |
| 3 | **실생활 사례 풀** — 다중 맥락 탐색 | 항상 |
| 4 | **문제 시나리오 설계** — PBL 문제 상황 | `teaching_model = pbl` |
| 5 | **형성평가 문항 풀** — SLO 매핑 기반 | `formative_assessment ≠ none` |

**5단계 워크플로우**: Step 0(맥락 분석·조건부 축 결정) → Step 1(발산 — 축별 15~25개 아이디어) → Step 2(수렴 — Bloom's 배치, Gagne-GRR 매핑, SLO 정렬, 페르소나 시뮬레이션) → Step 3(다관점 검증 — 교수법 회의론자·학습자 대리인·활동 정렬 중재자) → Step 4(최종 통합 — 11개 섹션)

**핵심 수렴 매트릭스 3개**:
1. **발문 수업 단계 배치** — 교수 모델별 Bloom's 패턴에 따라 도입/전개/정리 배치
2. **활동-Gagne-GRR 매핑** — Gagne 사태 × GRR 단계 교차 매트릭스
3. **SLO-활동-발문-평가 정렬** — 모든 SLO에 활동+평가 커버리지 확인

**산출물**: `03_brainstorm_divergent.md` → `03_brainstorm_convergent.md` → `03_brainstorm_review.md` → `03_brainstorm_result.md` (★ 최종)

상세 워크플로우: `.claude/agents/brainstorm-agent/AGENT.md`의 "강의교안 브레인스토밍 (Phase 3) 세부 워크플로우" 섹션 참조

#### Phase 4 심화 리서치 — deep-research 스킬 적용 (교안 맥락)

- 스킬: `199-biotechnologies/claude-deep-research-skill@deep-research` (설치 완료, `--copy`)
- research-agent가 `.claude/skills/deep-research/SKILL.md`의 8단계 파이프라인을 교안 설계 맥락에 적용
- 8단계: Scope → Plan → Retrieve → Triangulate → Outline Refinement → Synthesize → Critique → Refine → Package
- 브레인스토밍 결과(`03_brainstorm_result.md` §11)를 입력으로 4대 영역 심화 수집:
  - **A. 교수법 적용 사례**: 교수 모델(Hunter/PBL/플립러닝)별 레슨 플랜 패턴, Gagne 사태별 구현, GRR 단계별 활동
  - **B. 발문 뱅크**: Bloom's 수준별(L1~L6) 발문 예시, 교수 모델별 단계별 발문 배치
  - **C. 활동/보충 자료**: 실습 가이드, 워크시트, 루브릭, 실생활 사례
  - **D. 형성평가 도구**: SLO별 매핑된 평가 도구, 교수 모델 × 평가 유형별 추천
- 교안 특화 검증 3기준: 교수 모델 정합성, SLO-활동-평가 삼각 정렬(Bloom's 수준 일치), 인지 부하 적정성
- 비판적 자기검증 (Critique): 교수법 편향, 실용성, Bloom's 균형, SLO 커버리지, 학습자 접근성
- 구성안 심화 리서치(`01_outline/04_deep_research.md`)의 CK 결과를 상속하고, PCK(교수학적 내용 지식) 수집에 집중
- 조건부 예산 조정: `gagne_display`, `questioning_design`, `formative_assessment`, `teaching_model`에 따라 검색 예산·범위 동적 조정
- 고착 효과 필터 (교안 심화 버전): 타강의 교안 스크립트 전사 금지, 교수법 패턴·활동 유형·발문 패턴은 허용
- 검증 스크립트: `verify_citations.py`, `validate_report.py` 활용
- 모드 선택: Standard(5-10분, 기본) 또는 Deep(10-20분, 철저 검증)
- 상세 워크플로우: `.claude/agents/research-agent/AGENT.md`의 "강의교안 심화 리서치" 섹션 참조

#### 교수 모델별 교안 구조

| 교수 모델 | 주 모델 | 도입 | 전개 | 정리 | GRR 중심 |
|-----------|---------|------|------|------|---------|
| **직접교수법** | Hunter 6단계 | 10% (복습→목표제시) | 60% (제시→시범→안내연습→독립연습) | 30% (피드백→복습→차시예고) | I Do → We Do → You Do |
| **PBL** | PBL 6단계 | 10% (문제 시나리오→목표 연결) | 75% (문제정의→탐구→해결책개발) | 15% (발표→성찰→동료평가) | You Do Together 중심 |
| **플립러닝** | Before/During/After | 5% (사전학습 확인→핵심 보완) | 80% (개념명확화→그룹활동→심화적용) | 15% (피드백→사후과제안내) | We Do + You Do Together |
| **혼합** | 차시별 조합 | 10% | 70% | 20% | 차시별 개별 |

**교수 모델별 Bloom's 발문 수준**:

| 수업 단계 | 직접교수법 | PBL | 플립러닝 |
|----------|-----------|-----|---------|
| 도입 | L1~L2 (기억, 이해) | L4~L5 (분석, 평가) | L2~L3 (이해, 적용) |
| 전개 초반 | L2~L3 | L3~L4 | L3~L4 |
| 전개 후반 | L3~L4 | L5~L6 | L4~L5 |
| 정리 | L2~L3 | L5~L6 | L5 |

**형성평가 × 교수 모델 추천 도구**:

| 형성평가 유형 | 직접교수법 | PBL | 플립러닝 |
|-------------|-----------|-----|---------|
| 섹션별 체크 | 퀴즈, 화이트보드 응답 | 진행 발표, 동료 피드백 | 개념 적용 미니 과제 |
| Exit Ticket | 3-2-1 반성, 1분 작문 | 성찰 일지 | "사전학습과 연결된 새 발견" |
| 실습 통합 | 수행 체크리스트, 코드 리뷰 | 루브릭 기반 산출물 평가 | 실습 결과 동료 검토 |

#### Phase 5: 교안 구조 설계 상세

**핵심 전환**: 구성안 Phase 5가 "어떤 차시에 무엇을 가르칠까"(코스 레벨)를 설계했다면, 교안 Phase 5는 **"각 차시 안에서 어떻게 가르칠까"**(레슨 레벨)를 분 단위로 상세화한다.

**Compositional Stacking 상속**: 구성안의 Stage 1(학습목표)·Stage 2(평가 프레임)는 그대로 상속하고, Stage 3 내부만 교수 모델별 차시 구조로 확장.

- 4단계 워크플로우: Step 0(컨텍스트 로드+교수 모델 결정) → Step 1(레슨 레벨 Backward Design: SLO 정제, 형성평가 매핑, 활동 선택) → Step 2(차시별 내부 구조: 도입-전개-정리, Gagné, 발문, GRR, 분 단위 시간) → Step 3(3중 검증+산출물 작성)
- 교수 모델별 차시 내부 구조 템플릿: 직접교수법(Hunter+GRR), PBL(6단계), 플립러닝(Before/During/After)
- Gagné 사태 3모드 배치: `all_9`(9사태 전부) / `core_5`(핵심 5사태: 1·2·3·6·9) / `none`(라벨 생략)
- GRR Phase별 비율 자동 배치: Phase A(I Do 70%) → B(40/40/20) → C(You Do 70%) → D(You Do 90%)
- Bloom's 기반 발문 배치: 교수 모델 × 수업 단계별 패턴에 따라 차시당 N개 배분
- 시간 비율 자동 산출: 교수 모델별 기본값(직접교수법 10/60/30, PBL 10/75/15, 플립러닝 5/80/15) + Phase별 보정(A: 도입+5%, D: 정리-5%/전개+5%)
- 조건부 분기: `gagne_display.mode`, `questioning_design.include`, `formative_assessment.type`, `time_ratio.source`, `teaching_model`에 따라 설계 범위·검증 항목 동적 조정

**3중 검증** (`05_arch_lesson_plan.md` 완성 전 필수):

| 검증 | 기준 | FAIL 시 처리 |
|------|------|------------|
| 시간 합산 | 차시별 하위 단계 합 = session_minutes (±1분), 전체 도입:전개:정리 비율 오차 5% 이내 | 유연한 활동에서 시간 조정 |
| SLO 정렬 | SLO 완전 커버리지, Bloom's 일치, 형성평가 커버리지, 활동-평가 정합 | 미배치 SLO에 활동/평가 추가 |
| Gagné 순차 | 배치 순서 준수, 누락 없음, 각 사태 ≥1분 | 사태 재배치 또는 시간 재배분 |

- 상세 워크플로우: `.claude/agents/architecture-agent/AGENT.md`의 "강의교안 레슨 플랜 설계 (Phase 5) 세부 워크플로우" 섹션 참조

#### Phase 6: 차시별 반복 작성 + 4시간 모듈 통합 (3계층 아키텍처)

**핵심 전환**: Phase 5가 "각 차시 안에서 어떻게 가르칠까"를 분 단위로 설계했다면, Phase 6은 **"교수자가 실제로 무엇을 말하고, 어떤 질문을 던지고, 어떤 활동을 진행하는가"**를 스크립트 수준으로 구체화한다.

**설계 근거**: SAM(반복적 교수설계), LLM Hierarchical Expansion(계층적 생성), AWS Evaluate-Reflect-Refine 패턴, 교수설계 Macro-Meso-Micro 3단계 레이어

**3계층 구조 (Macro-Meso-Micro)**:

```
Macro (전체 과정) ── Phase 7a: 최종 통합 + Phase 7b: 전체 검토
  └── Meso (4시간 모듈 = 반일) ── Phase 6c: 모듈 통합 + Phase 6d: 모듈 검토
        └── Micro (차시 = 50분) ── Phase 6a: 차시 작성 + Phase 6b: 차시 검토
```

**모듈 경계 결정 로직**:

```
sessions_per_day = floor(hours_per_day × 60 / (session_minutes + break_minutes))
sessions_per_module = ceil(sessions_per_day / 2)  # 반일(4시간) 단위

예시 (88교시, 11일, 8h/day, 50분+10분):
  sessions_per_day = 8, sessions_per_module = 4
  → 22개 모듈 (Day 1 오전 4교시 / Day 1 오후 4교시 / ...)
```

Phase A→B 등 매크로 구조 전환점이 모듈 중간에 발생하면, 해당 전환점을 모듈 경계로 보정한다.

##### Phase 6a: 차시별 교안 작성 (writer-agent, session_write 모드)

**입력**:
- `05_arch_lesson_plan.md` 해당 차시 섹션 (분 단위 레슨 플랜)
- `01_input_data.json` (script_settings)
- `03_brainstorm_result.md` 관련 섹션 (발문, 활동, 사례)
- `04_deep_research.md` 관련 섹션
- `_running_summary.md` (이전 차시 누적 요약)
- 직전 차시 파일 `06_session_{N-1}.md` (전환 멘트 연결용)

**산출물**: `06_sessions/06_session_{NNN}.md` (예: `06_session_001.md`)

**작성 범위**: 정확히 1개 차시. script-template.md의 차시 섹션 형식 준수.

**Running Summary 규칙**:
- 차시 작성 완료 시 핵심 내용 2~3문장을 `_running_summary.md`에 append
- 항목: (1) 다룬 핵심 개념, (2) 마지막 전환 멘트/예고, (3) 학습자 산출물
- 모듈 통합(6c) 시 모듈 수준 요약으로 리셋 (누적 오차 방지)

##### Phase 6b: 차시별 경량 검토 (review-agent, session_review 모드)

**검토 항목 (3개)**:

| 항목 | 기준 | FAIL 시 처리 |
|------|------|------------|
| 시간 합산 | 도입+전개+정리 하위단계 합 = session_minutes (±1분) | writer-agent 1회 재작성 |
| SLO 커버리지 | 해당 차시 SLO 전체가 전개 활동에 매핑 | writer-agent 1회 재작성 |
| Bloom's 점진 | 도입(저)→전개(중)→정리(중~고) 패턴 | writer-agent 1회 재작성 |

**재시도 규칙**: FAIL→writer 1회 재호출→2차 FAIL→경고 기록 후 진행 (모듈 검토에서 처리). 최대 재시도 1회.

##### Phase 6c: 4시간 모듈 통합 (writer-agent, module_integrate 모드)

**발동 조건**: 해당 모듈의 모든 차시가 6a+6b를 통과한 후 자동 발동

**입력**:
- 해당 모듈의 `06_session_{N}.md` 파일들 (4교시 기준 4개)
- `05_arch_lesson_plan.md` 해당 Day/모듈 섹션
- `_running_summary.md`
- 이전 모듈 통합본 `06_module_{K-1}.md` (모듈 간 연결용)

**작업 내용**:
1. **병합**: 차시 파일들을 단일 `06_module_{NN}.md`로 병합
2. **일관성 패치**: 용어 통일, 전환 멘트 매끄럽게 다듬기, tone_examples 배분 확인
3. **Synthesizer 삽입**: 모듈 말미에 "이 4시간에서 배운 핵심" 종합 요약 (분절 학습 방지)
4. **Running Summary 리셋**: 모듈 수준 요약으로 갱신 (이전 모듈 1~2문장 + 현재 모듈 상세)

**산출물**: `06_modules/06_module_{NN}.md` (예: `06_module_01.md`)

##### Phase 6d: 모듈 수준 검토 (review-agent, module_review 모드)

**검토 항목 (4개)**:

| 항목 | 기준 |
|------|------|
| 차시 간 연결성 | 전환 멘트 존재, 산출물 연쇄, 이전 차시 참조 |
| 톤 일관성 | 모듈 내 문체/용어 통일, tone_examples 배치 |
| 난이도 점진 | 모듈 내 Bloom's 수준 상승 패턴 |
| 모듈 SLO 커버리지 | 해당 반일의 SLO 전체가 활동+평가에 커버 |

**FAIL 시**: writer-agent 모듈 수정 모드로 1회 재호출. 2차 FAIL 시 경고 기록 후 진행.

**산출물**: `06_modules/06_module_{NN}_review.md`

##### 스크립트 상세도·교수 모델별 구조·발문·형성평가 (공통 규칙)

**스크립트 상세도 3수준** (`script_detail_level`):

| 요소 | full_script | semi_structured (기본값) | bullet_notes |
|------|-------------|------------------------|-------------|
| 교수자 발화 | 발화 전문 | 핵심 설명 단락 + 불릿 | 키워드 불릿 |
| 전환 멘트 | 완전 문장 | 완전 문장 | 키워드만 |
| 발문 | 완전 문장 + `[대기 3~5초]` + APPLE | 완전 문장 + `[대기]` | 완전 문장 (**항상 완전**) |
| 예상 답변 | 포함 (`include_expected_answers` 설정 따름) | 설정 따름 (기본: 미포함) | 미포함 |
| 활동 설명 | 단계별 상세 지시 (절차 1, 2, 3...) | 단계 목록 + 핵심 지시 | 활동명 + 소요시간 |
| 형성평가 | 문항 + 기준 + 피드백 분기 | 문항 + 기준 | 유형 + SLO |
| 적합 대상 | 초보 강사, 대리 강의 | 경험 있는 강사 | 숙련 강사, 리허설 |

**교수 모델별 차시 스크립트 구조**:

| 교수 모델 | 도입 | 전개 | 정리 |
|---------|------|------|------|
| 직접교수법 | Hook → SLO → 복습 | I Do(시범) → CFU → We Do(안내연습) → You Do(독립연습) | 피드백 → 형성평가 → 요약+예고 |
| PBL | 문제 시나리오 → SLO | NTK 도출 → 소그룹 탐구 → 해결책 | 갤러리워크 → 성찰 |
| 플립러닝 | 입장카드 → 오개념 정리 | 심화 활동 → 공유/발표 | 출구카드 + 사후과제 안내 |
| 혼합 | 차시별 개별 적용 | 차시별 개별 적용 | 차시별 개별 적용 |

**발문 설계 표기** (`questioning_design.include = true` 시):
- APPLE 모델: Ask → Pause(3~5초) → Pounce → Listen → Expound
- Bloom's 레벨 + 수렴/발산 유형 + SLO 연결 + 예상 답변(선택)
- 차시당 기본 4개: 도입 1 + 전개 2 + 정리 1

**형성평가 통합** (`formative_assessment.type`에 따라 배치):
- `sectional_check`: 전개 중간 1~2회 체크포인트 (I Do 후, We Do 후)
- `exit_ticket`: 정리 단계에서만 배치 — 역진설계로 체크포인트 결정
- `practice_integrated`: 실습 활동 내 통합 (수행 체크리스트)
- `none`: 형성평가 블록 생략
- SLO별 최소 1개 평가 커버리지 필수 (type ≠ none 시)
- 결과에 따른 분화 방안 표기 (80%+→진행 / 50~79%→재설명 / 50%미만→We Do 추가)

**템플릿**: `.claude/templates/script-template.md` (차시별 교안, 모듈별 통합, 최종 교안 구조 정의)

**Gagné 사태 표기 3모드**: `all_9`(9사태 전체 라벨 `[Gagné 1: 주의집중]`) / `core_5`(핵심 5사태 `[G1 주의집중]`) / `none`(라벨 생략, 수업 단계명만)

- 상세 워크플로우: `.claude/agents/writer-agent/AGENT.md`의 "강의교안 차시별 작성", "모듈 통합", "최종 통합" 섹션 참조

#### Phase 7: 최종 전체 통합 + 품질 검토

##### Phase 7a: 최종 전체 통합 (writer-agent, final_integrate 모드)

**입력**: 모든 `06_module_{NN}.md` + `01_input_data.json` + `05_arch_lesson_plan.md` + 구성안 `06_write_lecture_outline.md`

**작업**:
- 모든 모듈을 `06_write_lecture_script.md`로 최종 병합
- 코스 레벨 섹션 추가: 메타데이터, 강의 개요, 학습 목표, 시간표, 교수학적 정렬 매트릭스, 부록
- 전체 발문 인덱스 생성 (questioning_design.include 시)

##### Phase 7b: 최종 품질 검토 (review-agent, script_full_review 모드)

- 기존 5영역 25항목 + QM Rubric 전체 검토
- 하위 레벨(차시·모듈)에서 이미 통과한 항목은 경량 확인
- **교차 모듈 정합성**에 집중: 전체 정렬 매트릭스 완전성, 파이프라인 정합성, 전체 톤 일관성
- 기존 4단계 판정 (APPROVED / AWN / REVISE / REVISE MAJOR) 유지

#### 품질 검토 3계층 비교

| 계층 | 시점 | 에이전트 모드 | 검토 항목 수 | 초점 | FAIL 시 처리 |
|------|------|------------|-----------|------|------------|
| **Micro (차시)** | 6b | session_review | 3 | 시간/SLO/Bloom's | writer 1회 재작성 |
| **Meso (모듈)** | 6d | module_review | 4 | 연결성/톤/난이도/모듈SLO | writer 모듈수정 1회 |
| **Macro (전체)** | 7b | script_full_review | 25+QM | 파이프라인 정합/전체 정렬 | 기존 판정 분기 |

**원칙**: 하위 계층에서 걸러진 문제는 상위 계층에서 재검토하지 않는다. 상위는 하위에서 볼 수 없는 **교차 차시/교차 모듈** 문제에 집중한다.

**데이터 흐름**:
```
구성안 로드 → 01_input_data.json → 02_explore_research.md → 03_brainstorm_result.md
→ 04_deep_research.md → 05_arch_lesson_plan.md
→ [차시 루프: 06_session_{NNN}.md × N] → [모듈 루프: 06_module_{NN}.md × M]
→ 06_write_lecture_script.md → 07_review_quality.md
```

**산출물**: `lectures/YYYY-MM-DD_{강의명}/02_script/06_write_lecture_script.md`

---

### 워크플로우 3: 슬라이드 기획 (`/slide-planning`)

**목적**: 교안을 기반으로 슬라이드의 구조, 수, 레이아웃을 설계합니다.

| Phase | 단계 | 에이전트 | 핵심 작업 |
|-------|------|---------|----------|
| 1 | 입력 수집 | input-agent | 교안 4계층 로드(아키텍처+세션+통합본+JSON), 자동 추론 6개 + 사용자 질문 1개(도구) |
| 2 | 브레인스토밍 | brainstorm-agent | 조건부 3~5축 발산(유형별 시각화·메타포·코드·활동퀴즈·레이아웃), 4기준 가중 수렴(밀도·시각계층·인지부하·도구호환), A-E/도구 호환 매핑, 3역할 다관점 검증(시각 디자인 회의론자·청중 대리인·슬라이드 정렬 중재자), 11섹션 최종 통합 |
| 3 | 구조 설계 | architecture-agent | 12필드 추출+상속 범위 정의, 세션 유형 분류(교수 모델 보정), 도구별 레이아웃 카탈로그(Marp/Slidev/reveal.js/Gamma), 7조건 분기, 예산 공식, brainstorm §별 매핑, SLO→유형+Gagné→위치+3-소스 통합, 세션 유형별 시퀀스 템플릿+7규칙 시퀀싱+레이아웃 배정, 6중 검증(슬라이드수+시간+밀도+SLO+인지부하+내러티브, 조건부 SKIP) |
| 4 | 기획안 작성 | writer-agent | 3계층(Micro-Meso-Macro) 세션별 순차 작성, 슬라이드별 9필드(Assertion Headline/콘텐츠/시각자료/레이아웃/발표자 노트 5항목/시간/SLO/교안매핑), Visual Summary, 자체 검증 4항목, Day 통합, 6중 검증 |
| 5 | 품질 검토 | review-agent | QM Rubric 기준 4(교수자료)·5(학습활동) 상세 + 5영역 25항목 가중치 체크리스트(밀도25/시각20/정렬25/슬라이드수15/가독성15), 조건부 SKIP 5개, 독립 재산출, 파이프라인 정합성 3테이블, 4단계 판정, 자동 REVISE 4조건, Phase 2/3/4 재실행 결정 |

**설계 원칙**:
- 슬라이드당 1개 아이디어 (인지 부하 이론)
- 적응형 밀도: 슬라이드 유형별 자동 결정 (13유형, 3-35줄 범위)
- 시간 기준: 2-5분/슬라이드 (유형별 적응), 활동 시간 별도
- Assertion-Evidence 구조: partial 기본 (Headline 유지 + 시각 증거 30-40%)

**슬라이드 유형 (13가지)**:
제목, 아젠다, 섹션전환, 개념설명, 코드, 비교, 데이터+인사이트, 이미지, 타임라인, 인용, 핵심요약, 실습/활동, **퀴즈/형성평가**

**적응형 슬라이드 수** — 세션 유형별 범위:
- 개념 중심: 13-20장, 코드 중심: 10-15장, 실습 중심: 8-13장, 프로젝트: 8-13장, 혼합: 10-15장
- 차시당 구성: 도입 2-3장 + 전개 (유형별 적응) + 정리 2-3장

#### Phase 1: 입력 수집 상세

**교안 4계층 분리 로드 (사용자 개입 없음)**:
- L1: `05_arch_architecture.md` → 매크로 구조, 시간표, 연결 맵, 정렬 매트릭스
- L2: `06_sessions/06_session_{NNN}.md` × N개 → 차시별 상세 콘텐츠 (도입/전개/정리, 발문, 활동, 형성평가)
- L3: `06_write_lecture_script.md` 상단 ~100줄 → 메타데이터, SLO
- L4: `01/02_input_data.json` → 보조 설정 (teaching_model, tone_examples, keywords)

**자동 추론 6개** — 이전 단계 산출물에서 결정적 파생 (사용자에게 재질문하지 않음)

| # | 카테고리 | 추론 소스 | 추론 로직 |
|---|---------|----------|----------|
| PQ1 | 슬라이드 작성 범위 | `script_settings.target_scope` | 교안 작성 범위와 동기화 |
| PQ3 | 적응형 밀도 | 세션 유형 + 슬라이드 유형 | 유형별 밀도 자동 적용: concept 15-25, code 25-35, comparison 20-30, data_insight 10-20, image 8-15, timeline 12-20, summary 15-25, activity 15-25, quiz 10-20, title 3-5, agenda 5-12, section_transition 3-5, quote 5-10 (전체 3-35줄) |
| PQ4 | Assertion-Evidence | 고정값 | partial (Headline 유지, 시각 증거 30-40%) |
| PQ5 | 디자인 톤 | L3: `inherited.tone` (primary), L4: `tone_examples` (보조) | 키워드 매칭 (비유/친근→educational, 전문/기업→professional, 간결/미니멀→minimal, fallback→format.type 기반) |
| PQ6 | 발표자 노트 | 교안 워크플로우 특성 | 항상 `include = true` |
| PQ7 | 코드 테마 | `derived.has_code_content` | 코드 존재 시 `dark` 자동 설정 |

**사용자 질문 1개** — AskUserQuestion 1회 호출

| # | 카테고리 | 질문 | 입력 형태 |
|---|---------|------|----------|
| PQ2 | 슬라이드 도구 | 슬라이드 생성에 사용할 도구 선택 | 선택지: Marp(추천) / Slidev / Gamma / reveal.js — 콘텐츠 기반 자동 추천 |

**콘텐츠 유형 자동 탐지** — 세션 파일 내 키워드 매칭으로 `derived` 필드 생성:
- `has_code_content`: "코드", "코딩", "라이브 코딩", "코드 워크스루", "프로그래밍", "디버깅", ` ``` ` → 하나 이상 매칭 시 `true`
- `has_activity_content`: "실습", "그룹 활동", "팀 활동", "프로젝트", "워크숍", "핸즈온" → 하나 이상 매칭 시 `true`
- `has_quiz_content`: "형성평가", "퀴즈", "확인 문제", "자가진단", "평가 문항" → 하나 이상 매칭 시 `true`

**자동 추론 확인 게이팅** — AskUserQuestion 1회
- "자동 설정 그대로 진행"(추천) / "세부 설정 조정"
- 세부 조정 시 → PQ3(밀도) + PQ4(A-E) + PQ5(톤) + PQ7(코드 테마) 추가 질문 1회

**AskUserQuestion 호출 전략**: 최소 2회 ~ 최대 3회 (폴더선택 + PQ2 + 자동추론 확인 게이팅)

스키마: `.claude/temp/design_slide_plan_phase1.md` B섹션 참조

#### Phase 2: 브레인스토밍 상세

**핵심 전환**: 구성안 "무엇을 가르칠까" → 교안 "어떻게 가르칠까" → 슬라이드 기획 **"어떻게 시각적으로 전달할까"**

**입력 3개**: `03_slide_plan/01_input_data.json` + `01_outline/05_arch_architecture.md` + `02_script/06_sessions/` (대표 세션 2-3개 샘플링)

**조건부 브레인스토밍 축 (3~5개)**:

| 축 | 내용 | 생성 조건 |
|----|------|----------|
| 1 | **유형별 시각화 전략** — 13가지 슬라이드 유형 각각의 시각적 표현 | 항상 |
| 2 | **메타포·스토리텔링 시각화** — 교안의 비유·메타포를 시각 요소로 변환 | `tone_examples` 존재 시 |
| 3 | **코드 시각화 전략** — 코드 블록 표현·하이라이트·실행 결과 | `has_code_content = true` |
| 4 | **활동·퀴즈 시각화** — 실습 절차·퀴즈 레이아웃 | `has_activity_content` 또는 `has_quiz_content` |
| 5 | **레이아웃·색상·아이콘 시스템** — 크로스-세션 시각 일관성 | 항상 |

**5단계 워크플로우**: Step 0(시각화 맥락 분석·축 도출·가정 진술 5개) → Step 1(발산 — 축별 시각화 아이디어 × 6가지 기법 = 총 40-60개, 서브섹션 1-1~1-6) → Step 2(수렴 — 4기준 가중 평가, 유형-시각화-레이아웃 매핑 매트릭스, A-E partial 구조 검증, 도구 호환 매트릭스, 세션별 시각 전략+크로스-세션 일관성) → Step 3(다관점 검증 — 시각 디자인 회의론자·청중 대리인·슬라이드 정렬 중재자, Hard Stop 5개) → Step 4(최종 통합 — 11개 섹션 + Phase 3 연결 가이드)

**가정(Assumption) 5개**: 도구 가정(구현 가능성), 밀도 가정(범위 내 배치), A-E 가정(30-40% 시각 증거 충분), 일관성 가정(전 세션 적용), 접근성 가정(색각 이상 대응)

**핵심 수렴 매트릭스/검증 4개**:
1. **4기준 가중 평가** — 밀도 적합(30%) + 시각 계층(25%) + 인지 부하(25%) + 도구 호환(20%), 1-5점 척도, 가중 평균 3.5 이상 선별
2. **유형-시각화-레이아웃 매핑 매트릭스** — 13유형 전체에 시각화+레이아웃 매핑 필수 (빈 유형 시 Step 1 미선별 아이디어에서 보충)
3. **A-E partial 구조 검증** — 유형별 Assertion Headline + Evidence 30-40% 충족 확인
4. **도구 호환 매트릭스** — 선택 도구(Marp/Slidev/Gamma/reveal.js)에서 구현 가능성 (✅ 기본지원/△ 제한적/✗ 미지원), ✗ 시각화 → 대안 교체

**다관점 검증 (3역할 순차 전환)**:

| 역할 | 핵심 질문 | 허용 | 금지 |
|------|---------|------|------|
| 시각 디자인 회의론자 | "이 시각화가 불릿포인트보다 이해를 돕는가?" | 밀도 초과, A-E/Mayer 위반, 접근성 문제 지적 | 새 아이디어 제안, 레이아웃 직접 변경 |
| 청중 대리인 | "3초 내에 핵심을 파악할 수 있는가?" | 과부하·시선 흐름·단조로움·코드 가독성 지적 | 시각화 전략 변경, 세션 구조 변경 |
| 슬라이드 정렬 중재자 | "SLO-시각화 매핑이 완전한가?" | 우려 수용/기각 판정, 수정 지시, 정렬·일관성 확인 | 새 시각화 추가, 세션 구조 변경 |

**종료 기준 (Hard Stop)**: 3역할 리뷰 완료 + 우려사항 수용/기각 판정 + 수정사항 반영 + 결정 로그 작성 + 중재자 APPROVED

**조건부 분기**: `tone_examples` 없음 → 축 2 전체 생략 / `has_code_content = false` → 축 3 생략 / `has_activity_content = false AND has_quiz_content = false` → 축 4 생략 / `tool.selected` 값에 따라 도구 특화 시각화 추가 / `design_tone = minimal` → 장식 요소 감점 강화

**산출물**: `03_brainstorm_divergent.md` → `03_brainstorm_convergent.md` → `03_brainstorm_review.md` → `03_brainstorm_result.md` (★ 최종, 11개 섹션: 시드 요약·유형별 시각화·세션별 계획·메타포 매핑·코드 전략·활동/퀴즈 전략·색상/아이콘 체계·크로스-세션 일관성·다관점 검증 결과·설계 결정 로그·Phase 3 연결 가이드)

상세 워크플로우: `.claude/agents/brainstorm-agent/AGENT.md`의 "슬라이드 기획 브레인스토밍 (Phase 2) 세부 워크플로우" 섹션 참조

#### Phase 3: 구조 설계 상세

**핵심 전환**: 구성안 "어떤 차시에 무엇을 가르칠까"(코스 레벨) → 교안 "각 차시 안에서 어떻게 가르칠까"(레슨 레벨) → 슬라이드 기획 **"각 세션을 몇 장의 어떤 슬라이드로 전달할까"**(프레젠테이션 레벨)

**상속 vs 신규 설계**:
- **상속** (교안에서 그대로 사용): SLO 목록+Bloom's 수준, 매크로 구조(Phase A→B→C→D), 정렬 맵(SLO↔활동↔평가), 세션별 주제·활동·형성평가
- **신규 설계**: 슬라이드 수(세션별), 유형 배정, 순서(시퀀스), 슬라이드별 시간, 도구별 레이아웃, Assertion Headline 초안

**입력 4개**: `03_slide_plan/01_input_data.json` + `03_slide_plan/03_brainstorm_result.md` + `01_outline/05_arch_architecture.md` + `02_script/06_sessions/`

**4단계 워크플로우**: Step 0(컨텍스트 로드+슬라이드 예산 산출) → Step 1(Backward Design — 슬라이드 스토리 설계) → Step 2(슬라이드 시퀀스 설계) → Step 3(6중 검증 + 산출물 작성)

**Step 0 — 컨텍스트 로드 + 슬라이드 예산 산출 (7개 서브섹션)**:
- 0-A: 12필드 추출 (`slide_settings` 4개 + `inherited` 5개 + `derived` 3개)
- 0-B: 상속 범위 정의 — 상속 5항목(SLO, 매크로 구조, 정렬 맵, 주제·키워드, 활동·형성평가) vs 신규 6항목(슬라이드 수, 유형 배정, 순서, 시간, 레이아웃, Headline 초안) 명시적 구분
- 0-C: 세션 유형 분류 — 5유형(개념/코드/실습/프로젝트/혼합) + 교수 모델 보정 계수(DI ×1.0, PBL ×0.7, Flipped ×0.5, Mixed ×0.85)
- 0-D: 도구별 레이아웃 카탈로그 — Marp 7종, Slidev 12종(19종 중 주요 매핑), reveal.js 6종, Gamma 6종(블록 기반)
- 0-E: 조건부 분기 결정 — 7개 조건(`tool`, `has_code`, `has_activity`, `has_quiz`, `assertion_evidence`, `design_tone`, `teaching_model`)이 후속 Step 동작 결정
- 0-F: 슬라이드 예산 공식 — `실질_노출_시간 = session_minutes - activity_time - 5분` → `권장_슬라이드_수 = 실질_노출_시간 ÷ 평균_시간_per_장` → 교수모델 보정 → clamp(세션유형 최소~최대)
- 0-G: 브레인스토밍 결과(§1-§11) 섹션별 매핑 — 각 섹션을 Step 1~3의 구체 입력으로 분배(§2→유형 시각화, §3→세션별 시퀀스, §5→코드 레이아웃, §8→내러티브 검증 등)

**Step 1 — Backward Design: 슬라이드 스토리 설계 (3개 서브섹션)**:
- 1-A: SLO→슬라이드 유형 매핑 — Bloom's 6수준별 주/보조 유형 매핑 (기억→concept, 이해→concept/comparison, 적용→code/activity, 분석→comparison/data_insight, 평가→comparison/quiz, 창조→activity/code). 모든 SLO가 최소 1개 '주' 유형에 커버 필수
- 1-B: Gagné 9사태→슬라이드 위치 매핑 — 도입(사태 1·2·3, 3-4장) → 전개(사태 4·5·6·7·8, 본론 장수) → 정리(사태 9, 2-3장). `has_quiz=false` 시 사태 3·8의 quiz를 concept/activity 대체
- 1-C: 3-소스 통합 — ①brainstorm §3+§9(최우선, 검증된 시각화 전략) → ②도구 카탈로그(구현 제약) → ③SLO+Gagné 기본값(fallback). 인지부하 관리: 1아이디어/장, 15-20분마다 유형 전환, Mayer 분절 원칙

**Step 2 — 슬라이드 시퀀스 설계 (4개 서브섹션)**:
- 2-A: 세션 유형별 시퀀스 템플릿 — 5유형(개념/코드/실습/프로젝트/혼합) 각각의 기본 슬라이드 순서 패턴 정의 + brainstorm §3·§6으로 커스터마이즈
- 2-B: 시퀀싱 규칙 7개 — R1(도입: title→agenda→선수지식), R2(본론: concept→시각→activity→체크 자유 구성), R3(마무리: summary→Exit Ticket→예고), R4(동일 유형 3장 연속 금지), R5(concept 2장 후 시각적 유형 삽입), R6(15-20분 인지부하 전환 간격), R7(Assertion Headline 내러티브 연속성)
- 2-C: 시간 배분 — 유형별 시간 범위(title 1-2분, concept 3-5분, code 4-5분, activity 표시만 등) + 시간 예산 검증(슬라이드+활동+전환 ≈ session_minutes, ±5분 허용) + 초과/부족 시 조정 우선순위
- 2-D: 도구별 레이아웃 배정 — ①유형→카탈로그 후보 추출 → ②brainstorm §2/§5/§7 권장 우선 적용 → ③기본 레이아웃 배정 → ④`design_tone` 보정(educational/professional/minimal)

**Step 3 — 6중 검증 + 산출물 작성 (2개 서브섹션)**:
- 3-A: 6중 검증 — ①슬라이드 수(보정 범위 이탈 30% 초과→FAIL) ②시간 합산(±5분 오차) ③밀도 호환(조건부 SKIP: `has_code=false` 시 코드 밀도 건너뜀) ④SLO 커버리지(<90%→FAIL) ⑤인지부하(조건부 SKIP: 총 슬라이드 ≤10) ⑥내러티브 연속성(조건부 SKIP: `assertion_evidence=none`)
- 3-B: 산출물 `05_arch_slide_structure.md` — 7섹션 구조: §1 세션 유형 분류, §2 세션별 슬라이드 시퀀스(유형·Headline·시간·SLO·교안 매핑·레이아웃·Gagné), §3 SLO-슬라이드 매핑, §4 검증 결과(6항목), §5 설계 결정 로그, §6 도구별 레이아웃 매핑, §7 인지부하 관리 맵

**수정 모드** (Phase 5 REVISE 후속): 수정 가능 7항목(슬라이드 수/시퀀스/SLO 매핑/시간/유형 다양성/레이아웃/인지부하 전환) vs 수정 불가 4항목(SLO 변경/세션 추가삭제/교수 모델/슬라이드 도구). 수정 후 6중 검증 재실행 필수.

상세 워크플로우: `.claude/agents/architecture-agent/AGENT.md`의 "슬라이드 기획 구조 설계 (Phase 3) 세부 워크플로우" 섹션 참조

#### Phase 4: 기획안 작성 상세

**핵심 전환**: 구성안 "어떤 차시에 무엇을 가르칠까"(코스 레벨) → 교안 "각 차시 안에서 어떻게 가르칠까"(레슨 레벨) → 구조 설계 "몇 장의 어떤 슬라이드로 전달할까"(프레젠테이션 레벨) → 기획안 작성 **"각 슬라이드에 무엇을 넣고 어떻게 배치할까"**(슬라이드 레벨)

**설계 근거**: 교안 Phase 6(차시별 작성)의 3계층 아키텍처(Micro-Meso-Macro)를 슬라이드 기획에 적응. 순차 작성, Visual Summary, 경량 자체 검증, Day 통합 패턴을 적용.

**입력 4개**: `05_arch_slide_structure.md` + `01_input_data.json` + `03_brainstorm_result.md` + `{script_dir}/06_sessions/`

**3계층 아키텍처**:

| 계층 | 교안 대응 | Step | 핵심 작업 | 검증 |
|------|---------|------|---------|------|
| **Micro** (세션 단위) | 6a+6b | Step 1: `session_plan` | 1세션씩 순차 작성 + 직전 세션 참조 | 자체 검증 4항목 (밀도·시간·SLO·Assertion) |
| **Meso** (Day 단위) | 6c+6d | Step 2: `day_integrate` | Day 병합 + 시각 패턴 패치 + 전환 연결 | 일관성 검증 5항목 |
| **Macro** (전체) | 7a+7b | Step 3: `final_integrate` | 전체 병합 + 코스 레벨 섹션 + SLO 매핑 | 6중 검증 |

**분할 전략 (규모별 동작 모드)**:

| 총 세션 수 | 모드 | Micro (Step 1) | Meso (Step 2) | Macro (Step 3) | 산출물 구조 |
|-----------|------|--------------|-------------|-------------|-----------|
| ≤10 | 단일 | 세션별 순차 (단일 draft 내 섹션) | **생략** — 크로스-세션 일관성 검증 5항목만 | 최종 정제 + 6중 검증 | `_visual_summary.md` + `06_write_slide_plan_draft.md` + `06_write_slide_plan.md` |
| 11-20 | 2계층 | 세션별 순차 (단일 draft) | Day 통합 (draft 내 Day 섹션) | 최종 정제 + 6중 검증 | 동일 |
| 21+ | 3계층 full | 세션별 파일 (`06_session_plans/`) | Day별 파일 (`06_day_plans/`) | 전체 병합 + 코스 레벨 섹션 | `06_session_plans/` + `06_day_plans/` + `06_write_slide_plan.md` |

**Step 0 — 컨텍스트 로드 + 작성 전략 수립** (5개 동작):
- 0-A: `05_arch_slide_structure.md` 전체 로드 — §2 세션별 시퀀스(유형·순서·시간·SLO·레이아웃·Gagné), §4 검증 결과, §1 세션 유형
- 0-B: `03_brainstorm_result.md`에서 시각화 전략 추출 — §2 유형별 시각화, §3 세션별 계획, §5 코드 전략(조건부), §7 색상/아이콘, §8 크로스-세션 일관성
- 0-C: 분할 모드 결정 — `total_sessions` 기반 (≤10 / 11-20 / 21+), 3계층 full 시 디렉토리 생성
- 0-D: Visual Summary 초기화 — `_visual_summary.md` 생성 (글로벌 설정: 도구, 디자인 톤, 코드 테마, A-E 수준)
- 0-E: 도구별 레이아웃 제약 확인 — Marp(2열·이미지 배경, 복잡 그리드 불가), Slidev(2열·그리드·Vue), Gamma(블록 자유), reveal.js(HTML/CSS 자유)

**Step 1 — 세션별 슬라이드 기획 (`session_plan` 모드)**:

세션별 작성 절차 (7단계): ①컨텍스트 로드(arch §2 + 교안 + brainstorm §3 + Visual Summary) → ②도입 슬라이드 기획(2-3장: title + agenda + 선수지식 확인[`has_quiz_content=true` 시], Gagné 1-3) → ③본론 슬라이드 기획(유형별 상세, Gagné 4-8, 1아이디어/1슬라이드, Rule of Three ≤3) → ④마무리 슬라이드 기획(2-3장: summary + Exit Ticket[`formative_assessment.type≠none` 시] + 예고, Gagné 9) → ⑤발표자 노트 보강(5항목) → ⑥자체 검증(4항목) → ⑦Visual Summary 갱신

슬라이드별 9필드:

| 필드 | 설명 | 작성 가이드 |
|------|------|-----------|
| 유형 | 13가지 중 선택 | arch §2에서 결정됨 |
| Assertion Headline | 완전한 주장 문장 | A-E partial: "~이다", "~한다" (아래 A-E 규칙 참조) |
| 콘텐츠 구성 | 핵심 내용 요소 | 불릿 3-5개, 핵심 요소 ≤3 (Rule of Three) |
| 시각 자료 | 시각 증거 설명 | brainstorm §2 반영, 30-40% 시각 증거 |
| 레이아웃 | 배치 패턴 | 도구 제약 내, arch §6 매핑 참조 |
| 발표자 노트 | 교수자 지원 정보 | 5항목 (핵심포인트·보충설명·전환멘트·교수자행동·타이밍큐) |
| 시간 | 분 | 유형별 범위 내 |
| SLO | 관련 학습 목표 | arch §2 매핑 |
| 교안 매핑 | 교안 섹션 참조 | 세션 N의 도입/전개/정리 중 어디에서 파생 |

Assertion Headline 작성 규칙 — 유형별 A-E 적용:

| 유형 | A-E | Headline 형식 |
|------|-----|-------------|
| concept | ✅ partial | 주장 문장 — "~이다", "~한다" |
| comparison | ✅ partial | 비교 결론 — "A가 B보다 ~하다" |
| data_insight | ✅ partial | 데이터 해석 — "~를 보여준다" |
| image | ✅ partial | 시각 해석 — "~를 보여준다" |
| code | 변형 | 핵심 원리 — "~로 구현한다" |
| summary | 변형 | 핵심 메시지 — "~가 핵심이다" |
| timeline | 변형 | 시간적 주장 — "~에서 ~로 발전했다" |
| activity | 변형 | 활동 목표 — "~를 실습한다" |
| title, agenda, section_transition, quiz, quote | — | 일반 제목/질문/인용 (A-E 불필요) |

발표자 노트 5항목:

| # | 항목 | 내용 |
|---|------|------|
| 1 | 핵심 포인트 | 반드시 전달할 메시지 (1-2문장) |
| 2 | 보충 설명 | 슬라이드에 없지만 구두로 보충할 내용 |
| 3 | 전환 멘트 | 다음 슬라이드로 넘어갈 연결 문장 (3유형: 개념 연결·질문 전환·시각 전환) |
| 4 | 교수자 행동 | 시연, 질문, 대기 등 물리적 행동 지시 |
| 5 | 타이밍 큐 | `[N분/누적 M분] 설명 X분 + 질문 Y분` 형식 |

세션별 자체 검증 (4항목) — FAIL 시 해당 슬라이드만 1회 수정, 2차 FAIL → 경고 기록(`[WARN]`) 후 진행:

| # | 검증 | 기준 | FAIL 처리 |
|---|------|------|---------|
| V1 | 밀도 범위 | 콘텐츠 항목 수 ≤ 유형별 밀도 범위, 핵심 요소 ≤3 | 항목 추가/삭제 |
| V2 | 시간 합산 | 슬라이드 시간 합 ≈ session_minutes - activity_time (±3분) | 시간 재배분 |
| V3 | SLO 커버리지 | 대상 SLO 전체가 최소 1개 슬라이드에 매핑 | 슬라이드 추가/매핑 |
| V4 | Assertion 품질 | partial 유형의 Headline이 완전한 주장 문장 (키워드·질문 금지) | Headline 재작성 |

**session_revise 모드**: 자체 검증 FAIL 또는 Step 2 Day 통합에서 특정 세션 수정 필요 시 — FAIL 항목만 수정, 나머지 유지, 자체 검증 4항목 재실행

**Context7 조회 프로토콜** (조건부): `has_code_content=true` AND code 유형 슬라이드 시 → ①`resolve-library-id` → ②`get-library-docs(tokens=3000)` → ③슬라이드 반영 + 출처 기록 `[C7] {라이브러리명}`. 세션당 최대 1회, 기술 코드/API 전용 (교수법·시각화 금지)

**Visual Summary** — 교안 Running Summary에 대응:
- 매 세션 완료 후 `_visual_summary.md`에 append — 5항목: 레이아웃 빈도, 유형 분포, Assertion 톤(마지막 3개 Headline), 시각 요소, 전환 상태
- Day 통합(Step 2) 시 Day 수준 요약으로 리셋 (누적 오차 방지)
- 이전 세션 전체를 다시 읽지 않고 Visual Summary만으로 시각 맥락 유지

**Step 2 — Day 통합 (`day_integrate` 모드)** — ≤10세션 시 생략(크로스-세션 일관성 검증 5항목만):
- 발동: 해당 Day 모든 세션 Step 1 완료 후 자동
- 0\. 사전 검증 [MANDATORY]: 세션 기획 파일 존재 확인, 누락 시 **즉시 중단** → 누락 세션 목록 반환
- 1\. 병합: Day 내 세션 순서대로 결합 + Day 헤더 삽입
- 2\. 시각 패턴 일관성 패치: 동일 유형 레이아웃 통일, 색상/아이콘 스타일, 메타포 중복 제거, design_tone 확인
- 3\. 전환 슬라이드 연결: `section_transition`의 내러티브 확인, 부자연스러운 전환 멘트 재작성
- 4\. 일관성 검증 5항목: C1(레이아웃 일관) + C2(시각 패턴 일관) + C3(Assertion 톤 일관) + C4(전환 연결성) + C5(시간 배분 Day 내 ±5분) — 위반 시 직접 수정
- 5\. Visual Summary 리셋: Day 수준 요약으로 갱신

**Step 3 — 최종 통합 (`final_integrate` 모드)**:
- 발동: 모든 Day 통합 완료 후 (≤10세션: Step 1 + 크로스-세션 검증 완료 후)
- 1\. 코스 레벨 섹션 작성: 메타데이터 테이블(slide-plan-template.md) + 슬라이드 유형 범례(13유형) + 설계 개요(1-2문단)
- 2\. Day 병합: 모든 Day 순서대로 + Day-세션 계층 구조 유지
- 3\. SLO-슬라이드 매핑 매트릭스: 전체 SLO × 세션 × 슬라이드 유형 × Bloom's 크로스 테이블 + COVERED/MISSING 상태
- 4\. 세션 간 전환 테이블: From→To, 전환 유형(주제 연결/쉬는 시간/Day 전환/Phase 전환), 연결 요소
- 5\. 설계 결정 로그: 주요 결정 + 근거 + 대안 + 카테고리(V=시각, S=구조, D=밀도, T=도구)
- 6\. 6중 검증 실행:

| # | 검증 | 기준 | FAIL 처리 | 조건부 SKIP |
|---|------|------|---------|-----------|
| 1 | 슬라이드 수 | 세션별 보정 범위 이탈 30% 초과 | 슬라이드 추가/삭제 | — |
| 2 | 시간 합산 | 세션별 ±5분 | 시간 재배분 | — |
| 3 | 밀도 호환 | 유형별 밀도 범위 내 | 콘텐츠 조정 | `has_code=false` 시 code 밀도 SKIP |
| 4 | SLO 커버리지 | 전체 SLO 90%+ 매핑 | 슬라이드 추가/매핑 | — |
| 5 | 인지 부하 | 동일 유형 3장 연속 금지, 15-20분마다 전환 | 유형 교차 삽입 | 총 슬라이드 ≤10 시 SKIP |
| 6 | 내러티브 연속성 | Assertion Headline 논리적 흐름 | Headline 재작성 | `assertion_evidence=none` 시 SKIP |

**수정 모드** (Phase 5 REVISE 후속):
- `07_review_quality.md`의 Phase 4 수정 지시만 반영, 지적된 슬라이드만 수정 (Surgical Changes), `06_write_slide_plan.md` 직접 수정, 수정 후 6중 검증 재실행
- 수정 가능 8항목: Assertion 품질, 시각 자료, 밀도 범위, 발표자 노트, 레이아웃 비일관, 전환 멘트, SLO 미커버, 인지 부하 과다
- **APPROVED WITH NOTES 경량 수정**: P3(Suggestion) 항목만 반영, 6중 검증·재검토 불필요

상세 워크플로우: `.claude/agents/writer-agent/AGENT.md`의 "슬라이드 기획 기획안 작성 (Phase 4) 세부 워크플로우" 섹션 참조

#### Phase 5: 품질 검토 상세

**핵심 전환**: Phase 1-4가 설계·작성했다면, Phase 5는 **독립적 외부 검토자 관점**에서 "이 기획안으로 효과적인 슬라이드를 제작할 수 있는가?"를 QM 기준 4(교수자료)·5(학습활동) 중심으로 검증한다.

**입력 4개**: `06_write_slide_plan.md` + `01_input_data.json` + `05_arch_slide_structure.md` + `{script_dir}/06_sessions/` (샘플 2-3개)

**4단계 워크플로우**: Step 0(컨텍스트 로드 + 검토 기준선 수립) → Step 1(QM Rubric 적합성 검토) → Step 2(5영역 25항목 심층 검토) → Step 3(종합 판정 + 산출물 작성)

**Step 0 — 컨텍스트 로드 + 검토 기준선 수립** (6개 동작):
- 06_write_slide_plan.md 전체 로드 + 01_input_data.json 핵심 필드 12개 추출 (`slide_settings` 6개 + `inherited` 4개 + `derived` 3개)
- 05_arch_slide_structure.md §1-§7 로드 — §1(세션 유형), §2(시퀀스), §3(SLO 매핑), §4(6중 검증), §6(레이아웃), §7(인지부하 관리 맵)
- 교안 세션 파일 샘플 2-3개 로드 (서로 다른 세션 유형 우선)
- 조건부 SKIP 5개 확정 — `has_code=false`→1-5 SKIP, `A-E=none`→2-1·5-1 기준 완화, `총 슬라이드≤10`→4-5 SKIP, `has_activity=false AND has_quiz=false`→3-5 SKIP, `notes.include=false`→5-2 SKIP
- **독립 재산출** — 검토자가 세션별 기대 슬라이드 수를 직접 계산: `실질_노출_시간 = session_minutes - activity_time - 5분` → `÷ 유형별_평균_시간` → `× 교수모델_보정(DI:1.0, PBL:0.7, Flipped:0.5, Mixed:0.85)` → `clamp(세션유형 최소~최대)`. 이 값과 기획안 실제 슬라이드 수를 대조 (항목 4-1)

**Step 1 — QM Rubric 적합성 검토**:
- 기준 4(Instructional Materials) **상세** — 4.1(SLO 기여), 4.2(목적 명확, Assertion Headline), 4.3(신뢰 자료, 시각 구체성), 4.4(다양한 관점, comparison 슬라이드)
- 기준 5(Learning Activities) **상세** — 5.1(활동-SLO 정렬), 5.2(인터랙션 다양성, activity+quiz ≥15%, AWSM 루브릭), 5.3(피드백 분기 설계)
- 기준 1·2·8 간략 확인 (메타데이터 정합, SLO 상속, 접근성)
- QM 점수: 3(충족)/2(부분)/1(미충족). **기준 4 또는 5가 1점 → 자동 REVISE**

**Step 2 — 5영역 25항목 심층 검토**:

| 영역 | 가중치 | 5개 항목 핵심 | 참조 프레임워크 |
|------|--------|-------------|-------------|
| 정보 밀도 | 25% | 유형별 밀도 범위, 1아이디어/1슬라이드, Rule of Three(≤3), 노트 분리(Mayer 중복성), 코드 밀도 | AWSM 루브릭, Mayer 중복성 원칙 |
| 시각 계층 | 20% | A-E partial 적용, 레이아웃 일관성, 시각 자료 구체성, 도구 호환성, 크로스-세션 시각 일관 | Assertion-Evidence, 도구별 제약 |
| 학습목표 정렬 | 25% | SLO 커버리지, Bloom's-유형 정합(교차표), 교안 매핑 정확성, Gagné 배치 정합, 활동·퀴즈 SLO 연결 | QM 기준 4·5, Bloom's Taxonomy |
| 슬라이드 수 | 15% | 세션 유형별 범위(독립 재산출 대조), 시간 합산, 유형별 시간 범위, 활동 시간 분리, 인지부하 전환 간격 | 인지 부하 이론, R4·R6 규칙 |
| 가독성 | 15% | Assertion Headline 명확성, 노트 5항목 완성도, 전환 멘트 연속성, 접근성(WCAG AA), 문서 완성도 | WCAG 2.1, slide-plan-template.md |

항목별 0~10점, SKIP 항목 제외 후 영역 평균 → 가중 합산 = 총점

**Step 3 — 종합 판정 + 산출물 작성**:
- **총점**: `(밀도 × 0.25) + (시각 × 0.20) + (정렬 × 0.25) + (슬라이드수 × 0.15) + (가독성 × 0.15)`
- **4단계 판정**: APPROVED(8.0+), APPROVED WITH NOTES(6.0-7.9), REVISE(4.0-5.9), REVISE MAJOR(0-3.9)
- **자동 REVISE 4조건**: ①QM 기준 4 or 5 = 1점, ②영역1+3 평균 <5.0, ③SLO 커버리지 <90%, ④범위 이탈 세션 >30%
- **수정 지시**: P0(Critical) ~ P3(Suggestion) 우선순위, 항목별 권장 Phase(2/3/4) 기록
- **파이프라인 정합성 3테이블**: input→plan(8필드), arch→plan(§1-§7 6항목), script→plan(샘플 교안 매핑·콘텐츠 파생·시간 대응)
- **재실행 Phase 결정**:
  - Phase 4(writer-agent): Assertion 재작성, 밀도 조정, 노트 보강, 레이아웃 교체, 전환 멘트, 시각 자료 구체화 — arch 구조 변경 불필요일 때
  - Phase 3(architecture-agent): 슬라이드 수 부적절, 시퀀스 비논리적, SLO 미커버 구조 원인, Gagné 배치 역전
  - Phase 2(brainstorm-agent): 영역 2(시각 계층) 평균 <4.0, 도구 호환 전략 부재, 크로스-세션 일관성 없음

**수정 모드**: Phase 5는 검토만 수행. REVISE 시 Skill 오케스트레이터가 `07_review_quality.md`의 `§6-1 권장 재실행 시작 Phase`를 파싱하여 해당 에이전트를 수정 모드로 호출.

**산출물**: `07_review_quality.md` — §1(종합 판정·점수·SKIP·강점) + §2(QM 적합성) + §3(5영역 상세) + §4(수정 지시) + §5(파이프라인 정합성) + §6(재실행 가이드, REVISE 시만)

상세 워크플로우: `.claude/agents/review-agent/AGENT.md`의 "슬라이드 기획 품질 검토 (Phase 5) 세부 워크플로우" 섹션 참조

**데이터 흐름**:
```
교안 4계층 로드 → 01_input_data.json → 03_brainstorm_result.md
→ 05_arch_slide_structure.md → 06_write_slide_plan.md → 07_review_quality.md
```

**산출물**: `lectures/YYYY-MM-DD_{강의명}/03_slide_plan/06_write_slide_plan.md`

---

### 워크플로우 4: 슬라이드 생성 프롬프트 (`/slide-generation`)

**목적**: 기획안을 기반으로 실제 슬라이드 콘텐츠 또는 AI 도구용 프롬프트를 생성합니다.

| Phase | 단계 | 에이전트 | 핵심 작업 |
|-------|------|---------|----------|
| 1 | 입력 수집 | input-agent | 기획안 로드, 출력 형식 선택(Marp/Slidev/Gamma 프롬프트) |
| 2 | 프롬프트 생성 | writer-agent | slide-prompt-template.md 기반, 슬라이드별 마크다운 또는 프롬프트 |
| 3 | 품질 검토 | review-agent | 형식 검증, 콘텐츠 정확성, 일관성, 접근성 |

**지원 출력 형식**:

| 도구 | 적합 용도 | 특징 |
|------|----------|------|
| **Marp** | 범용 교육 강의 | 학습 곡선 최소, AI 생성 용이, Git 친화적 |
| **Slidev** | 코드 중심 개발 교육 | 줄별 하이라이트, Vue 컴포넌트, 라이브 코딩 |
| **Gamma** | 디자인 중시 프레젠테이션 | 시각적 완성도, 비개발자 접근성 |
| **reveal.js** | 고급 인터랙션 | 최대 커스터마이징, 웹 배포 |

**프롬프트 6대 필수 요소**: 청중 정의, 프레젠테이션 유형, 콘텐츠 범위, 톤과 스타일, 시각적 선호, 핵심 데이터

**산출물**: `lectures/YYYY-MM-DD_{강의명}/04_slides/06_write_slides.md`

---

## 산출물 저장 구조

### 폴더 명명 규칙

- **날짜 형식**: `YYYY-MM-DD` (예: `2026-03-05`)
- **강의명**: 사용자 입력 Q1(핵심 주제)에서 추출, 공백은 하이픈(`-`)으로 대체
- **예시**: `lectures/2026-03-05_claude-code-활용/`

### 산출물 명명 규칙

파일명 형식: **`{NN}_{단계이름}_{산출물이름}.{확장자}`**

| 구성 요소 | 설명 | 예시 |
|----------|------|------|
| `NN` | 2자리 Phase 번호 (01~07) | `01`, `06` |
| `단계이름` | 축약된 Phase 영문명 | `input`, `write` |
| `산출물이름` | 산출물의 서술적 이름 | `data`, `lecture_outline` |

**Phase별 단계이름 매핑**:

| Phase | 단계이름 | 용도 |
|-------|---------|------|
| 1 | `input` | 입력 수집 |
| 2 | `explore` | 탐색적 리서치 |
| 3 | `brainstorm` | 브레인스토밍 |
| 4 | `deep` | 심화 리서치 |
| 5 | `arch` | 아키텍처 설계 |
| 6 | `write` | 구성안/문서 작성 |
| 7 | `review` | 품질 검토 |

### 폴더 구조

```
lectures/
└── YYYY-MM-DD_{강의명}/                      # 날짜 + 강의명 (Phase 1에서 자동 생성)
    ├── 01_outline/                           # /lecture-outline 산출물
    │   ├── 01_input_data.json                   # Phase 1: 사용자 입력 (Q1~Q14)
    │   ├── 02_explore_plan.md                   # Phase 2: 리서치 계획
    │   ├── 02_explore_local.md                  # Phase 2: 로컬 참고자료 분석
    │   ├── 02_explore_nblm.md                   # Phase 2: NotebookLM 쿼리 결과
    │   ├── 02_explore_web.md                    # Phase 2: 인터넷 리서치 결과
    │   ├── 02_explore_research.md               # Phase 2: 4자료원 통합 최종 ★
    │   ├── 03_brainstorm_divergent.md           # Phase 3: 발산적 탐색 (중간)
    │   ├── 03_brainstorm_convergent.md          # Phase 3: 수렴 및 매핑 (중간)
    │   ├── 03_brainstorm_review.md              # Phase 3: 다관점 검증 (중간)
    │   ├── 03_brainstorm_result.md              # Phase 3: 브레인스토밍 최종 ★
    │   ├── 04_deep_plan.md                      # Phase 4: 심화 리서치 계획 (Scope+Plan)
    │   ├── 04_deep_local_nblm.md                # Phase 4: 로컬/NBLM 심화 재분석
    │   ├── 04_deep_web.md                       # Phase 4: 웹 심화 수집 (Retrieve)
    │   ├── 04_deep_research.md                  # Phase 4: 심화 리서치 최종 ★
    │   ├── 05_arch_architecture.md              # Phase 5: 아키텍처 설계
    │   ├── 06_write_outline_draft.md            # Phase 6: 구성안 초안 (중간)
    │   ├── 06_write_lecture_outline.md           # Phase 6: 최종 구성안 ★
    │   └── 07_review_quality.md                 # Phase 7: 품질 검토
    ├── 02_script/                            # /lecture-script 산출물
    │   ├── 01_input_data.json                   # Phase 1: 구성안 로드 + 교안 설정
    │   ├── 02_explore_plan.md                   # Phase 2: 탐색적 리서치 계획
    │   ├── 02_explore_local.md                  # Phase 2: 로컬 참고자료 분석
    │   ├── 02_explore_nblm.md                   # Phase 2: NotebookLM 쿼리 결과
    │   ├── 02_explore_web.md                    # Phase 2: 인터넷 리서치 결과
    │   ├── 02_explore_research.md               # Phase 2: 6자료원 통합 최종 ★
    │   ├── 03_brainstorm_divergent.md           # Phase 3: 발산적 탐색 (Step 1)
    │   ├── 03_brainstorm_convergent.md          # Phase 3: 수렴 및 매핑 (Step 2)
    │   ├── 03_brainstorm_review.md              # Phase 3: 다관점 검증 (Step 3)
    │   ├── 03_brainstorm_result.md              # Phase 3: 브레인스토밍 최종 ★
    │   ├── 04_deep_plan.md                      # Phase 4: 심화 리서치 계획
    │   ├── 04_deep_local_nblm.md                # Phase 4: 로컬/NBLM 심화 재분석
    │   ├── 04_deep_web.md                       # Phase 4: 웹 심화 수집 결과
    │   ├── 04_deep_research.md                  # Phase 4: 심화 리서치 최종 ★
    │   ├── 05_arch_lesson_plan.md               # Phase 5: 차시별 레슨 플랜 구조 설계
    │   ├── _running_summary.md                  # Phase 6: 누적 요약 (차시·모듈 작성 시 갱신)
    │   ├── 06_sessions/                         # Phase 6a: 차시별 교안
    │   │   ├── 06_session_001.md
    │   │   ├── 06_session_002.md
    │   │   └── ...
    │   ├── 06_modules/                          # Phase 6c: 4시간 모듈 통합
    │   │   ├── 06_module_01.md
    │   │   ├── 06_module_01_review.md           # Phase 6d: 모듈 검토
    │   │   └── ...
    │   ├── 06_write_lecture_script.md           # Phase 7a: 최종 통합 교안 ★
    │   └── 07_review_quality.md                 # Phase 7b: 최종 품질 검토
    ├── 03_slide_plan/                        # /slide-planning 산출물
    │   ├── 01_input_data.json                   # Phase 1: 교안 로드 + 도구/밀도 설정
    │   ├── 03_brainstorm_divergent.md           # Phase 2: 시각화 아이디어 발산 (중간)
    │   ├── 03_brainstorm_convergent.md          # Phase 2: 수렴·패턴 매핑 (중간)
    │   ├── 03_brainstorm_review.md              # Phase 2: 다관점 검증 (중간)
    │   ├── 03_brainstorm_result.md              # Phase 2: 브레인스토밍 최종 ★
    │   ├── 05_arch_slide_structure.md           # Phase 3: 슬라이드 구조 설계
    │   ├── _visual_summary.md                   # Phase 4: Visual Summary (작성 중 갱신)
    │   ├── 06_session_plans/                    # Phase 4: 세션별 기획 [21+세션 시]
    │   │   ├── 06_session_plan_001.md
    │   │   └── ...
    │   ├── 06_day_plans/                        # Phase 4: Day 통합 [21+세션 시]
    │   │   ├── 06_day_plan_01.md
    │   │   └── ...
    │   ├── 06_write_slide_plan_draft.md         # Phase 4: 기획안 초안 [≤20세션 시] (중간)
    │   ├── 06_write_slide_plan.md               # Phase 4: 최종 기획안 ★
    │   └── 07_review_quality.md                 # Phase 5: 품질 검토
    └── 04_slides/                            # /slide-generation 산출물
        └── 06_write_slides.md                   # 최종 슬라이드 ★
```

**참고**: 위 폴더 구조 트리에 모든 중간 산출물이 포함되어 있습니다. ★ 표시는 각 Phase의 최종 산출물(후속 Phase의 입력으로 사용)을 의미합니다.

### 공통 규칙

1. **폴더 자동 생성**: Phase 1(input-agent)에서 강의 루트 폴더 + 해당 워크플로우 폴더를 생성
2. **이전 단계 참조**: 각 워크플로우는 이전 단계 폴더의 최종 산출물을 입력으로 로드
3. **중간 산출물 보존**: 각 Phase의 중간 산출물도 해당 폴더에 저장 (디버깅·재실행 용도)
4. **input_data.json**: 각 워크플로우 폴더에 독립적으로 생성 (워크플로우별 입력이 다름)

---

## 품질 검증 프레임워크

### 워크플로우별 검증 기준

5영역 가중치 체크리스트(영역별 5개 항목, 항목별 0~10점)를 사용하되, 워크플로우 특성에 따라 영역명과 가중치가 다릅니다.

| 영역 | 구성안 | 교안 | 슬라이드 기획 | 슬라이드 생성 |
|------|--------|------|-------------|-------------|
| 영역 1 | 학습 목표 명확성 (25%) | 목표-활동-평가 정렬 (20%) | 정보 밀도 (25%) | 형식 정합 (30%) |
| 영역 2 | 목표-활동-평가 정렬 (25%) | 전개 완성도 (25%) | 시각 계층 (20%) | 콘텐츠 정확 (25%) |
| 영역 3 | 콘텐츠 구조/흐름 (15%) | 발문 수준 (20%) | 학습목표 정렬 (25%) | 스타일 일관 (20%) |
| 영역 4 | 시간 배분 현실성 (15%) | 시간 현실성 (20%) | 슬라이드 수 (15%) | 접근성 (15%) |
| 영역 5 | 자료 정확성/최신성 (20%) | 톤 일관성 (15%) | 가독성 (15%) | 완성도 (10%) |
| **QM 초점** | 기준 1~5 전체 (2·3 필수) | 기준 3·5 중심 (3·5 필수) | 기준 4·5 중심 | 기준 8 중심 |

**교안 조건부 가중치 재배분**: 교안은 설정에 따라 영역 가중치가 동적으로 변동합니다.
- `questioning_design.include == false` → 영역 3(발문 20%) 소멸 → 영역 2(+10%=35%), 영역 4(+10%=30%)
- `formative_assessment.type == "none"` → 형성평가 관련 항목(1-3, 4-5) SKIP (영역 내 나머지 항목 평균)
- `gagne_display.mode == "none"` → Gagné 완성도 항목(2-3) SKIP (영역 내 나머지 항목 평균)

**슬라이드 기획 조건부 SKIP**: 슬라이드 기획은 입력 데이터의 `derived` 필드와 설정에 따라 특정 체크 항목을 동적으로 건너뛴다.
- `has_code_content == false` → 항목 1-5(코드 슬라이드 밀도) SKIP
- `assertion_evidence.level == "none"` → 항목 2-1·5-1 기준 완화 (제목형 Headline 허용, 시각 증거 비율 SKIP)
- 총 슬라이드 ≤ 10 → 항목 4-5(인지부하 전환 간격) SKIP
- `has_activity == false AND has_quiz == false` → 항목 3-5(활동·퀴즈 SLO 연결) SKIP
- `speaker_notes.include == false` → 항목 5-2(발표자 노트 완성도) SKIP

**워크플로우별 상세**: `.claude/agents/review-agent/AGENT.md`의 "워크플로우별 동작" 비교 테이블 참조

### REVISE 판정 후속 처리 (재실행 루프)

| 판정 | 후속 처리 | 재검토 |
|------|---------|-------|
| APPROVED (8.0+) | 없음. 워크플로우 종료 | — |
| APPROVED WITH NOTES (6.0~7.9) | 사용자 선택 시 writer-agent 경량 수정 (P3 항목 반영) | 불필요 |
| REVISE (4.0~5.9) | 권장 Phase부터 재실행 → 최종 Phase 재검토 | 1회 |
| REVISE MAJOR (0~3.9) | 권장 Phase부터 재실행 → 최종 Phase 재검토 | 1회 |

**재실행 Phase 결정**: review-agent가 수정 지시별 "권장 재실행 Phase" 기록 → `max(P0/P1 항목의 권장 Phase)`가 시작점

| 문제 유형 | 구성안 예시 | 교안 예시 | 슬라이드 기획 예시 | 권장 Phase (구성안/교안 → 슬기획) |
|---------|-----------|---------|---------------|----------|
| 콘텐츠 수준 수정 | 동사 교체, 시간 조정, BOPPPS 보완, 메타데이터, 톤 | 발화 보강, 발문 교체, 시간 미세조정, 톤 수정, 형성평가 문항 보강, 발표자 노트, 전환 멘트 | Assertion 재작성, 밀도 조정, 노트 보강, 레이아웃 교체, 전환 멘트, 시각 자료 구체화 | Phase 6 → Phase 4 |
| 구조적 문제 | 정렬 맵, 차시 재배치, 평가 재설계, Phase 비율 | 레슨 플랜 재설계, GRR/Gagné 재배치, 활동 대폭 교체, SLO-활동 매핑 변경 | 슬라이드 수 부적절, 시퀀스 비논리적, SLO 미커버 구조 원인, Gagné 배치 역전, 유형 다양성 부족 | Phase 5 → Phase 3 |
| 전략/자료 부족 | 참고문헌 부족, 근거 자료 부재, 미검증 과다 | 발문 은행 부족, 활동 사례 부족, 형성평가 도구 부족 | 시각 계층 전반 미흡, 도구 호환 전략 부재, 크로스-세션 일관성 없음, 메타포 시각화 부재 | Phase 4 → Phase 2 |

**최대 재시도**: 1회 (원본 + 수정 = 총 2회). 2차 REVISE 시 사용자 개입 요청.

### Bloom's Taxonomy 참조

| 수준 | 핵심 동사 | 발문 패턴 |
|------|-----------|----------|
| 기억 | 정의, 나열, 식별 | "~은 무엇인가요?" |
| 이해 | 설명, 요약, 비교 | "자신의 말로 설명해 보세요" |
| 적용 | 적용, 사용, 실행 | "이 상황에 어떻게 적용하겠습니까?" |
| 분석 | 비교, 분류, 구분 | "A와 B의 차이점은?" |
| 평가 | 판단, 비판, 정당화 | "이 해결책의 장단점은?" |
| 창조 | 설계, 구성, 개발 | "새로운 해결 방안을 제안해 보세요" |

---

## 적용 교수설계 프레임워크 요약

| 프레임워크 | 적용 워크플로우 | 핵심 원칙 |
|-----------|--------------|----------|
| **Backward Design** | 구성안, 교안 | 학습결과 → 평가 → 학습경험 역순 설계 |
| **GAIDE** | 구성안 | Setup → 초안 → 매크로 정제 → 마이크로 정제 → 통합 |
| **BOPPPS** | 구성안 차시 설계 | Bridge-Outcomes-Pre-Participatory-Post-Summary (참여학습 50% 보호) |
| **Gagne 9사태** | 교안 | 주의획득 → 목표고지 → ... → 파지와 전이 촉진 (전체 9/핵심 5/라벨 없음 선택 가능) |
| **Hunter 6단계** | 교안 (직접교수법) | 복습 → 목표제시 → 제시 → 시범 → 안내연습 → 독립연습 |
| **PBL 6단계** | 교안 (PBL) | 문제 시나리오 → 문제 정의 → 탐구 → 해결책 개발 → 발표 → 성찰 |
| **Before/During/After** | 교안 (플립러닝) | 사전학습 확인 → 개념 명확화 → 그룹 활동 → 심화 적용 → 사후 과제 |
| **GRR** | 교안 | I Do(교사 시범) → We Do(안내 연습) → You Do(독립 수행), Phase A~D별 비율 자동 배치 |
| **QM Rubric** | 품질 검토 | 8개 일반 기준 (구성안: 2·3 필수, 교안: 3·5 필수, 슬기획: 4·5 필수), 목표-활동-평가 정렬 |
| **2-Pass Research** | 구성안, 교안 | 탐색적 리서치(문제 공간) → 브레인스토밍 → 심화 리서치(아이디어 검증) |
| **Assertion-Evidence** | 슬라이드 | 주장 제목 + 시각 증거 (불릿포인트 대체) |

---

## 리서치 출처

### 교수설계 프레임워크
- Virginia Tech CETL (Backward Design)
- Design Council UK, Double Diamond Framework
- Quality Matters Rubric 7th Edition
- OLC Course Review Scorecard

### AI 기반 교육 콘텐츠 설계
- GAIDE Framework, Purdue University (arXiv:2308.12276)
- NC State DELTA (AI 지원 코스 설계)
- EduCraft System, Tsinghua University (CIKM 2025)
- IdeaSynth, arXiv:2410.04025 (문헌 기반 반복 아이디어 개발)

### 브레인스토밍 · 창의적 아이디어 생성
- Minas et al. 2018, Decision Sciences (프라이밍과 전자 브레인스토밍)
- Kohn & Smith 2011, Applied Cognitive Psychology (고착 효과)

### 강의 · 교안 설계
- Carnegie Mellon Eberly Center (강의계획서 설계)
- PMC Gottlieb et al. 2024 (Educator's Blueprint)

### 슬라이드 설계
- PMC Naegle 2021 (Ten Simple Rules for Effective Slides)
- McGill University Teaching KB (교육용 슬라이드 설계)
- Marp, Slidev, reveal.js 공식 문서

### 슬라이드 품질 검증
- AWSM Rubric (Academic Writing and Slide Making) — 슬라이드 복잡도 최고 가중치
- PresentBench 2024 (프레젠테이션 품질 5차원 평가)
- Mayer's Multimedia Learning Principles — 중복성 원칙(노트≠슬라이드 복사), 분절 원칙
- WCAG 2.1 AA (색상 대비 4.5:1/3:1, 색각 이상 대응)
- Assertion-Evidence Design (Garner & Alley, IEEE/ASEE)
