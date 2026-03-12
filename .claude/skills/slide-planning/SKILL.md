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

**지시**: 슬라이드 기획을 위해 교안을 로드하고 슬라이드 설정을 수집하세요.
**산출물 위치**: `{output_dir}/01_input_data.json`
**상세**: `.claude/agents/input-agent/AGENT.md`의 "슬라이드 기획 입력 수집" 섹션 참조

**input-agent 실행 절차**:

1. **강의 폴더 탐색**: `lectures/` 폴더를 Glob으로 스캔 → 최신순 정렬 → AskUserQuestion으로 선택
   - 폴더 없음/0개 → 오류 메시지 출력 후 종료
   - 폴더 1개: 해당 폴더 + "기타 직접 입력" = 2개 선택지
   - 폴더 2개: 2개 + "기타 직접 입력" = 3개 선택지
   - 폴더 3개+: 최신 3개 + "기타 직접 입력" = 4개 선택지
   - 선택된 폴더의 `02_script/06_write_lecture_script.md` + `02_script/06_modules/` 존재 확인 (없으면 오류 후 종료)

2. **2계층 교안 자동 파싱 + 자동 추론**: 사용자 개입 없음
   - Layer 1: `06_write_lecture_script.md` 상단 ~200줄 → 메타데이터·SLO·시간표·정렬 매트릭스
   - Layer 2: `06_modules/06_module_{NN}.md` × N개 → 차시별 상세 콘텐츠
   - Layer 3: `01_input_data.json` (교안·구성안) → 보조 설정
   - 콘텐츠 유형 자동 탐지 (has_code/activity/quiz)
   - 자동 추론 5개: PQ1(교안 범위 동기화), PQ5(톤 키워드 매칭 → design_tone), PQ6(발표자 노트 항상 포함), PQ7(코드 테마 dark 자동), PQ4(Step 3에서 PQ3 확정 후 밀도→A-E 파생)

3. **PQ2+PQ3 수집**: AskUserQuestion 1회 (2개 묶음)
   - PQ2: 슬라이드 도구 — {자동추천}(추천) / Marp / Slidev / Gamma
   - PQ3: 정보 밀도 — 교육용 표준(추천) / 고밀도 참조형 / 프레젠테이션형
   - PQ3 확정 → PQ4 자동 파생 (밀도→A-E 결정적 매핑)

4. **자동 추론 확인 게이팅**: AskUserQuestion
   - "자동 설정 그대로 진행"(추천) / "세부 설정 조정"
   - "세부 설정" → PQ4(A-E) + PQ5(톤) + PQ7(코드 테마) 추가 질문 1회

5. **output_dir 결정**: `lectures/{선택한 강의 폴더}/03_slide_plan/`
   - 폴더 생성 (없으면)
   - `01_input_data.json` 저장 (inherited + slide_settings + derived + source + metadata)
   - Phase 1 완료 검증 (설계 문서 §I 기준: 필수 필드 존재, modules 배열 비어있지 않음, enum 유효성)
   - 수집된 설정 요약 출력

### Phase 2: 브레인스토밍 → brainstorm-agent (시각화 아이디어, 레이아웃 구상)
### Phase 3: 슬라이드 구조 설계 → architecture-agent (슬라이드 수, 유형, 순서, 시간 배분)
### Phase 4: 기획안 작성 → writer-agent (슬라이드별 목적, 레이아웃, 핵심 콘텐츠)
### Phase 5: 품질 검토 → review-agent (정보 밀도, 시각 계층, 학습목표 정렬)

## 산출물 (03_slide_plan/)

```
lectures/YYYY-MM-DD_{강의명}/03_slide_plan/
├── 01_input_data.json               # Phase 1: 교안 로드 + 도구/밀도 설정
├── 03_brainstorm_divergent.md       # Phase 2: 시각화 아이디어 발산 (중간)
├── 03_brainstorm_convergent.md      # Phase 2: 수렴·패턴 매핑 (중간)
├── 03_brainstorm_result.md          # Phase 2: 브레인스토밍 최종 ★
├── 05_arch_slide_structure.md       # Phase 3: 슬라이드 구조 설계
├── 06_write_slide_plan.md           # Phase 4: 최종 기획안 ★
└── 07_review_quality.md             # Phase 5: 품질 검토
```
