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

**선택 질문 (7개)** — 기본값 존재, 미입력 시 자동 적용

| # | 카테고리 | 기본값 |
|---|---------|--------|
| Q7 | 선수 지식 | 없음 (초급부터) |
| Q8 | 제외 범위 | 없음 |
| Q9 | 평가 방식 | 형성평가 (퀴즈+실습) |
| Q10a | 교수 전략 | PBL + AI-first, 실습 50%+ |
| Q10b | 톤·스타일 | 비유 중심 설명 + 메타포 목록 |
| Q11 | 참고 자료 | 없음 (로컬 폴더 경로 / NotebookLM URL, 수집만) |
| Q12 | 맥락 | 독립 강의 |

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
- 4자료원 통합: 01_input_data.json + 로컬 참고자료 + NotebookLM + 인터넷 리서치
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
- 5자료원 통합: 01_input_data.json + 구성안 02_explore_research.md(상속) + 로컬 참고자료 + NotebookLM + 인터넷 리서치
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
| 1 | 입력 수집 | input-agent | 교안 로드, 슬라이드 도구(Marp/Slidev/Gamma) 선택 |
| 2 | 브레인스토밍 | brainstorm-agent | 시각화 아이디어, 레이아웃 패턴 구상, 인터랙션 요소 |
| 3 | 구조 설계 | architecture-agent | 슬라이드 수 결정, 유형 배정, 순서, 시간 배분 |
| 4 | 기획안 작성 | writer-agent | slide-plan-template.md 기반, 슬라이드별 목적/레이아웃/콘텐츠/시각자료 |
| 5 | 품질 검토 | review-agent | 정보 밀도, 시각 계층, 학습목표 정렬, 슬라이드 수 적절성 |

**설계 원칙**:
- 슬라이드당 1개 아이디어 (인지 부하 이론)
- 정보 밀도: 텍스트 5-7줄, 불릿 4-5개 이하
- 시간 기준: 1-2분/슬라이드 (교육용은 활동 시간 별도)
- Assertion-Evidence 구조 (불릿포인트 대체)

**슬라이드 유형 (12가지)**:
제목, 아젠다, 섹션전환, 개념설명, 코드, 비교, 데이터+인사이트, 이미지, 타임라인, 인용, 핵심요약, 실습/활동

**시간별 슬라이드 수 기준**:

| 시간 | 슬라이드 수 |
|------|-----------|
| 30분 | 15-32장 |
| 60분 | 30-45장 |
| 90분 | 45-70장 |

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
    │   ├── 01_input_data.json                   # Phase 1: 사용자 입력 (Q1~Q12)
    │   ├── 02_explore_research.md               # Phase 2: 탐색적 리서치 최종 ★
    │   ├── 03_brainstorm_result.md              # Phase 3: 브레인스토밍 최종 ★
    │   ├── 04_deep_research.md                  # Phase 4: 심화 리서치 최종 ★
    │   ├── 05_arch_architecture.md              # Phase 5: 아키텍처 설계
    │   ├── 06_write_lecture_outline.md           # Phase 6: 최종 구성안 ★
    │   └── 07_review_quality.md                 # Phase 7: 품질 검토
    ├── 02_script/                            # /lecture-script 산출물
    │   ├── 01_input_data.json                   # Phase 1: 구성안 로드 + 교안 설정
    │   ├── 02_explore_plan.md                   # Phase 2: 탐색적 리서치 계획
    │   ├── 02_explore_research.md               # Phase 2: 리서치 결과
    │   ├── 03_brainstorm_result.md              # Phase 3: 브레인스토밍 최종 ★
    │   ├── 04_deep_plan.md                      # Phase 4: 심화 리서치 계획
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
    │   └── 06_write_slide_plan.md               # 최종 슬라이드 기획 ★
    └── 04_slides/                            # /slide-generation 산출물
        └── 06_write_slides.md                   # 최종 슬라이드 ★
```

**중간 산출물 명명 예시** (01_outline/ 내부):

```
03_brainstorm_divergent.md       # Phase 3 중간: 발산적 탐색
03_brainstorm_convergent.md      # Phase 3 중간: 수렴 및 매핑
03_brainstorm_review.md          # Phase 3 중간: 다관점 검증
06_write_outline_draft.md        # Phase 6 중간: 구성안 초안
```

**중간 산출물 명명 예시** (02_script/ 내부):

```
03_brainstorm_divergent.md       # Phase 3 중간: 발산적 탐색 (Step 1)
03_brainstorm_convergent.md      # Phase 3 중간: 수렴 및 매핑 (Step 2)
03_brainstorm_review.md          # Phase 3 중간: 다관점 검증 (Step 3)
04_deep_local_nblm.md            # Phase 4 중간: 로컬/NBLM 심화 재분석 (Step 1)
04_deep_web.md                   # Phase 4 중간: 웹 심화 수집 결과 (Step 1)
06_sessions/06_session_NNN.md    # Phase 6a: 차시별 교안 (NNN=001,002,...)
06_modules/06_module_NN.md       # Phase 6c: 4시간 모듈 통합 (NN=01,02,...)
06_modules/06_module_NN_review.md # Phase 6d: 모듈 검토 결과
```

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

**워크플로우별 상세**: `.claude/agents/review-agent/AGENT.md`의 "워크플로우별 동작" 비교 테이블 참조

### REVISE 판정 후속 처리 (재실행 루프)

| 판정 | 후속 처리 | 재검토 |
|------|---------|-------|
| APPROVED (8.0+) | 없음. 워크플로우 종료 | — |
| APPROVED WITH NOTES (6.0~7.9) | 사용자 선택 시 writer-agent 경량 수정 (P3 항목 반영) | 불필요 |
| REVISE (4.0~5.9) | 권장 Phase부터 재실행 → 최종 Phase 재검토 | 1회 |
| REVISE MAJOR (0~3.9) | 권장 Phase부터 재실행 → 최종 Phase 재검토 | 1회 |

**재실행 Phase 결정**: review-agent가 수정 지시별 "권장 재실행 Phase" 기록 → `max(P0/P1 항목의 권장 Phase)`가 시작점

| 문제 유형 | 구성안 예시 | 교안 예시 | 권장 Phase |
|---------|-----------|---------|----------|
| 문서/스크립트 수준 수정 | 동사 교체, 시간 조정, BOPPPS 보완, 메타데이터, 톤 | 발화 보강, 발문 교체, 시간 미세조정, 톤 수정, 형성평가 문항 보강, 발표자 노트, 전환 멘트 | Phase 6 |
| 구조적 문제 | 정렬 맵, 차시 재배치, 평가 재설계, Phase 비율 | 레슨 플랜 재설계, GRR/Gagné 재배치, 활동 대폭 교체, SLO-활동 매핑 변경 | Phase 5 |
| 자료 부족 | 참고문헌 부족, 근거 자료 부재, 미검증 과다 | 발문 은행 부족, 활동 사례 부족, 형성평가 도구 부족 | Phase 4 |

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
| **QM Rubric** | 품질 검토 | 8개 일반 기준 (구성안: 2·3 필수, 교안: 3·5 필수), 목표-활동-평가 정렬 |
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
