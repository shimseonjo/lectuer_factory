---
name: lecture-script
description: 강의교안 생성 - 차시별 반복 파이프라인 (입력수집 → 탐색리서치 → 브레인스토밍 → 심화리서치 → 구조설계 → [차시작성+검토 → 모듈통합+검토] → 최종통합+검토)
context: fork
allowed-tools: Agent, Read, Write, Glob, Grep, WebSearch, WebFetch, AskUserQuestion, mcp__Context7__resolve-library-id, mcp__Context7__get-library-docs
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

### Phase 6: 차시별 반복 작성 + 4시간 모듈 통합 (3계층 아키텍처)

Phase 5에서 전체 차시 레슨 플랜이 확정된 후, 차시별로 순차 작성+검토하고 4시간 단위로 통합한다.

**3계층 구조**: Micro(차시) → Meso(4시간 모듈) → Macro(전체 교안)

**모듈 경계 결정**: `sessions_per_module = ceil(sessions_per_day / 2)` (반일 단위). Phase 전환점(A→B 등)은 모듈 경계로 보정.

**공통 입력 파일**:
  - `{output_dir}/05_arch_lesson_plan.md` (분 단위 레슨 플랜)
  - `{output_dir}/01_input_data.json` (script_settings, inherited)
  - `{output_dir}/03_brainstorm_result.md` (발문, 활동, 사례, 평가)
  - `{output_dir}/04_deep_research.md` (활용 가이드)

**공통 참조 파일**:
  - `{outline_dir}/06_write_lecture_outline.md` (코스 개요, 학습 목표, 시간표)
  - `{outline_dir}/05_arch_architecture.md` (차시 간 연결 맵)

**템플릿**: `.claude/templates/script-template.md`

**조건부 분기** (모든 Phase 6 하위 단계에 공통 적용):
- `script_detail_level` → full_script / semi_structured / bullet_notes별 밀도 조절
- `teaching_model = mixed` → 차시별 교수 모델에 따라 도입-전개-정리 구조 개별 적용
- `gagne_display.mode` → all_9 / core_5 / none별 라벨 표기 수준
- `questioning_design.include = false` → 발문 블록 전체 생략
- `questioning_design.include_expected_answers = true` → 발문에 예상 답변 포함
- `formative_assessment.type = none` → 형성평가 블록 생략
- `target_scope.type = "day"` → 지정 차시만 작성, 나머지 "scope 외" 표시

**상세**: `.claude/agents/writer-agent/AGENT.md`의 "강의교안 차시별 작성", "모듈 통합", "최종 통합" 섹션 참조

#### Phase 6 오케스트레이션 루프

```python
# Phase 5 산출물에서 차시/모듈 구조 파싱
lesson_plan = read("{output_dir}/05_arch_lesson_plan.md")
all_sessions = parse_sessions(lesson_plan)
modules = group_into_modules(all_sessions, sessions_per_module)
init_file("{output_dir}/_running_summary.md")

for module in modules:
    for session in module.sessions:
        # === Phase 6a: 차시 작성 ===
        Agent → writer-agent (session_write 모드)
          입력: lesson_plan[session], _running_summary.md,
                06_session_{prev}.md (전환 멘트용),
                brainstorm, deep_research, input_data
          산출물: {output_dir}/06_sessions/06_session_{NNN}.md
          부수작업: _running_summary.md에 차시 요약 append

        # === Phase 6b: 차시 경량 검토 ===
        Agent → review-agent (session_review 모드)
          입력: 06_session_{NNN}.md, lesson_plan[session], input_data
          검토: 시간합산(±1분), SLO커버리지, Bloom's점진 (3항목)
          판정: PASS / FAIL

        IF FAIL:
            Agent → writer-agent (session_revise 모드)
              입력: 06_session_{NNN}.md, review_feedback
            Agent → review-agent (session_review 모드)  # 2차 검토
            IF 2차 FAIL: log_warning → 모듈 검토에서 처리

    # === Phase 6c: 모듈 통합 ===
    Agent → writer-agent (module_integrate 모드)
      입력: 06_session_{*}.md (해당 모듈 차시들),
            06_module_{prev}.md (이전 모듈, 연결용),
            lesson_plan, _running_summary.md
      작업: 병합 + 일관성 패치 + Synthesizer 삽입
      산출물: {output_dir}/06_modules/06_module_{NN}.md
      부수작업: _running_summary.md를 모듈 수준으로 리셋

    # === Phase 6d: 모듈 검토 ===
    Agent → review-agent (module_review 모드)
      입력: 06_module_{NN}.md, lesson_plan, input_data
      검토: 차시간연결성, 톤일관성, 난이도점진, 모듈SLO커버리지 (4항목)
      판정: PASS / FAIL
      산출물: {output_dir}/06_modules/06_module_{NN}_review.md

    IF FAIL:
        Agent → writer-agent (module_revise 모드)
          입력: 06_module_{NN}.md, module_review_feedback
```

#### Phase 6a: 차시별 교안 작성 → writer-agent (session_write 모드)

**산출물**: `{output_dir}/06_sessions/06_session_{NNN}.md` (예: 06_session_001.md)
**작성 범위**: 정확히 1개 차시. script-template.md의 차시 섹션 형식.
**Running Summary**: 작성 완료 시 핵심 2~3문장을 `_running_summary.md`에 append
  - (1) 다룬 핵심 개념, (2) 마지막 전환 멘트/예고, (3) 학습자 산출물

#### Phase 6b: 차시별 경량 검토 → review-agent (session_review 모드)

| 검토 항목 | 기준 | FAIL 시 |
|---------|------|---------|
| 시간 합산 | 도입+전개+정리 = session_minutes (±1분) | writer 1회 재작성 |
| SLO 커버리지 | 해당 차시 SLO가 전개 활동에 매핑 | writer 1회 재작성 |
| Bloom's 점진 | 도입(저)→전개(중)→정리(중~고) | writer 1회 재작성 |

**재시도**: 최대 1회. 2차 FAIL → 경고 기록 후 진행.

#### Phase 6c: 4시간 모듈 통합 → writer-agent (module_integrate 모드)

**발동**: 모듈 내 모든 차시가 6a+6b 통과 후 자동
**작업**: (1) 차시 병합 (2) 일관성 패치(용어, 전환 멘트, tone_examples) (3) Synthesizer 삽입(모듈 말미 종합 요약) (4) Running Summary 모듈 수준 리셋
**산출물**: `{output_dir}/06_modules/06_module_{NN}.md`

#### Phase 6d: 모듈 수준 검토 → review-agent (module_review 모드)

| 검토 항목 | 기준 |
|---------|------|
| 차시 간 연결성 | 전환 멘트 존재, 산출물 연쇄 |
| 톤 일관성 | 모듈 내 문체/용어 통일 |
| 난이도 점진 | Bloom's 수준 상승 패턴 |
| 모듈 SLO 커버리지 | 해당 반일 SLO 전체 커버 |

**FAIL 시**: writer-agent 모듈 수정 1회. 2차 FAIL → 경고 기록 후 진행.
**산출물**: `{output_dir}/06_modules/06_module_{NN}_review.md`

### Phase 7: 최종 전체 통합 + 품질 검토

#### Phase 7a: 전체 통합 → writer-agent (final_integrate 모드)

**지시**: 모든 모듈을 최종 강의교안으로 통합하세요.
**입력 파일**:
  - `{output_dir}/06_modules/06_module_*.md` (모든 모듈 통합본)
  - `{output_dir}/01_input_data.json`
  - `{output_dir}/05_arch_lesson_plan.md`
**참조 파일**: `{outline_dir}/06_write_lecture_outline.md`
**작업**: (1) 모든 모듈 병합 (2) 코스 레벨 섹션 추가(메타데이터, 개요, 학습 목표, 시간표, 정렬 매트릭스, 부록) (3) 전체 발문 인덱스 생성
**산출물**: `{output_dir}/06_write_lecture_script.md`
**상세**: `.claude/agents/writer-agent/AGENT.md`의 "최종 통합" 섹션 참조

#### Phase 7b: 최종 품질 검토 → review-agent (script_full_review 모드)

**지시**: 최종 강의교안과 전체 파이프라인 산출물을 독립적 외부 검토자 관점에서 품질 검증하세요.
**입력 파일**:
  - `{output_dir}/06_write_lecture_script.md` (주 검토 대상)
  - `{output_dir}/01_input_data.json`
  - `{output_dir}/05_arch_lesson_plan.md`
  - `{output_dir}/03_brainstorm_result.md`
  - `{output_dir}/04_deep_research.md`
**참조 파일**: `{outline_dir}/06_write_lecture_outline.md` (코스 레벨 정합성)
**산출물 위치**: `{output_dir}/07_review_quality.md`
**검토 프레임워크**: QM Rubric 7th Ed. (기준 3·5 중심) + 5영역 가중치 체크리스트(20/25/20/20/15)
**검토 초점**: 하위 레벨(차시·모듈)에서 이미 통과한 항목은 경량 확인, **교차 모듈 정합성**에 집중
**판정 체계**: APPROVED(8.0+) / APPROVED WITH NOTES(6.0~7.9) / REVISE(4.0~5.9) / REVISE MAJOR(0~3.9)
**자동 REVISE**: QM 기준 3·5 미충족, 영역 1(정렬)·2(전개 완성도) 평균 5.0 미만
**워크플로우**: Step 0(컨텍스트 로드) → Step 1(QM 적합성) → Step 2(5영역 심층) → Step 3(판정+작성)
**상세**: `.claude/agents/review-agent/AGENT.md`의 "강의교안 품질 검토" 섹션 참조

**조건부 분기**:
- `questioning_design.include == false` → 영역 3(발문 20%) 가중치를 영역 2(+10%), 영역 4(+10%)로 재배분
- `formative_assessment.type == "none"` → 형성평가 관련 항목(1-3, 4-5) SKIP
- `gagne_display.mode == "none"` → Gagné 완성도 항목(2-3) SKIP
- `teaching_model` → 교수 모델별 구조 검증 기준 분기 (직접교수법/PBL/플립러닝/혼합)

### Phase 7b 이후: 판정 분기 처리

Phase 7b 완료 후, `07_review_quality.md`를 읽어 판정에 따라 분기한다.

```
revision_count = 0

[Phase 7b 완료]
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
  │     → IF restart_phase == 7a:
  │         Agent → writer-agent (final_integrate 수정 모드)
  │     → IF restart_phase == 6:
  │         해당 모듈/차시 재작성 (6a~6d 루프 부분 재실행)
  │     → IF restart_phase == 5:
  │         Agent → architecture-agent (수정 모드) → Phase 6 전체 재실행
  │     → IF restart_phase == 4:
  │         Agent → research-agent (보강 모드) → architecture-agent → Phase 6 전체 재실행
  │
  │     → Phase 7 재실행 (7a → 7b)
  │     → 분기 로직 처음으로 (revision_count 확인)
```

**수정 모드 지시 템플릿**:

**writer-agent final_integrate 수정 모드 (교안)**:
```
지시: 07_review_quality.md의 수정 지시를 반영하여 06_write_lecture_script.md를 수정하세요.
규칙: (1) P0·P1 반드시 반영 (2) P2 가능 범위 반영 (3) §6-3 강점 보호
      (4) 수정 범위 외 섹션 변경 금지 (5) 06_write_lecture_script.md 직접 수정
입력: 07_review_quality.md (추가), 06_write_lecture_script.md, 05_arch_lesson_plan.md, 01_input_data.json
```

**architecture-agent 수정 모드 (교안)**:
```
지시: 07_review_quality.md §6-2의 Phase 5 수정 지시를 반영하여 05_arch_lesson_plan.md를 수정하세요.
규칙: (1) 지적된 구조적 문제만 수정 (2) 3중 검증 재실행 (3) 05_arch_lesson_plan.md 덮어쓰기
입력: 07_review_quality.md, 05_arch_lesson_plan.md, 01_input_data.json, 03_brainstorm_result.md, 04_deep_research.md
```

**research-agent 보강 모드 (교안)**:
```
지시: 07_review_quality.md §6-2의 Phase 4 수정 지시에 따라 부족한 자료(발문/활동/평가)를 보강하세요.
규칙: (1) 전면 재실행 아닌 보강(incremental) (2) 추가 웹 검색 5~10회 이내
      (3) 04_deep_research.md에 보강 섹션 추가(append)
입력: 07_review_quality.md, 04_deep_research.md, 01_input_data.json
```

**최대 재시도**: 1회 (원본 + 수정 = 총 2회). 2차 REVISE 시 사용자 개입 요청.

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
├── _running_summary.md            # Phase 6: 누적 요약 (차시·모듈 작성 시 갱신)
├── 06_sessions/                   # Phase 6a: 차시별 교안 (NEW)
│   ├── 06_session_001.md
│   ├── 06_session_002.md
│   └── ...
├── 06_modules/                    # Phase 6c: 4시간 모듈 통합 (NEW)
│   ├── 06_module_01.md
│   ├── 06_module_01_review.md     # Phase 6d: 모듈 검토
│   ├── 06_module_02.md
│   └── ...
├── 06_write_lecture_script.md     # Phase 7a: 최종 통합 교안 ★
└── 07_review_quality.md           # Phase 7b: 최종 품질 검토
```
