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

<!-- TODO: Phase 3 구현 예정 -->

### Phase 4: 심화 리서치 → research-agent (브레인스토밍 기반 예시·보충 콘텐츠 수집)

<!-- TODO: Phase 4 구현 예정 -->

### Phase 5: 교안 구조 설계 → architecture-agent (도입-전개-정리, Gagne 사태 배치)

<!-- TODO: Phase 5 구현 예정 -->

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
├── 03_brainstorm_result.md        # Phase 3: 브레인스토밍 결과
├── 04_deep_research.md            # Phase 4: 심화 리서치
├── 05_arch_lesson_plan.md         # Phase 5: 차시별 레슨 플랜 구조 설계
├── 06_write_script_draft.md       # Phase 6: 교안 초안 (중간)
├── 06_write_lecture_script.md     # Phase 6: 최종 교안 ★
└── 07_review_quality.md           # Phase 7: 품질 검토
```
