# 강의교안 워크플로우 Phase 1 입력 수집 설계안

## 설계 개요

- **작성일**: 2026-03-11
- **버전**: v3 (외부 설계안 P1·P2·P3 통합)
- **목적**: `/lecture-script` 워크플로우의 Phase 1(입력 수집) 상세 설계
- **담당 에이전트**: input-agent
- **참조 패턴**: 강의구성안 Phase 1 UX 패턴 준수

**v3 주요 변경사항**:
- 필수 질문 2개 → 6개 확장 (교수 모델, 활동 전략, 형성평가, 시간 비율 추가)
- AskUserQuestion 2회 묶음 호출 (3+3) 도입
- 교수 모델→교수설계 모델 매핑 테이블 신설 (§E)
- pedagogy→교수 모델 자동 추론 규칙 신설 (§F)
- Phase 5~6 전달 원칙 신설 (§G)
- Phase 1 완료 검증 기준 신설 (§H)

---

## A. 입력 수집 항목 설계

### A-1. 구성안 자동 로드 항목

강의교안은 강의구성안의 후속 워크플로우이므로, `01_outline/06_write_lecture_outline.md`에서
이미 확정된 정보를 자동으로 파싱한다. 이 항목들은 사용자에게 **재질문하지 않는다**.

| 분류 | 파싱 소스 | 파싱 항목 | input_data.json 키 |
|------|----------|----------|-------------------|
| 기본 메타데이터 | 구성안 §메타데이터 | 강의 주제 | `topic` |
| 기본 메타데이터 | 구성안 §메타데이터 | 대상 학습자 (설명, 수준) | `target_learner` |
| 기본 메타데이터 | 구성안 §메타데이터 | 강의 형태 (방식, 공간) | `format` |
| 기본 메타데이터 | 구성안 §메타데이터 | 시간 구성 | `schedule` |
| 기본 메타데이터 | 구성안 §메타데이터 | 교수 전략 | `pedagogy` |
| 기본 메타데이터 | 구성안 §메타데이터 | 톤·스타일 | `tone` |
| 학습 목표 | 구성안 §2 학습 목표 | 목표 목록 + Bloom's 수준 | `learning_goals` |
| 핵심 질문 | 구성안 §3 핵심 질문 | Essential Questions 목록 | `essential_questions` |
| 평가 체계 | 구성안 §4 평가 체계 | 총괄/형성 평가 방식 | `assessment` |
| 차시 구조 | 구성안 §5 강의 로드맵 | Phase A~D 구조, 일별 개요 | `course_structure` |
| 차시 상세 | 구성안 §6 차시 상세 | 차시별 주제, BOPPPS 구조 | `sessions` |
| 선수 지식 | 구성안 `01_input_data.json` | prerequisites | `prerequisites` |
| 제외 범위 | 구성안 `01_input_data.json` | exclusions | `exclusions` |
| 키워드 | 구성안 `01_input_data.json` | keywords | `keywords` |

**파싱 전략**:
- `06_write_lecture_outline.md` 우선 파싱 (최신 확정본)
- 구성안 `01_input_data.json`에서 보완 (keywords, prerequisites 등 구성안에 원문으로 저장된 항목)
- 두 파일 모두 존재해야 진행 가능. 누락 시 오류 보고 후 종료

---

### A-2. 추가 질문 항목

구성안에 없고 교안 작성에 실질적으로 영향을 미치는 항목만 추가 수집한다.

#### 필수 질문 (6개) — AskUserQuestion 2회 호출 (3+3)

**[1회차: SQ1 + SQ1a + SQ1b]** — 범위·교수법·활동

| # | 카테고리 | 질문 | 이유 |
|---|---------|------|------|
| SQ1 | 교안 대상 차시 범위 | **교안을 전체 차시에 작성할까요, 특정 차시만 작성할까요?** | 전체 차시 vs 특정 차시 선택. 차시 수가 많으면 일부만 작성이 현실적 |
| SQ1a | 교수 모델 | **교안의 기본 교수 모델을 선택하세요.** | 교안 구조(도입-전개-정리)가 교수 모델에 따라 근본적으로 달라짐 |
| SQ1b | 활동 전략 | **교안에 포함할 학습 활동 유형을 선택하세요.** (복수 선택) | Phase 3 브레인스토밍과 Phase 6 작성의 방향 결정 |

**SQ1 선택지** (AskUserQuestion):
```
1. 전체 차시 (모두 작성) [추천]
2. 특정 차시만 선택 → 후속 질문으로 Day 번호 입력
```

**결정 근거**: 강의구성안의 총 교시 수가 20~60교시에 달할 수 있으므로, 교안 작성 범위를 먼저 확정해야 산출물 규모와 소요 시간을 예측할 수 있다. 전체를 선택하면 분할 작업(chunk) 전략이 활성화된다.

**SQ1a 선택지** (AskUserQuestion):
```
1. {추론 결과} (추천) — pedagogy에서 자동 추론된 모델
2. 직접교수법 — 교수자가 체계적으로 설명→시범→연습 (Hunter 6단계)
3. PBL — 학습자가 실세계 문제를 팀으로 해결 (문제→탐구→해결→발표)
4. 플립러닝 — 사전학습 후 교실에서 심화 실습/토론 (Before/During/After)
```

- 추론 결과가 첫 번째 선택지로 표시됨 (§F 추론 규칙 참조)
- "혼합(차시별 상이)"은 AskUserQuestion 4개 선택지 제약으로 Other 입력으로 지원
- Other 선택 시 → 차시별 교수 모델 개별 지정 안내 (Phase 5에서 자동 결정 가능)

**결정 근거**: 구성안의 `pedagogy`는 자유 텍스트이므로 교수 모델이 명시적으로 구조화되지 않음. 교안의 도입-전개-정리 구조, GRR 적용, Bloom's 발문 수준이 교수 모델에 직접 의존하므로, 구조화된 선택이 필요하다. 자동 추론 후 사용자 확인 방식으로 마찰 최소화.

**SQ1b 선택지** (AskUserQuestion, `multiSelect: true`):
```
1. 개인 실습 — I Do→You Do 구조의 개인 연습
2. 그룹 활동 — 소그룹 협업, 페어 프로그래밍
3. 토론·발문 — Bloom's 기반 발문, 소크라테스식 질의
4. 프로젝트 — 차시를 관통하는 누적 산출물
```

- 기본 선택: pedagogy에서 추론 (§F-2 참조). 추론 실패 시 `["개인 실습", "그룹 활동"]`
- 복수 선택 허용

**결정 근거**: 활동 전략이 Phase 3 브레인스토밍의 아이디어 방향과 Phase 6 작성의 활동 구성에 직접 영향. 단일 선택이 아닌 복수 선택으로 교안의 활동 다양성 확보.

---

**[2회차: SQ2 + SQ3 + SQ4]** — 상세도·평가·시간

| # | 카테고리 | 질문 | 이유 |
|---|---------|------|------|
| SQ2 | 스크립트 상세도 | **교안의 스크립트 상세도를 선택하세요.** | 교수자 경험에 따라 상세도가 달라짐. 분량과 형식 결정 |
| SQ3 | 형성평가 유형 | **수업 중 형성평가를 어떻게 활용하나요?** | SLO-평가 정렬을 Phase 5에서 설계하려면 유형을 먼저 확정해야 함 |
| SQ4 | 시간 비율 | **도입·전개·정리 시간 비율을 확인하세요.** | 교수 모델 기반 자동 설정 후 사용자 확인. Phase 5 시간표 설계의 기준 |

**SQ2 선택지** (AskUserQuestion):
```
1. 완전 스크립트 (발화할 모든 내용을 문장으로 기술. 신규 강사, 비전문가용)
2. 반구조화 스크립트 (단락별 요점 + 주요 문장. 경험 1~3년 강사) [추천]
3. 불릿 노트 (주요 포인트 3~5개/섹션. 경험 강사, 전문가)
```

**결정 근거**: 이 선택이 교안 작성 에이전트(writer-agent)의 스크립트 생성 방식을 결정한다. 구성안의 `pedagogy`에 교수 스타일이 명시되어 있지만, 스크립트 "상세도"는 별도 차원이다. 기본값을 반구조화로 설정하여 빠른 진행 가능.

**SQ3 선택지** (AskUserQuestion):
```
1. 섹션별 체크 (각 전개 섹션 끝에 형성평가 1개 삽입) [추천]
2. 차시별 Exit Ticket (각 차시 정리 단계에서 Exit Ticket만 배치)
3. 실습 통합 평가 (실습 과제 자체를 형성평가로 활용)
4. 평가 없음 (형성평가 삽입하지 않음)
```

**SLO 자동 배정 원칙**:
- "섹션별 체크" 또는 "차시별 Exit Ticket" 선택 시 → Phase 5 architecture-agent가 각 SLO에 최소 1개의 형성평가 지점을 자동 배정
- 배정 근거: 구성안의 SLO 목록과 차시별 학습 목표 매핑
- Phase 7 review-agent가 "SLO별 평가 커버리지 100%" 검증

**결정 근거**: 형성평가는 교안의 핵심 구성요소. 유형에 따라 Phase 5의 구조 설계와 Phase 6의 평가 문항 작성 방식이 달라짐. "섹션별 체크"를 기본값으로 하여 SLO-평가 정렬을 최대화.

**SQ4 선택지** (AskUserQuestion):
```
1. 교수 모델 기반 자동 설정 (SQ1a 선택에 따라 최적 비율 적용) [추천]
2. 직접 입력 → Other로 "15:70:15" 형태 입력
```

**교수 모델별 자동 비율**:

| 교수 모델 (SQ1a) | 도입 | 전개 | 정리 | 근거 |
|----------------|------|------|------|------|
| 직접교수법 | 10% | 60% | 30% | Hunter 모델: 복습+제시+안내연습에 집중, 독립연습 포함 |
| PBL | 10% | 75% | 15% | 문제 탐구+해결에 최대 시간, 발표+성찰 정리 |
| 플립러닝 | 5% | 80% | 15% | 사전학습 전제로 도입 최소화, 교실 활동 극대화 |
| 혼합 | 10% | 70% | 20% | 범용 기본값 |

- 사용자가 "직접 입력"을 선택하면 교수 모델 자동값을 무시
- 혼합 교수법에서 차시별 비율이 다를 경우 → Phase 5 architecture-agent가 차시별 개별 결정

---

#### 선택 질문 (3개) — 기본값 존재. "추가 설정이 필요하십니까?" 확인 후 진행

| # | 카테고리 | 질문 | 기본값 | 이유 |
|---|---------|------|--------|------|
| SQ5 | Gagne 9사태 적용 수준 | **각 차시에서 Gagne 9가지 수업사태를 얼마나 명시적으로 적용할까요?** | 핵심 사태만 (1·2·3·6·9) | 9사태 전체를 모든 차시에 명시하면 교안이 과도하게 형식화됨. 핵심 5개만 명시하는 것이 실용적. 4(자극제시)·5(학습안내)는 전개부 본문 흐름에 내재화 |
| SQ6 | 발문 설계 포함 여부 | **교안에 Bloom's 수준별 발문을 포함할까요?** | 포함 (차시당 3~5개, 기본 4개) | 교안의 핵심 가치 중 하나. 기본 포함. 포함하지 않으면 브레인스토밍 단계 발문 설계 스킵 |
| SQ7 | 실습 가이드 상세도 | **실습 활동의 단계별 가이드를 얼마나 상세하게 작성할까요?** | 단계 목록 + 핵심 지시 | 실습 비율이 높은 교안(pedagogy에 실습 50%+ 명시된 경우)에서 중요. 완전 가이드는 별도 핸드아웃으로 분리하는 것이 일반적 |

**SQ5 선택지**:
```
1. 전체 9사태 명시 (모든 단계를 교안에 라벨링)
2. 핵심 5사태만 (1.주의획득·2.목표고지·3.선수학습회상·6.수행유도·9.파지와전이촉진) [기본값]
   ※ 4(자극제시)·5(학습안내)는 전개부 본문 흐름에 내재화됨
3. 라벨 없음 (흐름으로만 작성, 사태는 내재화)
```

**SQ6 선택지**:
```
1. 포함 — Bloom's 수준 태깅 + 예상 답변 포함
2. 포함 — 발문만 (예상 답변 제외) [기본값]
3. 제외
```

**SQ7 선택지**:
```
1. 완전 가이드 (학습자가 읽으며 따라할 수 있는 수준)
2. 단계 목록 + 핵심 지시 [기본값]
3. 활동 제목과 소요 시간만
```

---

### A-3. 항목 설계 결정 근거: 제외한 항목들

| 항목 | 제외 이유 |
|------|----------|
| 학습자 수준 | 구성안 `target_learner.level`에서 로드 |
| 평가 전략 | 구성안 §4 평가 체계에서 파싱 |
| GRR 비율 (I Do/We Do/You Do) | SQ1a 교수 모델에서 자동 파생 (§E 매핑 테이블) |
| 사용 도구/매체 | Phase 3 브레인스토밍에서 도출하는 것이 적합 (현 단계에서 결정하기 이르다) |
| 참고 자료 추가 | 구성안 Phase 1에서 이미 수집됨. 추가 자료는 Phase 2에서 처리 |

---

## B. input_data.json 스키마 설계

```json
{
  "$schema": "02_script/01_input_data.json 스키마 — 강의교안 워크플로우 input-agent 생성",

  "source": {
    "outline_dir": "string — 구성안 폴더 경로 (예: lectures/2026-03-10_딥러닝-기초/01_outline)",
    "outline_file": "string — 구성안 최종 파일 경로 (06_write_lecture_outline.md)",
    "outline_input_json": "string — 구성안 01_input_data.json 경로 (예: lectures/.../01_outline/01_input_data.json)"
  },

  "inherited": {
    "topic": "string — 구성안에서 로드",
    "target_learner": {
      "description": "string",
      "level": "입문 | 중급 | 고급"
    },
    "format": {
      "type": "강의 | 워크숍 | 세미나",
      "mode": "오프라인 | 온라인 | 혼합"
    },
    "schedule": {
      "preset": "단일 강의 | 다회차 강의 | 집중 워크숍",
      "days": "number",
      "hours_per_day": "number",
      "session_minutes": "number",
      "break_minutes": "number",
      "total_sessions": "number — 총 교시 수 (자동 산출)"
    },
    "pedagogy": "string",
    "tone": "string",
    "learning_goals": [
      {
        "id": "number",
        "text": "string",
        "blooms_level": "기억 | 이해 | 적용 | 분석 | 평가 | 창조",
        "priority": "핵심 | 중요 | 보충"
      }
    ],
    "essential_questions": ["string"],
    "assessment": {
      "summative": [
        {
          "method": "string",
          "related_goals": ["number"],
          "weight": "string"
        }
      ],
      "formative": [
        {
          "timing": "string",
          "method": "string",
          "related_goals": ["number"]
        }
      ]
    },
    "keywords": ["string"],
    "prerequisites": "string | null",
    "exclusions": "string | null",
    "course_structure": {
      "phases": [
        {
          "id": "A | B | C | D",
          "label": "string",
          "ratio": "string",
          "session_count": "number",
          "theory_practice_ratio": "string"
        }
      ],
      "days": [
        {
          "day": "number",
          "session_range": "string",
          "theme": "string",
          "phase": "string",
          "daily_output": "string"
        }
      ]
    },
    "sessions": [
      {
        "day": "number",
        "session": "number",
        "title": "string",
        "learning_goals": ["number"],
        "phase": "A | B | C | D",
        "boppps": {
          "bridge_in": "string",
          "outcomes": "string",
          "pre_assessment": "string",
          "participatory_learning": "string",
          "post_assessment": "string",
          "summary": "string"
        }
      }
    ]
  },

  "script_settings": {
    "target_scope": {
      "type": "전체 | day | session",
      "value": "all | number | number — SQ1 응답 결과",
      "session_ids": ["number — 작성 대상 교시 번호 목록 (자동 산출)"]
    },
    "teaching_model": {
      "selected": "direct_instruction | pbl | flipped | mixed",
      "label": "직접교수법 | PBL | 플립러닝 | 혼합",
      "inferred_from": "string — pedagogy 원문에서 추론된 값",
      "confidence": "high | medium | low"
    },
    "mixed_model_map": {
      "_comment": "teaching_model이 'mixed'일 때만 사용. Day별 교수 모델 매핑",
      "example": { "1": "direct_instruction", "2": "pbl", "3": "flipped" }
    },
    "activity_strategies": ["individual_practice | group_activity | discussion | project"],
    "script_detail_level": {
      "level": "full_script | semi_structured | bullet_notes",
      "label": "완전 스크립트 | 반구조화 스크립트 | 불릿 노트",
      "default": "semi_structured"
    },
    "formative_assessment": {
      "type": "sectional_check | exit_ticket | practice_integrated | none",
      "label": "섹션별 체크 | 차시별 Exit Ticket | 실습 통합 평가 | 평가 없음",
      "default": "sectional_check",
      "slo_coverage_rule": "type ≠ 'none'이면, Phase 5에서 각 SLO에 최소 1개 형성평가 지점 자동 배정"
    },
    "time_ratio": {
      "intro": "number — 도입 비율(%)",
      "main": "number — 전개 비율(%)",
      "wrap": "number — 정리 비율(%)",
      "source": "auto | manual — 교수 모델 기반 자동 / 사용자 직접 입력"
    },
    "gagne_display": {
      "mode": "all_9 | core_5 | none",
      "label": "전체 9사태 | 핵심 5사태만 | 라벨 없음",
      "default": "core_5",
      "core_events": [1, 2, 3, 6, 9],
      "core_event_labels": {
        "1": "주의획득 (Gain Attention)",
        "2": "목표고지 (Inform Objectives)",
        "3": "선수학습회상 (Stimulate Recall)",
        "6": "수행유도 (Elicit Performance)",
        "9": "파지와전이촉진 (Enhance Retention & Transfer)"
      },
      "note": "4(자극제시)·5(학습안내)는 전개부 본문 흐름에 내재화. architecture-agent는 별도 라벨 없이 구현"
    },
    "questioning_design": {
      "include": "boolean — 발문 포함 여부",
      "include_expected_answers": "boolean — 예상 답변 포함 여부",
      "questions_per_session": "number — 차시당 목표 발문 수 (범위 3~5, 기본값 4)",
      "default": {
        "include": true,
        "include_expected_answers": false,
        "questions_per_session": 4
      }
    },
    "practice_guide_detail": {
      "level": "full | step_list | title_only",
      "label": "완전 가이드 | 단계 목록+핵심 지시 | 제목+소요시간",
      "default": "step_list"
    },
    "instructional_model_map": {
      "_comment": "teaching_model에서 자동 파생. Phase 5~6 에이전트가 참조 (§E 매핑 테이블 참조)",
      "primary_model": "string — 주 교수설계 모델 (Hunter_6step | PBL_6step | Before_During_After)",
      "secondary_model": "string — 보조 모델 (Gagne 적용 범위)",
      "grr_focus": "string — GRR 중심 단계",
      "bloom_question_pattern": "string — 단계별 발문 수준 패턴 (예: L1-L2→L3-L4→L2-L3)"
    }
  },

  "metadata": {
    "created_at": "YYYY-MM-DD",
    "workflow": "lecture-script",
    "output_dir": "string — lectures/YYYY-MM-DD_{강의명}/02_script/"
  }
}
```

---

## C. 질문 흐름 설계

### 전체 흐름도

```
[시작]
  │
  ▼
[구성안 탐색]
  │  ├─ lectures/ 폴더 스캔 → 최신 강의 폴더 목록 제시
  │  ├─ AskUserQuestion: "어느 강의의 교안을 작성할까요?" (폴더 선택)
  │  └─ 선택된 폴더의 01_outline/ 에서 파일 존재 확인
  │       ├─ 06_write_lecture_outline.md 존재 → 파싱
  │       └─ 파일 없음 → 오류 보고 후 종료
  │
  ▼
[구성안 자동 파싱] (사용자 개입 없음)
  │  → inherited 필드 전체 추출
  │  → 총 교시 수, Phase 구조 산출
  │  → pedagogy에서 교수 모델·활동 전략 자동 추론 (§F)
  │
  ▼
[AskUserQuestion 1회차: SQ1 + SQ1a + SQ1b]
  │  SQ1: 교안 작성 범위 — 전체(추천) / 특정 차시 선택
  │  SQ1a: 교수 모델 — {추론결과}(추천) / 직접교수법 / PBL / 플립러닝
  │  SQ1b: 활동 전략 — 개인실습 / 그룹활동 / 토론·발문 / 프로젝트 (복수선택)
  │  ├─ SQ1 "특정 차시" → 후속 AskUserQuestion으로 Day 번호 입력
  │  └─ SQ1a "Other(혼합)" → 후속 안내 (Phase 5에서 Day별 자동 결정 가능)
  │
  ▼
[AskUserQuestion 2회차: SQ2 + SQ3 + SQ4]
  │  SQ2: 스크립트 상세도 — 완전스크립트 / 반구조화(추천) / 불릿노트
  │  SQ3: 형성평가 유형 — 섹션별체크(추천) / Exit Ticket / 실습통합 / 없음
  │  SQ4: 시간 비율 — 자동설정(추천) / 직접 입력
  │
  ▼
[추가 설정 확인]
  │  AskUserQuestion: "세부 설정을 조정하시겠습니까?"
  │  선택지: 기본값으로 진행(추천) / 세부 설정 조정
  │  │
  │  ├─ "기본값으로 진행" → SQ5~SQ7 기본값 적용 후 JSON 생성
  │  └─ "세부 설정 조정" → SQ5 → SQ6 → SQ7 순차 질문
  │
  │  ├─ [SQ5: Gagne 적용 수준]
  │  │      선택지: 전체 9사태 / 핵심 5사태(기본) / 라벨 없음
  │  │
  │  ├─ [SQ6: 발문 설계]
  │  │      선택지: 발문+예상답변 / 발문만(기본) / 제외
  │  │
  │  └─ [SQ7: 실습 가이드 상세도]
  │         선택지: 완전 가이드 / 단계목록+핵심지시(기본) / 제목+소요시간
  │
  ▼
[02_script/ 폴더 생성]
  │  → lectures/YYYY-MM-DD_{강의명}/02_script/ 생성
  │
  ▼
[01_input_data.json 생성]
  │  → 파싱된 inherited + script_settings + source + metadata 통합
  │  → instructional_model_map 자동 파생 (§E 매핑 테이블 기반)
  │
  ▼
[Phase 1 완료 검증] (§H 검증 기준)
  │
  ▼
[사용자 확인 출력]
  → 수집된 설정 요약 표시
  → Phase 2로 전달
```

### AskUserQuestion 호출 전략

| 단계 | 호출 | 질문 수 | 묶음 방식 | 이유 |
|------|------|---------|---------|------|
| 구성안 선택 | 1회 | 1 | 단독 | 선택 결과에 따라 파싱 내용이 결정됨 |
| 1회차 (SQ1+SQ1a+SQ1b) | 1회 | 3 | 묶음 | 범위·교수법·활동은 상호 독립. 동시 수집 가능 |
| SQ1 후속 (Day 선택) | 조건부 1회 | 1 | 단독 | "특정 차시" 선택 시에만 |
| 2회차 (SQ2+SQ3+SQ4) | 1회 | 3 | 묶음 | 상세도·평가·시간은 상호 독립. 동시 수집 가능 |
| 추가 설정 확인 | 1회 | 1 | 게이트 | "기본값 진행" 선택 시 SQ5~SQ7 전체 스킵 |
| SQ5~SQ7 | 조건부 1~3회 | 1~3 | 순차 | 게이트 통과 시에만 |
| **총 호출** | **최소 3회 ~ 최대 7회** | | | |

---

## D. 산출물 명명 규칙

### 산출물 위치

```
lectures/YYYY-MM-DD_{강의명}/02_script/
├── 01_input_data.json              # Phase 1: 이 설계안의 산출물
├── 02_explore_plan.md             # Phase 2: 탐색적 리서치 계획
├── 02_explore_research.md         # Phase 2: 리서치 결과
├── 03_brainstorm_result.md        # Phase 3: 브레인스토밍 결과
├── 04_deep_research.md            # Phase 4: 심화 리서치
├── 05_arch_lesson_plan.md         # Phase 5: 차시별 레슨 플랜 구조 설계
├── 06_write_script_draft.md       # Phase 6: 교안 초안 (중간)
├── 06_write_lecture_script.md     # Phase 6: 최종 교안 ★
└── 07_review_quality.md           # Phase 7: 품질 검토
```

**현행 SKILL.md의 파일명과 차이**:

| 현행 (SKILL.md) | 설계안 제안 | 변경 이유 |
|---------------|-----------|---------|
| `input_data.json` | `01_input_data.json` | 강의구성안과 동일한 번호 접두사 패턴 준수 |
| `research_exploration.md` | `02_explore_research.md` | Phase 번호 접두사 + 구성안과 일관된 명명 |
| `brainstorm_result.md` | `03_brainstorm_result.md` | 동일 패턴 |
| `research_deep.md` | `04_deep_research.md` | 동일 패턴 |
| `architecture.md` | `05_arch_lesson_plan.md` | 교안 Phase 5는 코스 아키텍처가 아닌 차시별 레슨 플랜 구조 설계(도입-전개-정리, Gagne 사태 배치)임을 명시 |
| `lecture_script.md` | `06_write_lecture_script.md` | CLAUDE.md의 최종 산출물 명 준수 |
| `quality_review.md` | `07_review_quality.md` | 동일 패턴 |

---

## E. 교수 모델 → 교수설계 모델 매핑 테이블

교수 모델 선택(SQ1a)이 확정되면, Phase 5(architecture-agent)와 Phase 6(writer-agent)에서 적용할 교수설계 모델 조합이 자동 결정된다.

### E-1. 교수 모델 × 교수설계 모델 × 교안 구조

| 교수 모델 (SQ1a) | 주 모델 | 보조 모델 | GRR 적용 | 도입 구조 | 전개 구조 | 정리 구조 |
|----------------|---------|----------|---------|----------|----------|----------|
| **직접교수법** | Hunter 6단계 | Gagne 9사태 (체크리스트) | I Do→We Do→You Do 전면 적용 | 복습 → 목표 제시 | 제시→시범→안내연습→독립연습 | 피드백 → 복습 → 차시 예고 |
| **PBL** | PBL 6단계 | Gagne 선택적 (사태1,2,6,7,9) | You Do Together 중심 | 문제 시나리오 제시 → 학습 목표 연결 | 문제 정의→탐구→해결책 개발 | 발표→성찰→동료 평가 |
| **플립러닝** | Before/During/After | Gagne 변형 (사태4-5 교실 외) | We Do + You Do Together | 사전학습 확인 → 핵심 보완 | 개념 명확화→그룹 활동→심화 적용 | 피드백→사후 과제 안내 |
| **혼합** | 차시별 위 3개 조합 | 차시별 개별 | 차시별 개별 | 차시별 개별 | 차시별 개별 | 차시별 개별 |

### E-2. 교수 모델 × Bloom's 발문 매핑

| 수업 단계 | 직접교수법 | PBL | 플립러닝 |
|----------|-----------|-----|---------|
| 도입 | L1~L2 (기억, 이해) | L4~L5 (분석, 평가 — 문제 분석) | L2~L3 (이해, 적용 — 사전학습 확인) |
| 전개 초반 | L2~L3 (이해, 적용) | L3~L4 (적용, 분석 — 탐구) | L3~L4 (적용, 분석) |
| 전개 후반 | L3~L4 (적용, 분석) | L5~L6 (평가, 창조 — 해결책) | L4~L5 (분석, 평가) |
| 정리 | L2~L3 (이해 확인, 적용 전이) | L5~L6 (평가, 창조 — 성찰) | L5 (평가 — 자기 점검) |

이 매핑은 Phase 6 writer-agent가 발문을 자동 생성할 때 참조한다.

### E-3. 형성평가 × 교수 모델 매핑

| 형성평가 유형 (SQ3) | 직접교수법 추천 도구 | PBL 추천 도구 | 플립러닝 추천 도구 |
|------------------|-------------------|-------------|-----------------|
| 섹션별 체크 | 퀴즈(O/X, 선택형), 화이트보드 응답 | 진행 발표, 동료 피드백 체크리스트 | 개념 적용 미니 과제 |
| 차시별 Exit Ticket | 3-2-1 반성, 1분 작문 | 성찰 일지, "오늘 배운 것/어려운 것" | "사전학습과 연결된 새 발견" |
| 실습 통합 평가 | 수행 체크리스트, 코드 리뷰 | 루브릭 기반 산출물 평가 | 실습 결과 동료 검토 |

---

## F. pedagogy → 교수 모델·활동 전략 자동 추론 규칙

구성안의 `pedagogy` 필드(자유 텍스트)에서 SQ1a(교수 모델)와 SQ1b(활동 전략)의 기본값을 추론한다.

### F-1. SQ1a 교수 모델 추론

`pedagogy` 텍스트를 소문자로 정규화한 뒤, 아래 규칙을 **순서대로** 적용한다. 첫 번째 매칭에서 정지.

| 우선순위 | 조건 (pedagogy 텍스트에 포함) | SQ1a 기본값 |
|---------|---------------------------|----------|
| 1 | `"pbl"` 또는 `"문제기반"` 또는 `"프로젝트 기반"` | `pbl` |
| 2 | `"플립"` 또는 `"flipped"` 또는 `"사전학습"` 또는 `"역전"` | `flipped` |
| 3 | `"직접교수"` 또는 `"direct instruction"` 또는 `"강의식"` | `direct_instruction` |
| 4 | 위 모두 해당 없음 | `direct_instruction` (기본값), confidence: `low` |

**예시**: `"PBL + AI-first, 실습 비율 50% 이상"` → 우선순위 1 매칭 → `pbl`

**low confidence 처리**: metadata에 `"pedagogy_parse_warning": "교수법 탐지 신뢰도 낮음. 자동 추론 결과를 사용자가 확인하세요."` 경고 기록.

### F-2. SQ1b 활동 전략 추론

동일한 `pedagogy` 텍스트에서 복수 매칭:

| 조건 (pedagogy 텍스트에 포함) | SQ1b에 추가 |
|---------------------------|-----------|
| `"실습"` 또는 `"hands-on"` 또는 `"practice"` | `individual_practice` |
| `"협업"` 또는 `"팀"` 또는 `"그룹"` 또는 `"페어"` | `group_activity` |
| `"토론"` 또는 `"발문"` 또는 `"discussion"` | `discussion` |
| `"프로젝트"` 또는 `"산출물"` 또는 `"project"` | `project` |

매칭 결과가 0개이면 기본값 `["individual_practice", "group_activity"]` 적용.

**예시**: `"PBL + AI-first, 실습 비율 50% 이상. 매일 산출물이 누적되는 프로젝트 기반 학습."`
→ `individual_practice` (실습), `project` (프로젝트, 산출물)

### F-3. 추론 결과 제시 방식

추론된 기본값을 AskUserQuestion 선택지에 "(추천)" 레이블로 표시한다.
사용자는 추론 결과를 확인 후 변경할 수 있다.

---

## G. Phase 간 데이터 흐름 및 전달 원칙

### G-1. Phase 5 (architecture-agent)에 전달할 원칙

1. `script_settings.teaching_model` + `instructional_model_map`을 참조하여 차시별 교안 골격 설계
2. `formative_assessment.type ≠ "none"`이면 → **각 SLO에 최소 1개의 형성평가 지점 배정** (SLO-평가 매핑 테이블 생성)
3. `time_ratio`를 차시별 분 단위 시간표로 변환
4. 혼합 교수법(`mixed`)이면 → `mixed_model_map`을 참조하여 Day별 구조 분기

### G-2. Phase 6 (writer-agent)에 전달할 원칙

1. `script_detail_level`에 따라 스크립트 밀도 조절
   - `full_script`: 발화 문장, 전환 멘트, 발문, 예상 답변까지 완전 기술
   - `semi_structured`: 핵심 설명은 단락형, 발문/전환은 완전 문장, 나머지 불릿
   - `bullet_notes`: 불릿 포인트 중심, 발문과 전환만 완전 문장
2. `instructional_model_map.bloom_question_pattern`에 따라 수업 단계별 발문 수준 자동 배정
3. `formative_assessment.type`에 맞는 평가 도구를 §E-3 매핑표에서 선택

### G-3. Phase 7 (review-agent) 검증 항목

1. SLO별 형성평가 커버리지 100% 검증 (formative_assessment ≠ "none" 시)
2. 목표-활동-평가 정렬 검증
3. 시간 비율 준수 검증 (도입:전개:정리)

---

## H. Phase 1 완료 검증 기준

Phase 1 완료 시 오케스트레이터가 확인할 항목:

| # | 확인 항목 | 검증 방법 |
|---|----------|----------|
| 1 | `02_script/01_input_data.json` 파일 존재 | Glob 확인 |
| 2 | `script_settings` 객체에 필수 필드 존재 (target_scope, teaching_model, activity_strategies, script_detail_level, formative_assessment, time_ratio) | Read → JSON 파싱 |
| 3 | `source.outline_file` 경로의 파일이 실제 존재 | Glob 확인 |
| 4 | `time_ratio.intro + main + wrap = 100` | 산술 검증 |
| 5 | `teaching_model.selected`가 유효 enum 값 | 값 검증 |
| 6 | `target_scope.type = "day"`이면 `session_ids` 비어있지 않음 | 조건부 검증 |

---

## I. 설계 결정 로그

| # | 결정 | 대안 | 근거 |
|---|------|------|------|
| 1 | 구성안 탐색을 "lectures/ 폴더 스캔 후 선택"으로 설계 | 사용자가 직접 경로 입력 | 경로 오타 방지. 최신 폴더를 자동 제시하는 것이 UX에 유리 |
| 2 | 필수 질문을 6개(2회 묶음 호출)로 확장 | 2개 필수 + 4개 선택 | 교수 모델·형성평가·시간비율이 Phase 5~6 구조에 직접 영향하므로 필수 확보. 2회 AskUserQuestion 묶음으로 UX 부담 최소화 |
| 3 | SQ5~SQ7를 선택 질문으로 게이팅 | 모두 필수 질문 | 대부분의 사용자는 기본값으로 충분. 강의구성안의 Q7~Q12 패턴과 동일한 구조 |
| 4 | 교수 모델을 자동 추론 후 사용자 확인 방식 | pedagogy에서만 자동 결정 (질문 없음) | 자유 텍스트 추론은 신뢰도가 낮을 수 있음. "(추천)"으로 제시 후 사용자가 변경 가능하게 하여 정확성과 편의성 양립 |
| 5 | `total_sessions`를 자동 산출 필드로 설계 | 사용자 입력 | schedule + sessions 파싱에서 계산 가능. writer-agent의 분할 작업 전략 트리거에 사용 |
| 6 | `sessions` 배열에 BOPPPS 구조 포함 | BOPPPS 제외, 차시 제목만 | Phase 6 작성 시 구성안의 BOPPPS 활동 설명을 스크립트로 확장해야 함 |
| 7 | Gagne 핵심 5사태를 기본값으로 설정 | 전체 9사태 기본 | 핵심 5사태(1·2·3·6·9)는 교육학적으로 가장 영향력 있는 사태. 4(자극제시)·5(학습안내)는 전개부 흐름에 내재화 |
| 8 | 파일명에 번호 접두사 적용 (01_, 02_ 등) | 접두사 없는 명명 | 강의구성안의 명명 패턴과 일관성 확보. 정렬 순서가 파이프라인 순서와 일치 |
| 9 | 교수 모델→교수설계 모델 매핑 테이블을 설계 문서에 포함 | Phase 5에서 자체 결정 | 매핑이 Phase 1에서 결정되어야 Phase 5·6이 일관된 구조로 작업 가능. `instructional_model_map`으로 JSON에도 자동 파생 |
| 10 | 형성평가-SLO 자동 배정 원칙 명시 | 형성평가 유형만 수집 | SLO별 평가 커버리지 100%를 보장하려면 Phase 5에 배정 원칙을 전달해야 함. Phase 7에서 검증 |

---

## J. 구현 시 주의사항

### J-1. 구성안 파싱 복잡도

`06_write_lecture_outline.md`는 마크다운 형식이므로 정규 표현식 또는 섹션별 파싱이 필요하다.
파싱 실패 시 `01_outline/01_input_data.json`을 fallback으로 사용한다.

**파싱 우선순위**:
1. `06_write_lecture_outline.md` (최신 확정본, 풍부한 구조)
2. `01_outline/01_input_data.json` (원본 입력, 일부 항목만 보완)

파싱 불가능한 항목(예: BOPPPS 구조가 구성안에 누락된 경우)은 `null`로 처리하고 Phase 6에서 처리한다.

**`theory_practice_ratio` fallback 기본값**:
구성안에 Phase별 이론:실습 비율이 명시되지 않은 경우, 다음 기본값을 적용한다:

| Phase | 기본 이론:실습 비율 | 근거 |
|-------|----------------|------|
| A (기초) | 70:30 | 개념 도입 단계, 이론 중심 |
| B (핵심) | 50:50 | 이론과 실습 균형 |
| C (심화) | 30:70 | 적용·실습 중심 |
| D (통합) | 10:90 | 프로젝트·종합 실습 중심 |

### J-2. AskUserQuestion 선택지 수 제약

AskUserQuestion 도구는 선택지를 2~4개로 제한한다. 설계된 질문은 모두 이 제약을 만족한다:
- SQ1: 2개 (전체/특정차시)
- SQ1a: 4개 (추론결과/직접교수법/PBL/플립러닝) — 혼합은 Other
- SQ1b: 4개 (개인실습/그룹/토론/프로젝트) — multiSelect
- SQ2: 3개 (완전/반구조화/불릿)
- SQ3: 4개 (섹션별/Exit Ticket/실습통합/없음)
- SQ4: 2개 (자동/직접입력)
- 추가 설정 확인: 2개 (기본값/세부 설정)
- SQ5~SQ7: 각 3개

### J-3. 강의 폴더 선택 시 표시 개수

#### 엣지 케이스 처리

| 상황 | 처리 방법 |
|------|---------|
| `lectures/` 폴더가 없음 | 오류: "강의 폴더를 찾을 수 없습니다. `/lecture-outline`을 먼저 실행하세요." 출력 후 종료 |
| 강의 폴더 0개 (빈 디렉토리) | 동일 오류 메시지 출력 후 종료 |
| 강의 폴더 1개 | 해당 폴더 + "기타 직접 입력" = 2개 선택지 (AskUserQuestion 최소 요건 충족) |
| 강의 폴더 2개 | 2개 폴더 + "기타 직접 입력" = 3개 선택지 |
| 강의 폴더 3개+ | 최신 3개 + "기타 직접 입력" = 4개 선택지 |
| 선택한 폴더에 `01_outline/06_write_lecture_outline.md` 없음 | 오류: "선택한 강의의 구성안 파일이 없습니다. `/lecture-outline`을 먼저 완료하세요." 출력 후 종료 |
| "기타 직접 입력" 선택 | `lectures/` 하위 폴더명(예: `2026-03-10_딥러닝-기초`)만 입력받음. 절대경로 불필요 |

### J-4. 분할 작업 트리거 연계

`total_sessions`가 계산되면, 이 값이 Phase 6 writer-agent의 분할 작업 전략에 자동 전달된다:
- ≤20교시: 단일 작업
- 21~40교시: 2분할
- 41+교시: 3분할

이 로직은 Phase 6에서 처리되므로 Phase 1에서는 `total_sessions` 산출만 수행한다.

---

## K. 다음 단계 (Phase 2) 연계

Phase 1 완료 후 `01_input_data.json`이 생성되면, Phase 2(탐색적 리서치)의 입력이 된다.

교안 워크플로우의 Phase 2 리서치 방향은 구성안과 다르다:
- **교수법 사례**: 선택된 교수 모델(`teaching_model`)의 실제 교안 예시
- **Gagne/Hunter 적용 패턴**: 유사 주제에서의 수업사태 적용 사례
- **발문 패턴**: Bloom's 수준별 발문 예시 (SQ6 설정 반영)
- **실습 활동 아이디어**: `practice_guide_detail` 수준에 맞는 실습 구조
- **형성평가 도구**: `formative_assessment` 유형에 맞는 평가 도구 사례 (§E-3 매핑 기반)

Phase 2 리서치 에이전트는 `script_settings` 필드를 읽어 리서치 방향을 조정한다.
