---
name: lecture-script
description: 강의교안 생성 - 7단계 파이프라인 (입력수집 → 탐색리서치 → 브레인스토밍 → 심화리서치 → 구조설계 → 작성 → 검토)
context: fork
allowed-tools: Agent, Read, Write, Glob, Grep, WebSearch, WebFetch, AskUserQuestion
---

# 강의교안 생성 워크플로우

## 작업 지시

$ARGUMENTS

## 파이프라인 (7단계, 2-Pass Research)

### Phase 1: 입력 수집 → input-agent

**지시**: 강의교안 작성을 위해 구성안을 로드하고 교안 설정을 수집하세요.
**산출물 위치**: `{output_dir}/01_input_data.json`
**상세**: `.claude/agents/input-agent/AGENT.md`의 "강의교안 입력 수집" 섹션 참조

**input-agent 실행 절차**:

1. **구성안 탐색**: `lectures/` 폴더를 Glob으로 스캔 → 최신순 정렬 → AskUserQuestion으로 선택
   - 폴더 없음/0개 → 오류 메시지 출력 후 종료
   - 폴더 1개: 해당 폴더 + "기타 직접 입력" = 2개 선택지
   - 폴더 2개: 2개 + "기타 직접 입력" = 3개 선택지
   - 폴더 3개+: 최신 3개 + "기타 직접 입력" = 4개 선택지
   - 선택된 폴더의 `01_outline/06_write_lecture_outline.md` 존재 확인 (없으면 오류 후 종료)

2. **구성안 자동 파싱**: `06_write_lecture_outline.md` + `01_input_data.json`에서 inherited 필드 전체 추출
   - `total_sessions` 자동 산출 (sessions 배열 길이)
   - `pedagogy`에서 교수 모델·활동 전략 자동 추론 (설계 문서 §F 참조)
   - `theory_practice_ratio` fallback 적용 (Phase A:70:30, B:50:50, C:30:70, D:10:90)

3. **필수 질문 1회차**: AskUserQuestion (3개 묶음)
   - **SQ1 — 교안 작성 범위**: 전체 차시(추천) / 특정 차시 선택
   - **SQ1a — 교수 모델**: {추론결과}(추천) / 직접교수법 / PBL / 플립러닝
   - **SQ1b — 활동 전략** (multiSelect): 개인실습 / 그룹활동 / 토론·발문 / 프로젝트
   - SQ1 "특정 차시" → 후속 질문으로 Day 번호 입력받아 session_ids 산출

4. **필수 질문 2회차**: AskUserQuestion (3개 묶음)
   - **SQ2 — 스크립트 상세도**: 완전 스크립트 / 반구조화(추천) / 불릿 노트
   - **SQ3 — 형성평가 유형**: 섹션별 체크(추천) / Exit Ticket / 실습 통합 / 없음
   - **SQ4 — 시간 비율**: 교수 모델 기반 자동(추천) / 직접 입력

5. **추가 설정 게이팅**: AskUserQuestion
   ```
   "세부 설정을 조정하시겠습니까?"
   1. 기본값으로 진행 (Recommended)
   2. 세부 설정 조정
   ```
   - "기본값" → SQ5~SQ7 기본값 적용 (gagne: core_5, 발문: 포함/답변제외/4개, 실습: step_list)
   - "세부 설정" → SQ5(Gagne) → SQ6(발문) → SQ7(실습 가이드) 순차 질문

6. **output_dir 결정**: `lectures/{선택한 강의 폴더}/02_script/`
   - 폴더 생성 (없으면)
   - `01_input_data.json` 저장 (inherited + script_settings + source + metadata)
   - `instructional_model_map` 자동 파생 (설계 문서 §E 매핑 테이블 기반)
   - Phase 1 완료 검증 (설계 문서 §H 기준: 필수 필드 존재, time_ratio 합계 100, enum 유효성)
   - 수집된 설정 요약 출력

### Phase 2: 탐색적 리서치 → research-agent (교수법 사례, 유사 교안 벤치마킹)

**지시**: 강의교안을 위한 탐색적 리서치를 수행하세요. 교수법 적용 사례, 발문 전략, 학습활동 설계, 형성평가 도구를 중심으로 리서치합니다.
**입력 파일**: `{output_dir}/01_input_data.json`
**참조 파일**: `{outline_dir}/02_explore_research.md` (구성안 Phase 2 산출물 — 주제 개요·트렌드·학습자 분석은 여기서 상속, 중복 수집하지 않음)
**산출물 위치**: `{output_dir}/` (02_explore_plan.md, 02_explore_local.md, 02_explore_nblm.md, 02_explore_web.md, 02_explore_research.md)
**모드**: 탐색적 (orientation) — 다른 강의의 구체적 교안 스크립트 전사 금지 (고착 효과 방지)
**제약**: 총 웹 검색 15회 이내, NBLM 쿼리 노트북당 5회 이내
**워크플로우**: Step 0(계획) → Step 1(로컬) → Step 2(NBLM) → Step 3(웹) → Step 4(통합)
**상세**: `.claude/agents/research-agent/AGENT.md`의 "강의교안 탐색적 리서치" 섹션 참조

### Phase 3: 브레인스토밍 → brainstorm-agent (발문, 활동, 사례 구상)

**지시**: 교안 작성을 위한 브레인스토밍을 수행하세요. 발문 설계(Bloom's 기반), 학습활동 아이디어(GRR×Gagne), 실생활 사례, 형성평가 문항을 발산·수렴하고, SLO-활동-발문-평가 정렬 매트릭스를 생성합니다.
**입력 파일**:
  - `{output_dir}/01_input_data.json` (교수 모델, 활동 전략, 형성평가, Gagne 설정)
  - `{output_dir}/02_explore_research.md` (교수법 리서치 §1~§7 인사이트)
**참조 파일**:
  - `{outline_dir}/03_brainstorm_result.md` (페르소나 상속 — §3에서 로드, 신규 생성 안 함)
  - `{outline_dir}/06_write_lecture_outline.md` (차시별 BOPPPS 구조, SLO)
**산출물 위치**: `{output_dir}/` (03_brainstorm_divergent.md, 03_brainstorm_convergent.md, 03_brainstorm_review.md, 03_brainstorm_result.md)
**워크플로우**: Step 0(맥락 분석) → Step 1(발산) → Step 2(수렴·매핑) → Step 3(다관점 검증) → Step 4(통합)
**상세**: `.claude/agents/brainstorm-agent/AGENT.md`의 "강의교안 브레인스토밍 (Phase 3) 세부 워크플로우" 섹션 참조

**조건부 분기**:
- `questioning_design.include = false` → 축 1(발문) 생략
- `formative_assessment.type = none` → 축 5(평가) 생략
- `teaching_model = pbl` → 축 4(문제 시나리오) 추가
- `gagne_display.mode = none` → Step 2에서 Gagne 매핑 생략, GRR만 수행
- `teaching_model = mixed` → 차시별 교수 모델 분기하여 발문 배치 패턴 개별 적용

### Phase 4: 심화 리서치 → research-agent (브레인스토밍 기반 교수법 사례·발문·활동·평가 심화 수집)

**지시**: 교안 브레인스토밍 결과를 기반으로 심화 리서치를 수행하세요. 교수법 적용 사례, Bloom's 수준별 발문 뱅크, 학습활동/실습 자료, 형성평가 도구를 검증·수집하고, SLO-활동-평가 삼각 정렬을 확인합니다.
**입력 파일**:
  - `{output_dir}/03_brainstorm_result.md` (§11 "Phase 4 심화 리서치 가이드")
  - `{output_dir}/01_input_data.json` (script_settings, instructional_model_map)
  - `{output_dir}/02_explore_research.md` (Phase 2 탐색적 리서치 결과)
**참조 파일**:
  - `{outline_dir}/04_deep_research.md` (구성안 심화 리서치 — CK 수준 상속, 중복 수집하지 않음)
  - `.claude/skills/deep-research/SKILL.md` (8단계 파이프라인 방법론 레퍼런스)
**산출물 위치**: `{output_dir}/` (04_deep_plan.md, 04_deep_local_nblm.md, 04_deep_web.md, 04_deep_research.md)
**모드**: 심화 (deep dive) — 구체적 사례·발문·평가도구 수집 허용, 타강의 교안 스크립트 전사 금지
**제약**: 총 웹 검색 20~25회 이내, NBLM 쿼리 노트북당 3~5회 이내
**워크플로우**: Step 0(범위+계획) → Step 1(로컬+NBLM+웹) → Step 2(삼각측량+검증) → Step 3(합성+비판) → Step 4(산출물)
**상세**: `.claude/agents/research-agent/AGENT.md`의 "강의교안 심화 리서치" 섹션 참조

**조건부 분기**:
- `gagne_display.mode == "none"` → Gagne 사태별 검색 생략, 교수법 패턴에 통합
- `questioning_design.include == false` → 발문 뱅크 검색 예산 축소(1~2회), 활동·평가로 재배분
- `formative_assessment.type == "none"` → 형성평가 도구 검색 예산 축소(1회), 활동으로 재배분
- `teaching_model` 값에 따라 리서치 질문·검색 쿼리 분기 (직접교수법/PBL/플립러닝/혼합)

### Phase 5: 교안 구조 설계 → architecture-agent (도입-전개-정리, Gagne 사태 배치)

**지시**: 차시별 내부 구조를 설계하세요. 교수 모델별 도입-전개-정리 구조, Gagné 사태 배치, 발문 배치, GRR 단계를 분 단위로 설계하고, SLO-활동-발문-형성평가 정렬을 검증합니다.
**입력 파일**:
  - `{output_dir}/01_input_data.json` (script_settings, instructional_model_map, inherited.schedule)
  - `{output_dir}/03_brainstorm_result.md` (§2 발문, §3 활동, §5 형성평가, §6 SLO정렬, §7 Gagné-GRR매핑)
  - `{output_dir}/04_deep_research.md` (§8 Phase 5 활용 가이드: 사례, 평가도구, 발문은행, 시간권장)
**참조 파일**:
  - `{outline_dir}/05_arch_architecture.md` (코스 레벨 구조: Phase 매핑, 정렬 맵 — 상속용)
  - `{outline_dir}/06_write_lecture_outline.md` (차시별 BOPPPS 구조, SLO, 일정)
**산출물 위치**: `{output_dir}/05_arch_lesson_plan.md`
**워크플로우**: Step 0(컨텍스트 로드+교수 모델 결정) → Step 1(레슨 레벨 Backward Design) → Step 2(차시별 내부 구조 설계) → Step 3(3중 검증+산출물 작성)
**검증**: 시간합산(하위 단계 합 = session_minutes) + SLO정렬(커버리지, Bloom's 정합) + Gagné순차(배치 순서, 누락, 최소 시간)
**상세**: `.claude/agents/architecture-agent/AGENT.md`의 "강의교안 레슨 플랜 설계 (Phase 5) 세부 워크플로우" 섹션 참조

**조건부 분기**:
- `gagne_display.mode == "none"` → Gagné 라벨 생략, 도입-전개-정리만 표기, Gagné 순차 검증 SKIP
- `gagne_display.mode == "core_5"` → 핵심 5사태만 명시, 나머지 통합
- `questioning_design.include == false` → 발문 배치 생략, 발문 Bloom's 검증 SKIP
- `formative_assessment.type == "none"` → 형성평가 배치 생략, 형성평가 커버리지 검증 SKIP
- `time_ratio.source == "manual"` → 사용자 지정 비율 적용 (auto 시 교수 모델별 기본값 + Phase 보정)
- `teaching_model == "mixed"` → 차시별 교수 모델 자동 결정 (Phase A→직접교수법, C/D→PBL/플립러닝)

### Phase 6: 교안 작성 → writer-agent (섹션별 스크립트, 발문, 활동, 평가문항)

<!-- TODO: Phase 6 구현 예정 -->

### Phase 7: 품질 검토 → review-agent (목표-활동-평가 정렬, 시간 배분)

<!-- TODO: Phase 7 구현 예정 -->

## 산출물 (02_script/)

```
lectures/YYYY-MM-DD_{강의명}/02_script/
├── 01_input_data.json              # Phase 1: 구성안 로드 + 교안 설정
├── 02_explore_plan.md             # Phase 2: 탐색적 리서치 계획
├── 02_explore_research.md         # Phase 2: 리서치 결과
├── 03_brainstorm_divergent.md    # Phase 3: 발산적 탐색 (Step 1)
├── 03_brainstorm_convergent.md   # Phase 3: 수렴 및 매핑 (Step 2)
├── 03_brainstorm_review.md       # Phase 3: 다관점 검증 (Step 3)
├── 03_brainstorm_result.md        # Phase 3: 브레인스토밍 최종 통합 ★ (Step 4)
├── 04_deep_plan.md                # Phase 4: 심화 리서치 계획
├── 04_deep_research.md            # Phase 4: 심화 리서치 최종 ★
├── 05_arch_lesson_plan.md         # Phase 5: 차시별 레슨 플랜 구조 설계
├── 06_write_script_draft.md       # Phase 6: 교안 초안 (중간)
├── 06_write_lecture_script.md     # Phase 6: 최종 교안 ★
└── 07_review_quality.md           # Phase 7: 품질 검토
```
