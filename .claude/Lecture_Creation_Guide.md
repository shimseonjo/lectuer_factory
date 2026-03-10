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
| 6 | 구성안 작성 | writer-agent | outline-template.md 기반, GAIDE 5단계 + BOPPPS 차시 모델(B5-O2-P5-P25-P5-S5-휴식10) |
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
- 4자료원 통합: input_data.json + 로컬 참고자료 + NotebookLM + 인터넷 리서치
- 5단계 통합 알고리즘: 주제 축 추출 → 축별 배정 → 교차검증(삼각측량) → 고착효과 필터 → 구조화 작성
- 상세 워크플로우: `.claude/agents/research-agent/AGENT.md` 참조

#### Phase 4 심화 리서치 — deep-research 스킬 적용

- 스킬: `199-biotechnologies/claude-deep-research-skill@deep-research` (설치 완료, `--copy`)
- research-agent가 `.claude/skills/deep-research/SKILL.md`의 8단계 파이프라인을 방법론 레퍼런스로 따름
- 8단계: Scope → Plan → Retrieve → Triangulate → Outline Refinement → Synthesize → Critique → Refine → Package
- 브레인스토밍 결과(`brainstorm_result.md` §8)를 입력으로 구체적 사례·참고문헌·검증 데이터 수집
- 인용 기반 보고서 생성 (anti-hallucination protocol: 모든 팩트에 즉시 인용 `[N]` 부여)
- 비판적 자기검증 (Critique): 수집 편향, 최신성, 다양성, 커버리지 점검
- 검증 스크립트: `verify_citations.py`, `validate_report.py` 활용
- 모드 선택: Standard(5-10분, 기본) 또는 Deep(10-20분, 철저 검증)
- 상세 워크플로우: `.claude/agents/research-agent/AGENT.md`의 "강의구성안 심화 리서치" 섹션 참조

**데이터 흐름**:
```
사용자 입력 → input_data.json → research_exploration.md → brainstorm_result.md
→ research_deep.md → architecture.md → lecture_outline.md → quality_review.md
```

**산출물**: `lectures/YYYY-MM-DD_{강의명}/01_outline/lecture_outline.md`

#### Phase 5: 아키텍처 설계 상세

**4단계 매크로 구조 (Compositional Stacking)**:

```
Phase A: 환경/기초 정립 ─── ~15%  │ 이론:실습 = 70:30  │ I Do 중심
Phase B: 핵심 개념 습득 ─── ~35%  │ 이론:실습 = 50:50  │ I Do → We Do
Phase C: 심화 실습 ─────── ~30%  │ 이론:실습 = 30:70  │ We Do → You Do
Phase D: 통합/산출물 ────── ~20%  │ 이론:실습 = 10:90  │ You Do 중심
```

비율은 `schedule.days`, `pedagogy`, `target_learner.level`에 따라 보정.

**3중 검증** (architecture.md 완성 전 필수):

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

---

### 워크플로우 2: 강의교안 (`/lecture-script`)

**목적**: 구성안을 기반으로 실제 강의에 사용하는 상세 교안을 작성합니다.

| Phase | 단계 | 에이전트 | 핵심 작업 |
|-------|------|---------|----------|
| 1 | 입력 수집 | input-agent | 구성안 로드, 교수법 선택(직접교수/PBL/플립러닝/액티브러닝) |
| 2 | 탐색적 리서치 | research-agent | 교수법 사례, 유사 교안 벤치마킹, 실생활 사례 탐색 (방향 설정) |
| 3 | 브레인스토밍 | brainstorm-agent | 발문 설계(Bloom's 기반), 학습활동 아이디어, 실생활 사례 구상 |
| 4 | 심화 리서치 | research-agent | 브레인스토밍 기반 예시 자료, 보충 콘텐츠, 참고 문헌 |
| 5 | 교안 구조 설계 | architecture-agent | 도입-전개-정리(10:70:20), Gagne 9가지 수업사태 적용, 시간 배분 |
| 6 | 교안 작성 | writer-agent | script-template.md 기반, 섹션별 스크립트/발문/활동/평가문항/발표자 노트 |
| 7 | 품질 검토 | review-agent | 목표-활동-평가 정렬, 시간 배분 현실성, 용어/톤 일관성 |

**적용 프레임워크**:
- Madeline Hunter 8단계 (직접교수법)
- Gagne의 9가지 수업사태
- Gradual Release of Responsibility (I Do → We Do → You Do)

**교안 구조**:
```
도입 (10-15%): 주의집중 → 동기부여 → 선수학습 확인 → 학습목표 제시
전개 (70-80%): 내용제시 → 시범 → 안내연습 → 독립연습 (× 섹션 수)
정리 (10-15%): 핵심요약 → 실천방법 → 과제안내 → 차시예고
```

**데이터 흐름**:
```
구성안 로드 → input_data.json → research_exploration.md → brainstorm_result.md
→ research_deep.md → architecture.md → lecture_script.md → quality_review.md
```

**산출물**: `lectures/YYYY-MM-DD_{강의명}/02_script/lecture_script.md`

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

**산출물**: `lectures/YYYY-MM-DD_{강의명}/03_slide_plan/slide_plan.md`

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

**산출물**: `lectures/YYYY-MM-DD_{강의명}/04_slides/slides.md`

---

## 산출물 저장 구조

### 폴더 명명 규칙

- **날짜 형식**: `YYYY-MM-DD` (예: `2026-03-05`)
- **강의명**: 사용자 입력 Q1(핵심 주제)에서 추출, 공백은 하이픈(`-`)으로 대체
- **예시**: `lectures/2026-03-05_claude-code-활용/`

### 폴더 구조

```
lectures/
└── YYYY-MM-DD_{강의명}/              # 날짜 + 강의명 (Phase 1에서 자동 생성)
    ├── 01_outline/                   # /lecture-outline 산출물
    │   ├── input_data.json              # Phase 1: 사용자 입력 (Q1~Q12)
    │   ├── research_exploration.md      # Phase 2: 탐색적 리서치 최종 ★
    │   ├── brainstorm_result.md         # Phase 3: 브레인스토밍 최종 ★
    │   ├── research_deep.md             # Phase 4: 심화 리서치 최종 ★
    │   ├── architecture.md              # Phase 5: 아키텍처 설계
    │   ├── lecture_outline.md           # Phase 6: 최종 구성안 ★
    │   └── quality_review.md            # Phase 7: 품질 검토
    ├── 02_script/                    # /lecture-script 산출물
    │   └── lecture_script.md            # 최종 교안 ★
    ├── 03_slide_plan/                # /slide-planning 산출물
    │   └── slide_plan.md                # 최종 슬라이드 기획 ★
    └── 04_slides/                    # /slide-generation 산출물
        └── slides.md                    # 최종 슬라이드 ★
```

### 공통 규칙

1. **폴더 자동 생성**: Phase 1(input-agent)에서 강의 루트 폴더 + 해당 워크플로우 폴더를 생성
2. **이전 단계 참조**: 각 워크플로우는 이전 단계 폴더의 최종 산출물을 입력으로 로드
3. **중간 산출물 보존**: 각 Phase의 중간 산출물도 해당 폴더에 저장 (디버깅·재실행 용도)
4. **input_data.json**: 각 워크플로우 폴더에 독립적으로 생성 (워크플로우별 입력이 다름)

---

## 품질 검증 프레임워크

### 공통 검증 기준

| 영역 | 가중치 | 검증 내용 |
|------|--------|----------|
| 학습 목표 명확성 | 25% | 측정 가능한 동사, 학습자 중심, 수준 적합성 |
| 목표-활동-평가 정렬 | 25% | 정렬 맵 일관성, 누락 없음 |
| 콘텐츠 구조/흐름 | 15% | 논리적 순서, 난이도 점진적 상승 |
| 시간 배분 현실성 | 15% | 인지 과부하 방지, 활동 시간 충분 |
| 자료 정확성/최신성 | 20% | 팩트 체크, AI 환각 검증, 출처 신뢰성 |

### REVISE 판정 후속 처리 (재실행 루프)

| 판정 | 후속 처리 | 재검토 |
|------|---------|-------|
| APPROVED (8.0+) | 없음. 워크플로우 종료 | — |
| APPROVED WITH NOTES (6.0~7.9) | 사용자 선택 시 writer-agent 경량 수정 (P1~P3 반영) | 불필요 |
| REVISE (4.0~5.9) | 권장 Phase부터 재실행 → Phase 7 재검토 | 1회 |
| REVISE MAJOR (0~3.9) | 권장 Phase부터 재실행 → Phase 7 재검토 | 1회 |

**재실행 Phase 결정**: review-agent가 수정 지시별 "권장 재실행 Phase" 기록 → `max(P0/P1 항목의 권장 Phase)`가 시작점
- Phase 6: 문서 수준 수정 (동사 교체, 시간 조정, BOPPPS 보완 등)
- Phase 5: 구조적 문제 (정렬 맵, 차시 재배치, 평가 재설계 등)
- Phase 4: 자료 부족 (보강 모드, 추가 검색 5~10회)

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
| **Gagne 9사태** | 교안 | 주의획득 → 목표고지 → ... → 파지와 전이 촉진 |
| **Hunter 8단계** | 교안 | 선행조직자 → 목표 → 정보제공 → 시범 → 이해확인 → 연습 → 정리 |
| **QM Rubric** | 품질 검토 | 8개 일반 기준, 목표-활동-평가 정렬 |
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
