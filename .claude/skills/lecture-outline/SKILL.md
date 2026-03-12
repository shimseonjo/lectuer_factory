---
name: lecture-outline
description: 강의구성안 생성 - 7단계 파이프라인 (입력수집 → 탐색리서치 → 브레인스토밍 → 심화리서치 → 아키텍처 → 작성 → 검토)
context: fork
allowed-tools: Agent, Read, Write, Glob, Grep, WebSearch, WebFetch, AskUserQuestion, mcp__Context7__resolve-library-id, mcp__Context7__get-library-docs
---

# 강의구성안 생성 워크플로우

<!-- TODO: 오케스트레이터 로직 구현 예정 -->

## 작업 지시

$ARGUMENTS

## 파이프라인 (7단계, 2-Pass Research)

### Phase 1: 입력 수집 → input-agent
### Phase 2: 탐색적 리서치 → research-agent

**지시**: 강의구성안을 위한 탐색적 리서치를 수행하세요.
**입력 파일**: `{output_dir}/01_input_data.json`
**산출물 위치**: `{output_dir}/` (02_explore_plan.md, 02_explore_local.md, 02_explore_nblm.md, 02_explore_web.md, 02_explore_research.md)
**모드**: 탐색적 (orientation) — 구체적 강의 목차/구성 노출 금지 (고착 효과 방지)
**제약**: 총 웹 검색 15회 이내, NBLM 쿼리 노트북당 5회 이내
**워크플로우**: Step 0(계획) → Step 1(로컬) → Step 2(NBLM) → Step 3(웹) → Step 4(통합)
**상세**: `.claude/agents/research-agent/AGENT.md`의 "강의구성안 탐색적 리서치" 섹션 참조

### Phase 3: 브레인스토밍 → brainstorm-agent

**지시**: 탐색적 리서치 결과를 기반으로 발산-수렴-검증 브레인스토밍을 수행하세요.
**입력 파일**: `{output_dir}/01_input_data.json`, `{output_dir}/02_explore_research.md`
**산출물 위치**: `{output_dir}/` (03_brainstorm_divergent.md, 03_brainstorm_convergent.md, 03_brainstorm_review.md, 03_brainstorm_result.md)
**워크플로우**: Step 0(맥락분석) → Step 1(발산) → Step 2(수렴·매핑) → Step 3(다관점검증) → Step 4(통합)
**상세**: `.claude/agents/brainstorm-agent/AGENT.md`의 "강의구성안 브레인스토밍" 섹션 참조

### Phase 4: 심화 리서치 → research-agent

**지시**: 브레인스토밍 결과를 기반으로 deep-research 방법론에 따라 심화 리서치를 수행하세요.
**입력 파일**: `{output_dir}/03_brainstorm_result.md`, `{output_dir}/01_input_data.json`, `{output_dir}/02_explore_research.md`
**방법론 참조**: `.claude/skills/deep-research/SKILL.md` (8단계 파이프라인)
**산출물 위치**: `{output_dir}/` (04_deep_plan.md, 04_deep_local_nblm.md, 04_deep_web.md, 04_deep_research.md)
**모드**: Standard (기본, 15~30 소스) — 사용자 요청 시 Deep 전환 가능
**제약**: 총 웹 검색 20~25회 이내, NBLM 노트북당 3~5쿼리
**워크플로우**: Step 0(Scope+Plan) → Step 1(로컬 재분석 → NBLM 재분석 → 웹 수집) → Step 2(3소스 삼각측량+Refine) → Step 3(Synthesize+Critique) → Step 4(Package)
**3소스 통합**: 로컬/NBLM/웹 세 가지 소스를 필수로 수집하고 삼각측량에서 교차검증
**상세**: `.claude/agents/research-agent/AGENT.md`의 "강의구성안 심화 리서치" 섹션 참조

### Phase 5: 아키텍처 설계 → architecture-agent

**지시**: 브레인스토밍·심화리서치 결과를 기반으로 Backward Design 아키텍처를 설계하세요.
**입력 파일**: `{output_dir}/01_input_data.json`, `{output_dir}/03_brainstorm_result.md`, `{output_dir}/04_deep_research.md`
**산출물 위치**: `{output_dir}/05_arch_architecture.md`
**설계 프레임워크**: Backward Design 3단계 역순 (목표→평가→활동), Constructive Alignment, 4단계 매크로 구조
**워크플로우**: Step 0(컨텍스트 로드+제약분석) → Step 1(목표 정제+평가 프레임) → Step 2(차시 구조+정렬 맵) → Step 3(3중 검증+작성)
**검증**: 정렬 검증(목표↔활동↔평가 완전 매핑) + 시간 예산 검증(배정≤가용×1.05) + 인지 부하 검증(차시당 개념≤5)
**상세**: `.claude/agents/architecture-agent/AGENT.md`의 "강의구성안 아키텍처 설계" 섹션 참조

### Phase 6: 구성안 작성 → writer-agent

**지시**: 아키텍처 설계와 이전 단계 산출물을 통합하여 최종 강의구성안을 작성하세요.
**입력 파일**: `{output_dir}/05_arch_architecture.md`, `{output_dir}/01_input_data.json`, `{output_dir}/03_brainstorm_result.md`, `{output_dir}/04_deep_research.md`
**템플릿**: `.claude/templates/outline-template.md`
**산출물 위치**: `{output_dir}/` (06_write_outline_draft.md, 06_write_lecture_outline.md)
**모드**: 최종 작성 — GAIDE 5단계 (Setup → Draft → Macro → Micro → Consolidation)
**차시 내부 구조**: BOPPPS 모델 (50분 = B5 + O2 + P5 + P25 + P5 + S5 + 휴식10)
**분할 작업**: 총 교시 수 기준 자동 분할 (≤20: 단일, 21~40: 2분할, 41+: 3분할). Phase A~D 매크로 구조를 Day 그룹 경계로 사용. 중간 산출물: `06_write_outline_chunk_N.md`
**§시간표 생성**: Step 2에서 `01_input_data.json`의 `schedule` 기반으로 일별×교시별 시간 그리드 생성. §4-3 뒤, §5 앞에 배치
**제약**: 정렬·시간·가독성 3중 검증 통과 필수
**워크플로우**: Step 0(컨텍스트 로드) → Step 1(코스 레벨) → Step 2(차시 레벨 + 시간표) → Step 3(검증·보강) → Step 4(최종 정제)
**상세**: `.claude/agents/writer-agent/AGENT.md`의 "강의구성안 구성안 작성" 섹션 참조

### Phase 7: 품질 검토 → review-agent

**지시**: 최종 강의구성안과 전체 파이프라인 산출물을 독립적 외부 검토자 관점에서 품질 검증하세요.
**입력 파일**: `{output_dir}/06_write_lecture_outline.md`, `{output_dir}/01_input_data.json`,
  `{output_dir}/05_arch_architecture.md`, `{output_dir}/03_brainstorm_result.md`, `{output_dir}/04_deep_research.md`
**산출물 위치**: `{output_dir}/07_review_quality.md`
**검토 프레임워크**: QM Rubric 7th Ed.(8기준) + 5영역 가중치 체크리스트(25/25/15/15/20)
**판정 체계**: APPROVED(8.0+) / APPROVED WITH NOTES(6.0~7.9) / REVISE(4.0~5.9) / REVISE MAJOR(0~3.9)
**자동 REVISE**: QM 기준 2·3 미충족, 영역 1·2 평균 5.0 미만
**워크플로우**: Step 0(컨텍스트 로드) → Step 1(QM 적합성) → Step 2(5영역 심층) → Step 3(판정+작성)
**상세**: `.claude/agents/review-agent/AGENT.md`의 "강의구성안 품질 검토" 섹션 참조

### Phase 7 이후: 판정 분기 처리

Phase 7 완료 후, `07_review_quality.md`를 읽어 판정에 따라 분기한다.

```
revision_count = 0

[Phase 7 완료]
  │
  ├─ Read: {output_dir}/07_review_quality.md → verdict, restart_phase 파싱
  │
  ├─ APPROVED (8.0+):
  │     → 사용자에게 완료 보고. 워크플로우 종료.
  │
  ├─ APPROVED WITH NOTES (6.0~7.9):
  │     → 사용자에게 §4 수정 지시(P3 항목) 표시
  │     → AskUserQuestion: "권고 사항을 반영하시겠습니까?"
  │     → "예": writer-agent 경량 수정 (P3만, 재검토 없음) → 종료
  │     → "아니오": 종료
  │
  ├─ REVISE (4.0~5.9) / REVISE MAJOR (0~3.9):
  │     → IF revision_count >= 1:
  │         STOP & CONSULT — 사용자에게 1차·2차 수정 지시 비교 보고
  │         AskUserQuestion으로 사용자 판단 대기 → 종료 또는 수동 재실행
  │     → revision_count += 1
  │     → §6-1 "권장 재실행 시작 Phase" 파싱 → restart_phase
  │
  │     → IF restart_phase == 6:
  │         Agent → writer-agent (수정 모드)
  │     → IF restart_phase == 5:
  │         Agent → architecture-agent (수정 모드) → writer-agent (재작성)
  │     → IF restart_phase == 4:
  │         Agent → research-agent (보강 모드) → architecture-agent (수정) → writer-agent (재작성)
  │
  │     → Phase 7 재실행 (review-agent)
  │     → 분기 로직 처음으로 (revision_count 확인)
```

**수정 모드 지시 템플릿**:

**writer-agent 수정 모드**:
```
지시: 07_review_quality.md의 수정 지시를 반영하여 06_write_lecture_outline.md를 수정하세요.
규칙: (1) P0·P1 반드시 반영 (2) P2 가능 범위 반영 (3) §6-3 강점 보호
      (4) 수정 범위 외 섹션 변경 금지 (5) 06_write_lecture_outline.md 직접 수정 (06_write_outline_draft.md 미갱신)
입력: 07_review_quality.md (추가), 06_write_lecture_outline.md, 05_arch_architecture.md, 01_input_data.json
```

**architecture-agent 수정 모드**:
```
지시: 07_review_quality.md §6-2의 Phase 5 수정 지시를 반영하여 05_arch_architecture.md를 수정하세요.
규칙: (1) 지적된 구조적 문제만 수정 (2) 3중 검증 재실행 (3) 05_arch_architecture.md 덮어쓰기
입력: 07_review_quality.md, 05_arch_architecture.md, 01_input_data.json, 03_brainstorm_result.md, 04_deep_research.md
```

**research-agent 보강 모드**:
```
지시: 07_review_quality.md §6-2의 Phase 4 수정 지시에 따라 부족한 자료를 보강하세요.
규칙: (1) 전면 재실행 아닌 보강(incremental) (2) 추가 웹 검색 5~10회 이내
      (3) 04_deep_research.md에 보강 섹션 추가(append)
입력: 07_review_quality.md, 04_deep_research.md, 01_input_data.json
```

**최대 재시도**: 1회 (원본 + 수정 = 총 2회). 2차 REVISE 시 사용자 개입 요청.

## 산출물 (01_outline/)

```
lectures/YYYY-MM-DD_{강의명}/01_outline/
├── 01_input_data.json              # Phase 1: 사용자 입력 (Q1~Q14)
├── 02_explore_plan.md             # Phase 2: 리서치 계획
├── 02_explore_local.md            # Phase 2: 로컬 참고자료 분석
├── 02_explore_nblm.md             # Phase 2: NotebookLM 쿼리 결과
├── 02_explore_web.md              # Phase 2: 인터넷 리서치 결과
├── 02_explore_research.md      # Phase 2: 4자료원 통합 최종 ★
├── 03_brainstorm_divergent.md      # Phase 3: 발산적 탐색 (중간)
├── 03_brainstorm_convergent.md    # Phase 3: 수렴 및 매핑 (중간)
├── 03_brainstorm_review.md        # Phase 3: 다관점 검증 (중간)
├── 03_brainstorm_result.md        # Phase 3: 브레인스토밍 최종 ★
├── 04_deep_plan.md       # Phase 4: 심화 리서치 계획 (Scope+Plan)
├── 04_deep_local_nblm.md          # Phase 4: 로컬/NBLM 심화 재분석
├── 04_deep_web.md        # Phase 4: 웹 심화 수집 (Retrieve)
├── 04_deep_research.md             # Phase 4: 심화 리서치 최종 ★
├── 05_arch_architecture.md              # Phase 5: 아키텍처 설계
├── 06_write_outline_draft.md             # Phase 6: 구성안 초안 (중간)
├── 06_write_lecture_outline.md           # Phase 6: 최종 구성안 ★
└── 07_review_quality.md            # Phase 7: 품질 검토
```
