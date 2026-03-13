---
name: slide-planning
description: 슬라이드 기획 - 5단계 파이프라인 (입력수집 → 브레인스토밍 → 구조설계 → 기획안작성 → 검토)
context: fork
allowed-tools: Agent, Read, Write, Glob, Grep, AskUserQuestion
---

# 슬라이드 기획 워크플로우

## 작업 지시

$ARGUMENTS

## 파이프라인 (5단계)

### Phase 1: 입력 수집 → input-agent

**지시**: 슬라이드 기획을 위한 입력을 수집하세요. 교안 산출물에서 4계층 로드(아키텍처+세션+통합본+JSON)를 수행하고, 자동 추론 6개 + 사용자 질문 1개(도구)로 설정을 수집하세요.
**산출물 위치**: `{output_dir}/01_input_data.json`
**4계층 로드**:
- L1: `{outline_dir}/05_arch_architecture.md` → 매크로 구조, 시간표, 연결 맵, 정렬 매트릭스
- L2: `{script_dir}/06_sessions/06_session_{NNN}.md` × N → 차시별 콘텐츠
- L3: `{script_dir}/06_write_lecture_script.md` 상단 ~100줄 → 메타데이터, SLO
- L4: `01/02_input_data.json` → 보조 설정
**자동 추론**: PQ1(범위동기화) + PQ3(적응형 밀도) + PQ4(A-E=partial) + PQ5(톤매칭) + PQ6(노트=true) + PQ7(코드테마)
**사용자 질문**: PQ2(슬라이드 도구)
**AskUserQuestion**: 최소 2회 ~ 최대 3회 (폴더선택 + PQ2 + 자동추론 확인 게이팅)
**상세**: `.claude/agents/input-agent/AGENT.md`의 "슬라이드 기획 입력 수집" 섹션 참조

### Phase 2: 브레인스토밍 → brainstorm-agent

**지시**: 입력 데이터를 기반으로 슬라이드 시각화 아이디어를 발산-수렴-검증 브레인스토밍하세요.
**입력 파일**: `{output_dir}/01_input_data.json`, `{outline_dir}/05_arch_architecture.md`, `{script_dir}/06_sessions/` (샘플 2-3개)
**산출물 위치**: `{output_dir}/` (03_brainstorm_divergent.md, 03_brainstorm_convergent.md, 03_brainstorm_review.md, 03_brainstorm_result.md)
**워크플로우**: Step 0(맥락분석) → Step 1(발산: 6기법×유형별 시각화) → Step 2(수렴: 밀도30%+시각계층25%+인지부하25%+도구호환20%) → Step 3(검증: 시각디자인 회의론자, 청중 대리인, 슬라이드 정렬 중재자) → Step 4(통합: 세션별 시각화 전략)
**상세**: `.claude/agents/brainstorm-agent/AGENT.md`의 "슬라이드 기획 브레인스토밍" 섹션 참조

### Phase 3: 구조 설계 → architecture-agent

**지시**: 브레인스토밍 결과를 기반으로 세션별 슬라이드 구조(수, 유형, 순서, 시간)를 설계하세요.
**입력 파일**: `{output_dir}/01_input_data.json`, `{output_dir}/03_brainstorm_result.md`, `{outline_dir}/05_arch_architecture.md`, `{script_dir}/06_sessions/`
**산출물 위치**: `{output_dir}/05_arch_slide_structure.md`
**적응형 슬라이드 수**: 세션 유형별 범위 — 개념 13-20장, 코드 10-15장, 실습 8-13장, 프로젝트 8-13장, 혼합 10-15장
**워크플로우**: Step 0(컨텍스트+예산산출) → Step 1(Backward Design: 슬라이드 스토리+SLO매핑) → Step 2(시퀀스 설계) → Step 3(4중 검증: 슬라이드수+시간합산+밀도호환+SLO커버리지)
**상세**: `.claude/agents/architecture-agent/AGENT.md`의 "슬라이드 기획 구조 설계" 섹션 참조

### Phase 4: 기획안 작성 → writer-agent

**지시**: 구조 설계와 시각화 전략을 통합하여 최종 슬라이드 기획안을 작성하세요.
**입력 파일**: `{output_dir}/05_arch_slide_structure.md`, `{output_dir}/01_input_data.json`, `{output_dir}/03_brainstorm_result.md`, `{script_dir}/06_sessions/`
**템플릿**: `.claude/templates/slide-plan-template.md`
**산출물 위치**: `{output_dir}/` (06_write_slide_plan_draft.md, 06_write_slide_plan.md)
**슬라이드별 상세**: Assertion Headline, 콘텐츠 구성, 시각 자료, 레이아웃, 발표자 노트(4항목), 시간, SLO, 교안 매핑
**분할 기준**: ≤10세션 단일, 11-20 2분할, 21+ 3분할
**워크플로우**: Step 0(컨텍스트+작성전략) → Step 1(세션별 슬라이드 기획) → Step 2(크로스-세션 일관성) → Step 3(최종 정제+3중 검증)
**상세**: `.claude/agents/writer-agent/AGENT.md`의 "슬라이드 기획 기획안 작성" 섹션 참조

### Phase 5: 품질 검토 → review-agent

**지시**: 최종 슬라이드 기획안을 독립적 외부 검토자 관점에서 품질 검증하세요.
**입력 파일**: `{output_dir}/06_write_slide_plan.md`, `{output_dir}/01_input_data.json`, `{output_dir}/05_arch_slide_structure.md`, `{script_dir}/06_sessions/` (샘플)
**산출물 위치**: `{output_dir}/07_review_quality.md`
**검토 프레임워크**: QM 기준 4(교수자료)·5(학습활동) + 5영역 가중치(정보밀도25, 시각계층20, 학습목표정렬25, 슬라이드수15, 가독성15)
**판정 체계**: APPROVED(8.0+) / APPROVED WITH NOTES(6.0~7.9) / REVISE(4.0~5.9) / REVISE MAJOR(0~3.9)
**자동 REVISE**: 영역 1+3 평균 5.0 미만, SLO 커버리지 <90%, 범위 이탈 세션 >30%
**상세**: `.claude/agents/review-agent/AGENT.md`의 "슬라이드 기획 품질 검토" 섹션 참조

### Phase 5 이후: 판정 분기 처리

Phase 5 완료 후, `07_review_quality.md`를 읽어 판정에 따라 분기한다.

```
revision_count = 0

[Phase 5 완료]
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
  │     → IF restart_phase == 4:
  │         Agent → writer-agent (수정 모드)
  │     → IF restart_phase == 3:
  │         Agent → architecture-agent (수정 모드) → writer-agent (재작성)
  │     → IF restart_phase == 2:
  │         Agent → brainstorm-agent (수정 모드) → architecture-agent (수정) → writer-agent (재작성)
  │
  │     → Phase 5 재실행 (review-agent)
  │     → 분기 로직 처음으로 (revision_count 확인)
```

**수정 모드 지시 템플릿**:

**writer-agent 수정 모드**:
```
지시: 07_review_quality.md의 수정 지시를 반영하여 06_write_slide_plan.md를 수정하세요.
규칙: (1) P0·P1 반드시 반영 (2) P2 가능 범위 반영 (3) §5 강점 보호
      (4) 수정 범위 외 슬라이드 변경 금지 (5) 06_write_slide_plan.md 직접 수정
입력: 07_review_quality.md (추가), 06_write_slide_plan.md, 05_arch_slide_structure.md, 01_input_data.json
```

**architecture-agent 수정 모드**:
```
지시: 07_review_quality.md §6-2의 Phase 3 수정 지시를 반영하여 05_arch_slide_structure.md를 수정하세요.
규칙: (1) 지적된 구조적 문제만 수정 (2) 4중 검증 재실행 (3) 05_arch_slide_structure.md 덮어쓰기
입력: 07_review_quality.md, 05_arch_slide_structure.md, 01_input_data.json, 03_brainstorm_result.md
```

**brainstorm-agent 수정 모드**:
```
지시: 07_review_quality.md의 시각 계층 수정 지시를 반영하여 시각화 전략을 보강하세요.
규칙: (1) Step 2(수렴)부터 재실행 (2) 03_brainstorm_result.md 덮어쓰기
입력: 07_review_quality.md, 03_brainstorm_divergent.md, 01_input_data.json
```

**최대 재시도**: 1회 (원본 + 수정 = 총 2회). 2차 REVISE 시 사용자 개입 요청.

## 경로 규칙

```
{lecture_dir} = lectures/YYYY-MM-DD_{강의명}/
{outline_dir} = {lecture_dir}/01_outline/
{script_dir}  = {lecture_dir}/02_script/
{output_dir}  = {lecture_dir}/03_slide_plan/
```

## 산출물 (03_slide_plan/)

```
lectures/YYYY-MM-DD_{강의명}/03_slide_plan/
├── 01_input_data.json               # Phase 1: 교안 4계층 로드 + 도구 설정
├── 03_brainstorm_divergent.md       # Phase 2: 시각화 아이디어 발산 (중간)
├── 03_brainstorm_convergent.md      # Phase 2: 수렴·패턴 매핑 (중간)
├── 03_brainstorm_review.md          # Phase 2: 다관점 검증 (중간)
├── 03_brainstorm_result.md          # Phase 2: 브레인스토밍 최종 ★
├── 05_arch_slide_structure.md       # Phase 3: 슬라이드 구조 설계
├── 06_write_slide_plan_draft.md     # Phase 4: 기획안 초안 (중간)
├── 06_write_slide_plan.md           # Phase 4: 최종 기획안 ★
└── 07_review_quality.md             # Phase 5: 품질 검토
```
