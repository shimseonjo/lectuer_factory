# 슬라이드 기획 워크플로우 Phase 1 입력 수집 설계안

## 설계 개요

- **작성일**: 2026-03-12
- **버전**: v4
- **목적**: `/slide-planning` 워크플로우의 Phase 1(입력 수집) 상세 설계
- **담당 에이전트**: input-agent
- **참조 패턴**: 강의교안 Phase 1 UX 패턴 준수

**v3 주요 변경사항**:
- PQ1·PQ4·PQ5·PQ6·PQ7을 이전 단계 산출물에서 자동 추론으로 전환
- 사용자 질문을 PQ2(도구) + PQ3(밀도)로 축소
- AskUserQuestion 호출: 최소 2회 ~ 최대 3회 (기존 최대 5회에서 축소)
- §A-2 전면 재구성: 자동 추론 항목(5개) + 사용자 질문(2개)
- §B slide_settings에 `inferred_from` 필드 추가
- §C 질문 흐름 전면 재작성
- §J 설계 결정 #13 추가 (자동 추론 최적화 근거)

**v4 주요 변경사항**:
- 모듈 교안(`06_modules/`) → 세션+아키텍처 하이브리드 입력으로 전환
- PQ3 밀도 질문 → 자동 적응형으로 전환 (슬라이드 유형별 자동 결정)
- 적응형 슬라이드 수 반영 (envelope 8-20장, 유형별: 개념 13-20, 코드 10-15, 실습 8-13, 프로젝트 8-13, 혼합 10-15. Phase 3에서 산출)
- 2계층 → 4계층 로드 전략으로 확장

---

## A. 입력 수집 항목 설계

### A-1. 교안 자동 로드 항목 — 4계층 분리 로드 전략

슬라이드 기획은 강의교안의 후속 워크플로우이다. 교안 워크플로우가 생성하는 세션 파일과 아키텍처 파일을 하이브리드 입력으로 사용하며, 어떤 계층에서 무엇을 읽을지가 핵심 설계 결정이다.

**결정: 세션 파일(`06_sessions/`) + 아키텍처 파일(`05_arch_architecture.md`) 하이브리드 입력**

| 근거 # | 내용 |
|--------|------|
| 1 | **세션 파일 효율**: 세션 파일은 ~200줄 수준으로 차시별 콘텐츠를 직접 제공 |
| 2 | **아키텍처 파일 보완**: `05_arch_architecture.md`가 매크로 구조, 시간표, 연결 맵, 정렬 매트릭스 등 구조 맥락을 보완 |
| 3 | **적절한 청크 크기**: 세션 단위로 처리 가능 (한 세션 = 한 컨텍스트) |
| 4 | **전환 맥락 유지**: 아키텍처가 세션 간 연결 구조를 제공 → 섹션 전환 슬라이드에 반영 |

#### 4계층 분리 로드 상세

**L1: 아키텍처 파일** — `01_outline/05_arch_architecture.md` 전체

| 파싱 소스 | 파싱 항목 | JSON 키 |
|----------|----------|---------|
| 아키텍처 §매크로 구조 | 일별×교시별 구조, Phase 구분 | `inherited.timetable` |
| 아키텍처 §시간표 | 일별×교시별 그리드 | `inherited.timetable` |
| 아키텍처 §연결 맵 | 세션 간 연결 관계 | `inherited.connection_map` |
| 아키텍처 §정렬 매트릭스 | SLO-활동-평가 정렬 | `inherited.alignment_matrix` |

**L2: 세션 파일** — `02_script/06_sessions/06_session_{NNN}.md` × N개

| 파싱 소스 | 파싱 항목 | JSON 키 |
|----------|----------|---------|
| 세션 헤더 | 차시 번호, 제목, Phase, SLO | `inherited.sessions[N].header` |
| 세션 도입/전개/정리 | 차시별 구조, 발문, 활동, 형성평가 | `inherited.sessions[N].body` |
| 발표자 노트 | 타이밍, 오개념, 대안 활동, 자료 체크 | `inherited.sessions[N].speaker_notes` |

**L3: 통합본 상단** — `02_script/06_write_lecture_script.md` 처음 ~100줄

| 파싱 소스 | 파싱 항목 | JSON 키 |
|----------|----------|---------|
| 교안 §메타데이터 | 강의 주제, 대상 학습자, 강의 형태, 시간 구성, 교수 모델, 톤/스타일 | `inherited.topic`, `inherited.target_learner`, `inherited.format`, `inherited.schedule`, `inherited.teaching_model`, `inherited.tone` |
| 교안 §2 | SLO 목록 + Bloom's 수준 | `inherited.learning_goals` |

→ 통합본의 **처음 ~100줄**(메타데이터·SLO)만 파싱. 차시별 스크립트 본문은 읽지 않는다.

**L4: 보조 JSON** — input_data.json에서 보완

| 파싱 소스 | 파싱 항목 | JSON 키 |
|----------|----------|---------|
| `02_script/01_input_data.json` | teaching_model, script_settings | `inherited.pedagogy` 등 |
| `01_outline/01_input_data.json` | tone_examples, lab_environment | `inherited.tone_examples`, `inherited.lab_environment` |
| `01_outline/01_input_data.json` | keywords | `inherited.keywords` |

**파싱 우선순위** (동일 항목이 여러 소스에 있을 때):
1. `02_script/06_sessions/06_session_{NNN}.md` (세션별 상세 콘텐츠)
2. `01_outline/05_arch_architecture.md` (매크로 구조·시간표)
3. `02_script/06_write_lecture_script.md` 상단 섹션 (코스 레벨 메타데이터)
4. `01/02_input_data.json` (보조 설정)

**콘텐츠 유형 자동 탐지**: 세션 파일 파싱 시 차시 내용에서 다음 키워드를 탐지하여 `derived` 필드에 기록:
- `has_code_content`: 키워드 — "코드", "코딩", "라이브 코딩", "코드 워크스루", "코드 리뷰", "프로그래밍", "구현", "디버깅", "```" (코드 블록). 하나 이상 매칭 시 `true`
- `has_activity_content`: 키워드 — "실습", "그룹 활동", "팀 활동", "프로젝트", "워크숍", "핸즈온", "hands-on", "협업 활동". 하나 이상 매칭 시 `true`
- `has_quiz_content`: 키워드 — "형성평가", "퀴즈", "확인 문제", "자가진단", "평가 문항". 하나 이상 매칭 시 `true`

---

### A-2. 슬라이드 기획 설정 항목

교안에 없고 슬라이드 기획에 실질적으로 영향을 미치는 항목을 수집한다. 이전 단계 산출물에서 자동 추론 가능한 항목은 사용자에게 재질문하지 않는다.

#### 자동 추론 항목 (6개) — 이전 단계 산출물에서 결정적 파생

| # | 카테고리 | 추론 소스 | 추론 로직 | 결과 |
|---|---------|----------|----------|------|
| PQ1 | 슬라이드 작성 범위 | `02_script/01_input_data.json` → `script_settings.target_scope` | 교안 작성 범위와 동일하게 자동 매칭 | `target_scope` 동기화 |
| PQ3 | 슬라이드 밀도 | 세션 파일 콘텐츠 유형 분석 | 세션 내 콘텐츠 유형 비율 분석 → 슬라이드 유형별 적응형 밀도 자동 결정 | `density_profile.mode = "adaptive"` |
| PQ4 | Assertion-Evidence 적용 | 슬라이드 유형별 적응형 밀도 | 유형별 밀도 범위에 따른 A-E 적용 수준 자동 결정 | `assertion_evidence.level` |
| PQ5 | 디자인 톤 | L3: 교안 메타데이터 `톤/스타일` → `inherited.tone` (primary), L4: `01_outline/01_input_data.json` → `tone_examples` (보조 키워드) | 키워드 매칭 규칙 (아래 상세) | `design_tone.style` |
| PQ6 | 발표자 노트 | 교안 워크플로우 특성 | 교안은 항상 발표자 노트를 생성 → 항상 포함 | `speaker_notes.include = true` |
| PQ7 | 코드 테마 | `derived.has_code_content` | 코드 콘텐츠 존재 시 dark 기본값 자동 설정 | `code_theme.theme = "dark"` |

**PQ1 자동 추론 상세**:
- 교안의 `script_settings.target_scope`가 `"전체"`이면 → `target_scope.type = "전체"`, `session_ids = 전체 교시 배열`
- 교안이 특정 차시만 작성했으면 → 해당 차시만 슬라이드 기획 대상으로 동기화
- 근거: 교안이 작성되지 않은 차시의 슬라이드를 기획할 수 없음

**PQ3 적응형 밀도 자동 결정 로직**:

세션 파일에서 콘텐츠 유형 비율을 분석하여 슬라이드 유형별 적응형 밀도를 자동 결정한다. §F 적응형 밀도 기준표에 따라 각 유형에 독립적 밀도 범위를 적용한다.

- `code` 유형 비율이 높으면 → code 슬라이드 밀도 25-35줄 범위 내 자동 조정
- `activity` 비율이 높으면 → activity 슬라이드 밀도 20-30줄 범위 내 자동 조정
- `concept` 비율이 높으면 → concept 슬라이드 밀도 20-30줄 범위 내 자동 조정
- 근거: 사용자에게 밀도 숫자를 직접 묻는 것보다 콘텐츠 유형 기반 자동 결정이 더 일관성 있는 결과 제공

**PQ4 밀도→A-E 결정적 매핑**:

| 슬라이드 유형 | 적응형 밀도 | PQ4 A-E 수준 | 근거 |
|-------------|-----------|-------------|------|
| `concept`, `data_insight`, `image` | 15-25줄 | `partial` (부분 적용) | 시각 증거 공간 확보 필요 |
| `code`, `comparison` | 25-35줄 | `변형` | 코드/표 자체가 증거 역할 |
| `summary`, `activity`, `timeline` | 15-25줄 | `변형` | 구조적 시각화 적용 |
| `title`, `agenda`, `section_transition`, `quote`, `quiz` | 3-12줄 | `—` | A-E 불필요 유형 |

**PQ5 톤 키워드 매칭 규칙**:

| `tone` 텍스트 키워드 | 추론 결과 | 매핑 |
|--------------------|----------|------|
| "비유", "메타포", "친근", "일상" | `educational` | 교육적/친근 |
| "전문", "기업", "공식", "절제" | `professional` | 전문적/기업 |
| "간결", "미니멀", "깔끔", "단순" | `minimal` | 미니멀 |
| 매칭 없음 (fallback) | `format.type` 기반 | 강의/워크숍 → `educational`, 세미나 → `professional` |

**PQ6·PQ7 근거**:
- PQ6: `/lecture-script` 워크플로우는 모든 차시에 발표자 노트를 생성하므로 항상 `include = true`
- PQ7: 코드 콘텐츠가 있을 때 `dark`가 가독성·접근성 모두 우수. 없으면 `code_theme.applicable = false`

#### 사용자 질문 항목 (1개) — AskUserQuestion 1회 호출

이전 단계에서 추론할 수 없고, 사용자의 의도가 반영되어야 하는 항목만 질문한다.

| # | 카테고리 | 질문 | 이유 |
|---|---------|------|------|
| PQ2 | 슬라이드 도구 | **슬라이드 생성에 사용할 도구를 선택하세요.** | 도구별 레이아웃 제약과 기능 차이가 기획에 직접 영향. 사용자 환경·선호에 의존 |

**PQ2 선택지** (AskUserQuestion):
```
1. Marp (추천) — 마크다운 기반, AI 생성 용이, Git 친화적
2. Slidev — Vue 기반, 코드 데모·줄별 하이라이트, 개발 교육 특화
3. Gamma — AI 기반, 시각적 완성도, 비개발자 접근성
4. reveal.js — 최대 커스터마이징, 웹 배포, 고급 인터랙션
```

**도구 자동 추천 로직**: `has_code_content`와 `lab_environment`를 기반으로 추천 순서 조정
- 코드 중심 교육 → Slidev 추천 (줄별 하이라이트, 라이브 코딩 지원)
- 비개발자 대상 → Gamma 추천 (시각적, 접근성)
- 범용 교육 → Marp 추천 (학습 곡선 최소)

---

### A-3. 항목 설계 결정 근거: 제외한 항목들

| 항목 | 제외 이유 |
|------|----------|
| 학습 목표 | 교안에서 상속. 슬라이드가 커버해야 할 SLO는 이미 확정 |
| 차시 구조 | 교안 §3에서 완전한 도입-전개-정리 구조를 상속 |
| 슬라이드 유형 선택 | Phase 3(architecture-agent)에서 콘텐츠 기반 자동 결정 |
| 슬라이드 수 | Phase 3에서 시간·밀도·콘텐츠 기반 자동 산출 |
| 에니메이션/트랜지션 | Phase 4(writer-agent)에서 도구별 기본 설정 적용 |
| 이미지/아이콘 소스 | Phase 2(brainstorm-agent)에서 시각화 아이디어 도출 |

---

## B. input_data.json 스키마 설계

```json
{
  "$schema": "03_slide_plan/01_input_data.json 스키마 — 슬라이드 기획 워크플로우 input-agent 생성",

  "source": {
    "script_dir": "string — 교안 폴더 경로 (예: lectures/2026-03-10_딥러닝-기초/02_script)",
    "script_file": "string — 교안 최종 통합본 경로 (06_write_lecture_script.md) — 메타데이터·SLO 상단 추출용",
    "arch_file": "string — 아키텍처 파일 경로 (01_outline/05_arch_architecture.md) — 매크로 구조·시간표·연결 맵",
    "session_dir": "string — 세션 파일 폴더 경로 (02_script/06_sessions/)",
    "session_files": ["string — 세션 파일 경로 목록 (06_session_001.md, ...)"],
    "script_input_json": "string — 교안 01_input_data.json 경로",
    "outline_input_json": "string — 구성안 01_input_data.json 경로",
    "load_strategy": "4-tier — L1: 아키텍처(구조) + L2: 세션(콘텐츠) + L3: 통합본 상단(메타데이터) + L4: JSON 보조"
  },

  "inherited": {
    "topic": "string — 교안에서 로드",
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
      "total_sessions": "number — 총 교시 수"
    },
    "teaching_model": {
      "selected": "direct_instruction | pbl | flipped | mixed",
      "label": "string"
    },
    "pedagogy": "string",
    "tone": "string",
    "tone_examples": ["string — 메타포 목록"],
    "learning_goals": [
      {
        "id": "number",
        "text": "string",
        "blooms_level": "기억 | 이해 | 적용 | 분석 | 평가 | 창조",
        "priority": "핵심 | 중요 | 보충"
      }
    ],
    "timetable": [
      {
        "day": "number",
        "session": "number",
        "time": "string — 09:00-09:50",
        "title": "string",
        "teaching_model": "string",
        "key_activity": "string",
        "phase": "A | B | C | D"
      }
    ],
    "connection_map": [
      {
        "from_session": "number",
        "to_session": "number",
        "connection_type": "string — 연결 유형 (선수/심화/병렬)"
      }
    ],
    "sessions": [
      {
        "session_id": "number — 전체 교시 번호",
        "title": "string",
        "learning_goals": ["number — SLO ID"],
        "phase": "A | B | C | D",
        "source_file": "string — 06_session_{NNN}.md 파일 경로",
        "intro_summary": "string — 도입 핵심 내용 요약",
        "main_summary": "string — 전개 핵심 내용 요약",
        "wrap_summary": "string — 정리 핵심 내용 요약",
        "activities": [
          {
            "name": "string",
            "type": "개인 | 짝 | 소그룹 | 전체",
            "duration_min": "number",
            "slo_ids": ["number"],
            "content_type": "concept | code | practice | discussion | project"
          }
        ],
        "questions": [
          {
            "text": "string",
            "blooms_level": "string",
            "stage": "도입 | 전개 | 정리"
          }
        ],
        "assessments": [
          {
            "type": "string",
            "slo_ids": ["number"],
            "stage": "string"
          }
        ],
        "speaker_notes": {
          "timing": "string",
          "misconceptions": "string",
          "alternatives": "string",
          "prep_checklist": "string"
        }
      }
    ],
    "keywords": ["string"],
    "lab_environment": "string | null"
  },

  "slide_settings": {
    "target_scope": {
      "type": "전체 | day | session",
      "value": "all | number",
      "session_ids": ["number — 기획 대상 교시 번호 목록"],
      "inferred_from": "script_settings.target_scope — 교안 작성 범위와 동기화"
    },
    "tool": {
      "selected": "marp | slidev | gamma | revealjs",
      "label": "Marp | Slidev | Gamma | reveal.js",
      "auto_recommended": "string — 자동 추천 근거",
      "inferred_from": null,
      "_note": "사용자 질문 PQ2 — 자동 추론 불가, 사용자 환경·선호에 의존"
    },
    "density_profile": {
      "mode": "adaptive",
      "type_density_map": {
        "title": "3-5줄",
        "agenda": "8-12줄",
        "section_transition": "3-5줄",
        "concept": "20-30줄",
        "code": "25-35줄",
        "comparison": "20-30줄",
        "data_insight": "15-25줄",
        "image": "10-20줄",
        "timeline": "15-25줄",
        "quote": "5-10줄",
        "summary": "15-25줄",
        "activity": "20-30줄",
        "quiz": "15-25줄"
      },
      "inferred_from": "세션 콘텐츠 유형 분석 → 슬라이드 유형별 적응형 밀도 자동 결정",
      "_note": "PQ3 자동 추론 — 세션 파일 콘텐츠 유형 비율 분석 기반"
    },
    "assertion_evidence": {
      "level": "full | partial | bullet",
      "label": "전면 적용 | 부분 적용 | 전통 불릿포인트",
      "apply_to_types": ["concept", "comparison", "data_insight", "summary"],
      "inferred_from": "density_profile.type_density_map — 슬라이드 유형별 적응형 밀도→A-E 결정적 매핑 (§A-2 표 참조)"
    },
    "design_tone": {
      "style": "educational | professional | minimal",
      "label": "교육적/친근 | 전문적/기업 | 미니멀",
      "inferred_from": "inherited.tone — 키워드 매칭 규칙 (§A-2 표 참조)"
    },
    "speaker_notes": {
      "include": true,
      "inferred_from": "교안 워크플로우 특성 — 항상 발표자 노트 생성"
    },
    "code_theme": {
      "theme": "dark | light | monokai",
      "default": "dark",
      "applicable": "boolean — has_code_content 기반",
      "inferred_from": "derived.has_code_content — 코드 존재 시 dark 자동 설정"
    }
  },

  "derived": {
    "total_sessions": "number — 전체 교시 수",
    "target_sessions_count": "number — 기획 대상 교시 수",
    "has_code_content": "boolean — 코드 활동 존재 여부",
    "has_activity_content": "boolean — 실습/그룹 활동 존재 여부",
    "has_quiz_content": "boolean — 형성평가 존재 여부"
  },

  "metadata": {
    "created_at": "YYYY-MM-DD",
    "workflow": "slide-planning",
    "output_dir": "string — lectures/YYYY-MM-DD_{강의명}/03_slide_plan/"
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
[Step 0: 강의 폴더 탐색]
  │  ├─ lectures/ 폴더 스캔 → 최신 강의 폴더 목록 제시
  │  ├─ AskUserQuestion: "어느 강의의 슬라이드를 기획할까요?" (폴더 선택)
  │  └─ 선택된 폴더의 02_script/ 에서 파일 존재 확인
  │       ├─ 06_sessions/ 폴더 존재 → Step 1로
  │       └─ 세션 폴더 없음 → 오류 보고 후 종료
  │
  ▼
[Step 1: 4계층 교안 자동 파싱 + 자동 추론] (사용자 개입 없음)
  │  ── 교안 파싱 ──
  │  L1: 01_outline/05_arch_architecture.md → 매크로 구조, 시간표, 연결 맵, 정렬 매트릭스
  │  L2: 02_script/06_sessions/06_session_{NNN}.md × N개 → 차시별 상세 콘텐츠
  │  L3: 02_script/06_write_lecture_script.md 상단 ~100줄 → 메타데이터, SLO
  │  L4: 01/02_input_data.json → 보조 설정
  │  → sessions 배열 구성, 총 교시 수, 콘텐츠 유형(코드/활동/퀴즈) 자동 탐지
  │  → 도구 자동 추천 로직 실행
  │
  │  ── 자동 추론 (6개 항목) ──
  │  PQ1: script_settings.target_scope → target_scope 동기화
  │  PQ3: 세션 콘텐츠 유형 분석 → 슬라이드 유형별 적응형 밀도 자동 결정
  │  PQ4: 적응형 밀도 → A-E 수준 자동 결정
  │  PQ5: tone 키워드 매칭 → design_tone.style
  │  PQ6: 교안 항상 발표자 노트 생성 → speaker_notes.include = true
  │  PQ7: has_code_content → code_theme.theme = "dark" (또는 applicable = false)
  │
  ▼
[Step 2: AskUserQuestion — PQ2]
  │  PQ2: 슬라이드 도구 — {자동추천}(추천) / 나머지 3개
  │
  ▼
[Step 3: 자동 추론 결과 확인 게이팅]
  │  AskUserQuestion: "자동 설정을 확인하세요. 조정이 필요하면 선택하세요."
  │  (적응형 밀도 결정 근거 요약 표시 포함)
  │  선택지: 자동 설정 그대로 진행(추천) / 세부 설정 조정
  │  │
  │  ├─ "자동 설정 그대로" → PQ1·PQ3·PQ4·PQ5·PQ6·PQ7 자동값 확정 후 JSON 생성
  │  └─ "세부 설정 조정" → AskUserQuestion 1회 추가
  │     └─ [PQ4 + PQ5 + PQ7] — 1회 묶음 (최대 3개 질문)
  │        PQ4: Assertion-Evidence — {자동값}(추천) / 나머지 2개
  │        PQ5: 디자인 톤 — {자동값}(추천) / 나머지 2개
  │        PQ7: 코드 테마 — dark(추천) / light / monokai (has_code_content 시에만)
  │
  ▼
[Step 4: 03_slide_plan/ 폴더 생성]
  │  → lectures/YYYY-MM-DD_{강의명}/03_slide_plan/ 생성
  │
  ▼
[Step 5: 01_input_data.json 생성]
  │  → inherited + slide_settings + derived + source + metadata 통합
  │
  ▼
[Step 6: Phase 1 완료 검증] (§I 검증 기준)
  │
  ▼
[사용자 확인 출력]
  → 수집된 설정 요약 표시 (자동 추론값 + 적응형 밀도 결정 근거 포함)
  → Phase 2로 전달
```

### AskUserQuestion 호출 전략

| 단계 | 호출 | 질문 수 | 묶음 방식 | 이유 |
|------|------|---------|---------|------|
| 강의 선택 | 1회 | 1 | 단독 | 선택 결과에 따라 파싱 대상 결정 |
| PQ2 | 1회 | 1 | 단독 | 도구는 사용자 환경·선호에 의존 |
| 자동 추론 확인 | 1회 | 1 | 게이트 | "그대로 진행" 시 PQ4·PQ5·PQ7 조정 스킵 |
| PQ4+PQ5+PQ7 | 조건부 1회 | 2~3 | 묶음 | 게이트 통과 시에만. PQ7은 has_code_content 조건부 |
| **총 호출** | **최소 2회 ~ 최대 3회** | | | PQ3 자동화로 질문 1개 감소 |

### v3 대비 변경 요약

| 항목 | v3 | v4 (변경) |
|------|----------|----------|
| 입력 소스 | 모듈 교안(`06_modules/`) 주 입력 | **세션+아키텍처 하이브리드** (`06_sessions/` + `05_arch_architecture.md`) |
| 로드 전략 | 2계층 분리 로드 | **4계층 분리 로드** (L1-L4) |
| PQ3 (밀도) | 사용자 질문 (3개 프리셋) | **자동 적응형** (세션 콘텐츠 유형 분석) |
| PQ4 (A-E) | PQ3 밀도 프리셋 파생 | **슬라이드 유형별 적응형 밀도 파생** |
| 자동 추론 항목 수 | 5개 | **6개** (PQ3 추가) |
| 사용자 질문 항목 수 | 2개 (PQ2+PQ3) | **1개 (PQ2만)** |
| PQ1 (범위) | 자동 추론 | 자동 추론 (변경 없음) |
| PQ5 (톤) | 자동 추론 | 자동 추론 (변경 없음) |
| PQ6 (노트) | 자동 설정 | 자동 설정 (변경 없음) |
| PQ7 (코드) | 자동 설정 | 자동 설정 (변경 없음) |
| PQ2 (도구) | 사용자 질문 | 사용자 질문 (변경 없음) |
| 최대 호출 횟수 | 3회 | **3회** (변경 없음, PQ2·게이트·조정) |

---

## D. 산출물 명명 규칙

### 글로벌 Phase 번호 체계 적용

슬라이드 기획은 5개 Phase이지만, 파일명은 **글로벌 Phase 유형 번호**를 사용한다:

| 워크플로우 Phase | 글로벌 번호 | 에이전트 | 파일명 |
|----------------|-----------|---------|--------|
| Phase 1: 입력 수집 | 01 | input-agent | `01_input_data.json` |
| Phase 2: 브레인스토밍 | 03 | brainstorm-agent | `03_brainstorm_result.md` |
| Phase 3: 구조 설계 | 05 | architecture-agent | `05_arch_slide_structure.md` |
| Phase 4: 기획안 작성 | 06 | writer-agent | `06_write_slide_plan.md` ★ |
| Phase 5: 품질 검토 | 07 | review-agent | `07_review_quality.md` |

### 산출물 위치

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

---

## E. 슬라이드 유형 체계 (13가지)

기존 12가지에서 **퀴즈** 유형을 추가하여 13가지로 확장한다. 교안의 형성평가가 슬라이드에 직접 매핑되기 때문이다.

| # | 유형 ID | 유형명 | 목적 | A-E 적용 |
|---|---------|--------|------|---------|
| 1 | `title` | 제목 | 강의 제목, 발표자, 날짜 | 해당 없음 |
| 2 | `agenda` | 아젠다 | 전체 구조·순서 안내 | 해당 없음 |
| 3 | `section_transition` | 섹션 전환 | 주제 전환 신호, 진행률 표시 | 해당 없음 |
| 4 | `concept` | 개념 설명 | 핵심 개념·이론 전달 | **전면 적용** |
| 5 | `code` | 코드 | 코드 워크스루, 라이브 코딩 | **변형 적용** (assertion + code evidence) |
| 6 | `comparison` | 비교 | 옵션·개념 대조 | **변형 적용** (assertion + 비교표/다이어그램) |
| 7 | `data_insight` | 데이터+인사이트 | 데이터 시각화, 통계, 차트 | **전면 적용** |
| 8 | `image` | 이미지/다이어그램 | 시각적 설명, 아키텍처 도식 | **전면 적용** (assertion + 이미지 evidence) |
| 9 | `timeline` | 타임라인 | 순서, 프로세스, 로드맵 | 변형 적용 |
| 10 | `quote` | 인용 | 명언, 핵심 메시지 강조 | 해당 없음 |
| 11 | `summary` | 핵심 요약 | 섹션/차시 핵심 정리 | **변형 적용** |
| 12 | `activity` | 실습/활동 | 학습자 참여, 실습 절차 | 변형 적용 (assertion + 절차) |
| 13 | `quiz` | 퀴즈/형성평가 | 이해도 확인, 점검 | 해당 없음 |

---

## F. 슬라이드 밀도·시간 기준 (리서치 기반)

### F-1. 리서치 근거 요약

| 출처 | 핵심 발견 | 시사점 |
|------|---------|--------|
| Assertion-Evidence (Michael Alley, Penn State) | 주장 문장(assertion) + 시각적 증거(visual evidence)가 불릿포인트보다 이해도·기억 향상 | 개념 슬라이드에 A-E 구조 적용 |
| PMC Naegle 2021 (Ten Simple Rules) | 슬라이드당 1 아이디어, 교육용은 0.5 slides/min (2분/슬라이드) | 교육용 기본 속도 2-3분/슬라이드 |
| Brown University Sheridan Center | 슬라이드는 강의 내용의 반복이 아닌 보충. 최소 18pt 폰트, 산세리프 | 밀도 제한 필요. 접근성 기준 반영 |
| Mayer 멀티미디어 학습 원칙 | 분할 효과(Segmenting), 이중 채널(Dual-Channel), 중복 원칙(Redundancy) | 텍스트와 시각 자료 균형. 말과 동일 텍스트 동시 제시 회피 |
| 인지 부하 이론 (Sweller) | 작업 기억 용량 7±2 항목. 외재적 인지 부하 최소화 | 슬라이드당 불릿 5개 이하, 핵심 개념 1개 |
| Guy Kawasaki 10-20-30 Rule | 10 slides, 20 minutes, 30pt font minimum | 프레젠테이션 참고 (교육과 맥락 다름) |
| Teaching-specific research | 교육 슬라이드: 15-30 slides/hour, 복잡한 내용 3-5분/슬라이드 | 유형별 차등 적용 |

### F-2. 적응형 밀도 기준표 (유형별)

세션 콘텐츠 유형을 분석하여 슬라이드 유형별 독립적 밀도 범위를 자동 적용한다. 단일 프리셋이 아닌 유형별 적응형 기준이다.

| 유형 | 밀도(줄) | A-E | 시간(분) | 설명 |
|------|---------|-----|---------|------|
| title | 3-5 | — | 1-2 | 강의명, 발표자, 날짜 |
| agenda | 8-12 | — | 2-3 | 전체 목차 |
| section_transition | 3-5 | — | 0.5-1 | 섹션 제목 + 한 줄 요약 |
| concept | 20-30 | partial | 3-5 | assertion 헤드라인 + 시각 증거 + 핵심 설명 |
| code | 25-35 | 변형 | 4-5 | assertion 헤드라인 + 코드 블록 + 주석 |
| comparison | 20-30 | partial | 3-5 | assertion 헤드라인 + 비교표 |
| data_insight | 15-25 | partial | 3-5 | assertion 헤드라인 + 차트/그래프 |
| image | 10-20 | partial | 3-4 | assertion 헤드라인 + 이미지/다이어그램 |
| timeline | 15-25 | 변형 | 3-4 | assertion 헤드라인 + 순서도 |
| quote | 5-10 | — | 2-3 | 인용문 + 출처 |
| summary | 15-25 | 변형 | 3-5 | 핵심 포인트 3-5개 |
| activity | 20-30 | 변형 | 표시만 | assertion 목표 + 절차 (활동 시간은 별도) |
| quiz | 15-25 | — | 3-5 | 문항 + 선택지 |

- **슬라이드 수 (Phase 3에서 산출)**: 전체 envelope 8-20장. 세션 유형별 구체 범위는 Phase 3(architecture-agent)에서 적용:
  - 개념 중심: 13-20장 | 코드 중심: 10-15장 | 실습: 8-13장 | 프로젝트: 8-13장 | 혼합: 10-15장
  - 50분 세션 기본: ~10-13장 (평균 3-5분/슬라이드)
- **특징**: 유형별 독립 밀도 적용. 동일 세션 내에서도 유형에 따라 밀도가 달라짐

### F-3. 설계 원칙 검토 결과

| 원칙 | 검토 결과 | 적용 방식 |
|------|---------|---------|
| **슬라이드당 1 아이디어** | ✅ 모든 유형에 적용. 인지 부하 이론과 일치 | 전체 기본 원칙 |
| **유형별 차등 밀도** | ✅ 개념(20-30줄), 코드(25-35줄), 전환(3-5줄)으로 차등 | 적응형 기준표 적용 |
| **2-5분/슬라이드** | ✅ 교육 맥락에서 적절. 다만 전환/제목은 1-2분 | 활동 시간 별도. 표시 시간만 산정 |
| **Assertion-Evidence** | ✅ 개념·비교·데이터 슬라이드에 효과적. 코드/실습은 변형 A-E가 더 적합 | 유형별 A-E 수준 자동 적용 |

**종합 권장**: 세션 콘텐츠 유형 분석 기반 적응형 밀도 자동 결정. 사용자는 도구 선택(PQ2)만 입력하면 됨.

---

## G. Assertion-Evidence 적용 가이드

### G-1. Assertion-Evidence 핵심 원칙

Michael Alley(Penn State) 연구 기반:

1. **Assertion Headline**: 슬라이드 상단에 완전한 주장 문장 배치 (구 또는 키워드가 아닌 **문장**)
   - ❌ "머신러닝 개요" (구)
   - ✅ "머신러닝은 데이터에서 패턴을 학습하여 예측하는 알고리즘이다" (주장)

2. **Visual Evidence**: 불릿포인트 대신 시각적 증거로 주장을 뒷받침
   - 다이어그램, 차트, 이미지, 코드, 표 등
   - 텍스트가 필요하면 최소한으로, 시각 자료를 보조

3. **제한**: 제목/아젠다/전환 슬라이드는 A-E 불필요. 내용 전달 슬라이드에만 적용

### G-2. 유형별 Assertion-Evidence 변형

| 유형 | Assertion 예시 | Evidence 형태 |
|------|---------------|--------------|
| concept | "REST API는 자원(Resource)을 URI로 표현하고 HTTP 메서드로 조작한다" | 아키텍처 다이어그램, 요청-응답 흐름도 |
| code | "리스트 컴프리헨션은 for 루프보다 30% 빠르고 가독성이 높다" | 코드 비교 블록 (before/after) + 벤치마크 수치 |
| comparison | "Docker는 VM보다 경량이지만 격리 수준은 VM이 더 높다" | 비교표 (2-3열) 또는 벤 다이어그램 |
| data_insight | "2024년 기준 Python이 가장 많이 사용되는 AI 프레임워크 언어다" | 막대 차트/파이 차트 + 핵심 수치 강조 |
| image | "CNN의 합성곱 레이어는 이미지에서 특징(feature)을 순차적으로 추출한다" | CNN 아키텍처 다이어그램, 필터 시각화 |
| summary | "오늘 배운 3가지 핵심: API 설계, 에러 처리, 인증" | 핵심 포인트 아이콘 + 한 줄 요약 (불릿 허용) |
| activity | "실습: 나만의 REST API를 설계하고 Postman으로 테스트한다" | 단계 번호 리스트 (시각적 흐름) + QR/링크 |

### G-3. Assertion-Evidence와 밀도의 관계

- **A-E 전면 적용 시**: 시각 증거에 슬라이드 면적의 60-70%를 할애 → 텍스트 밀도 자연 제한 (8-15줄)
- **고밀도와 A-E의 충돌**: 15-21줄은 시각 증거 공간을 압축 → `high_density`에서는 A-E를 **부분 적용**으로 자동 전환
- **코드 슬라이드**: 코드 자체가 시각 증거 역할 → 15-25줄이어도 A-E 변형 적용 가능

---

## H. 교안→슬라이드 매핑 규칙

### H-1. 차시 구조별 슬라이드 매핑

교안의 도입-전개-정리 구조에서 슬라이드 유형으로의 체계적 매핑:

| 교안 구조 | 교안 요소 | 슬라이드 유형 | 비고 |
|---------|---------|-------------|------|
| **도입** | Hook/동기유발 | `quote` 또는 `image` | 흥미 유발 |
| 도입 | SLO 제시 | `agenda` (차시 수준) | 학습 목표 목록 |
| 도입 | 선수지식 확인 | `quiz` | 사전 점검 퀴즈 |
| **전개** | 개념 설명 | `concept` | A-E 적용 |
| 전개 | 코드 워크스루 | `code` | 줄별 설명 |
| 전개 | 비교/대조 | `comparison` | 표 또는 다이어그램 |
| 전개 | 데이터/통계 | `data_insight` | 차트 + 인사이트 |
| 전개 | 프로세스/순서 | `timeline` | 단계 시각화 |
| 전개 | 실습 활동 | `activity` | 절차 + 시간 |
| 전개 | 형성평가 | `quiz` | 중간 점검 |
| 전개 | 주제 전환 | `section_transition` | 섹션 간 전환 |
| **정리** | 핵심 요약 | `summary` | 3-5 포인트 |
| 정리 | Exit Ticket | `quiz` | 마무리 평가 |
| 정리 | 차시 예고 | `section_transition` | 다음 차시 연결 |

### H-2. 차시당 슬라이드 구성 패턴

**50분 교육용 표준 기준** (~17장):

```
[도입 — 2-3장]
  1. 제목/동기유발 (title/quote/image)
  2. 차시 목표 (agenda)
  3. 선수지식 체크 (quiz) — 선택적

[전개 — 10-15장]
  4. 섹션 A 개념 (concept) × 2-3장
  7. 섹션 A 실습 (activity) × 1장
  8. 중간 체크 (quiz) × 1장 — 선택적
  9. 섹션 전환 (section_transition)
  10. 섹션 B 개념 (concept) × 2-3장
  13. 섹션 B 실습 (activity) × 1장
  14. 형성평가 (quiz) × 1장

[정리 — 2-3장]
  15. 핵심 요약 (summary)
  16. Exit Ticket (quiz) — 선택적
  17. 다음 차시 예고 (section_transition)
```

### H-3. 세션 단위 기획

세션 파일이 주 입력이므로, 세션 단위의 슬라이드 기획 규칙:

| 세션 교안 요소 | 슬라이드 유형 | 비고 |
|-------------|-------------|------|
| 세션 헤더 (차시 제목) | `title` (차시 제목) | 차시 단위 제목 슬라이드 |
| 세션 SLO 목록 | `agenda` (차시 목표) | 차시 학습 목표 안내 |
| 세션 핵심 정리 | `summary` (차시 요약) | 차시 마무리 요약 슬라이드 |
| 다음 차시 예고 | `section_transition` | 차시 간 전환 |

**세션 단위 슬라이드 세트 구조** (8-20장):
```
[차시 제목 — 1장]
  [도입 슬라이드 — 2-3장]
  [전개 슬라이드 — 5-15장]
  [정리 슬라이드 — 1-2장]
```

→ 세션당 8-20장 (콘텐츠 유형 비율 기반 적응형 산출)

### H-4. 교수 모델별 슬라이드 비율

| 교수 모델 | 도입 슬라이드 | 전개 슬라이드 | 정리 슬라이드 | 특징 |
|---------|-----------|-----------|-----------|------|
| 직접교수법 | 10-15% | 60-70% | 15-20% | concept + code 비율 높음 |
| PBL | 10% | 70-75% | 15-20% | activity + quiz 비율 높음, concept 적음 |
| 플립러닝 | 5-10% | 75-80% | 10-15% | activity 비율 최대, quiz(사전학습 확인) |

---

## I. Phase 1 완료 검증 기준

| # | 확인 항목 | 검증 방법 |
|---|----------|----------|
| 1 | `03_slide_plan/01_input_data.json` 파일 존재 | Glob 확인 |
| 2 | `slide_settings` 객체에 필수 필드 존재 (target_scope, tool, density_profile, assertion_evidence, design_tone, speaker_notes) | Read → JSON 파싱 |
| 3 | `source.script_file` 경로의 파일이 실제 존재 | Glob 확인 |
| 3a | `source.session_dir` 경로가 존재하고 세션 파일이 1개 이상 | Glob 확인 (fallback 허용) |
| 4 | `inherited.sessions` 배열이 비어있지 않음 | 길이 검증 |
| 5 | `derived.total_sessions > 0` | 값 검증 |
| 6 | `target_scope.type = "day"`이면 `session_ids` 비어있지 않음 | 조건부 검증 |
| 7 | `tool.selected`가 유효 enum 값 (`marp | slidev | gamma | revealjs`) | 값 검증 |

---

## J. 설계 결정 로그

| # | 결정 | 대안 | 근거 |
|---|------|------|------|
| 1 | 밀도를 3단계 프리셋으로 제공 (교육용 표준/고밀도/프레젠테이션) | 단일 밀도 값 입력 | 사용자에게 줄 수를 직접 입력받는 것은 비현실적. 프리셋으로 유형별 밀도가 자동 결정되어 일관성 보장 |
| 2 | 유형별 차등 밀도 적용 (코드 15-25줄, 개념 8-15줄, 전환 3-5줄) | 모든 유형에 균일 15-21줄 | 인지 부하 이론과 Assertion-Evidence 연구에 따르면, 개념 슬라이드는 시각 증거 공간이 필요하여 텍스트 밀도를 제한해야 함. 코드/실습은 본질적으로 밀도가 높음 |
| 3 | Assertion-Evidence를 3단계로 적용 (전면/부분/불릿) | A-E 전면 강제 | 고밀도 슬라이드에서는 A-E와 충돌. 코드 중심 교육에서는 변형 A-E가 더 적합 |
| 4 | 필수 질문 3개, 선택 질문 4개로 설계 | 모두 필수 | 대부분의 사용자는 기본값으로 충분. 교안 Phase 1의 게이팅 패턴 재사용 |
| 5 | 도구 자동 추천 로직 도입 | 항상 Marp 기본값 | 코드 중심 교육에서 Slidev가 확실히 유리. 콘텐츠 분석 기반 추천이 사용자 경험 향상 |
| 6 | 슬라이드 유형에 퀴즈(quiz) 추가 (12→13가지) | 기존 12가지 유지 | 교안의 형성평가가 슬라이드에 직접 매핑됨. 활동(activity) 유형에 포함시키면 목적이 모호해짐 |
| 7 | 글로벌 Phase 번호 체계 적용 (01, 03, 05, 06, 07) | 워크플로우 내 순번 (01~05) | 구성안·교안과 동일한 파일명 패턴 유지. CLAUDE.md의 `06_write_slide_plan.md` 명명과 일치 |
| 8 | 교안의 발표자 노트를 슬라이드 노트로 변환 옵션 제공 | 발표자 노트 무시 | 교안의 발표자 노트(타이밍, 오개념, 대안활동, 자료체크)는 슬라이드 발표 시에도 유용 |
| 9 | 코드 테마를 조건부 질문으로 설계 | 항상 질문 | 코드 활동이 없는 강의에서는 불필요한 질문. has_code_content 탐지로 자동 분기 |
| 10 | 슬라이드 수 산출을 Phase 3(architecture)에 위임 | Phase 1에서 예상치 제공 | Phase 1은 입력 수집에 집중. 슬라이드 수는 콘텐츠 분석(Phase 2 브레인스토밍) 후 구조 설계(Phase 3)에서 결정이 적합 |
| 11 | 모듈 교안(`06_modules/`)을 주 입력으로, 통합본에서 코스 레벨 메타데이터만 추출 (2계층 분리 로드) | 통합본만 사용 / 차시별만 사용 | v3 결정. v4에서 세션+아키텍처 하이브리드로 전환 (#14 참조) |
| 12 | 차시별 교안(`06_sessions/`)이 아닌 모듈 교안(`06_modules/`)을 선택 | 차시별 교안 직접 사용 | v3 결정. v4에서 세션 파일 직접 사용으로 전환 (#14 참조) |
| 13 | PQ1·PQ4·PQ5·PQ6·PQ7을 이전 단계 산출물에서 자동 추론, 사용자 질문을 PQ2+PQ3으로 축소 (AskUserQuestion 최대 5회→3회) | 모든 PQ를 사용자에게 질문 | PQ1: 교안이 없는 차시는 슬라이드 기획 불가→동기화 필연. PQ4: 밀도→A-E가 결정적 관계→질문 불필요. PQ5: tone 텍스트에 디자인 의도 이미 반영. PQ6: 교안이 항상 발표자 노트 생성→항상 true. PQ7: dark가 코드 가독성 표준→별도 질문 불필요. 세부 조정이 필요한 사용자를 위해 게이팅 후 오버라이드 옵션 유지 |
| 14 | 세션 파일(`06_sessions/`) + 아키텍처 파일(`05_arch_architecture.md`) 하이브리드 입력으로 전환 (4계층 로드) | 모듈 교안(`06_modules/`) 유지 | 세션 파일이 ~200줄 수준으로 차시별 콘텐츠를 직접 제공하여 효율적. 아키텍처 파일이 매크로 구조·시간표·연결 맵을 보완하여 모듈 파일 없이도 구조 맥락 확보 가능 |
| 15 | PQ3 밀도 질문 → 자동 적응형으로 전환 (세션 콘텐츠 유형 분석 기반) | 3개 프리셋 선택 유지 | 프리셋 선택은 사용자에게 실질적 판단 근거를 요구. 세션 파일의 콘텐츠 유형 비율로 유형별 적응형 밀도를 자동 결정하면 더 정확하고 일관성 있는 결과 제공. 사용자 질문 1개 추가 절감 |
| 16 | 적응형 슬라이드 수 (세션 유형별 8-20장) | 고정 공식(session_minutes/N) | 세션 내 콘텐츠 유형 비율에 따라 슬라이드 수가 달라짐. 코드 중심 세션은 적게, 개념 중심 세션은 많게 자동 조정 |

---

## K. 구현 시 주의사항

### K-1. 4계층 교안 파싱 전략

**문제**: `06_write_lecture_script.md` 전체는 수십~수백 페이지의 마크다운 → 컨텍스트 윈도우 초과 위험

**해결**: 4계층 분리 로드로 파싱 복잡도를 분산하고 각 계층의 역할을 명확히 구분

#### L1: 아키텍처 파일 파싱 (구조 레벨)

`01_outline/05_arch_architecture.md` 전체 Read:
1. **매크로 구조**: 일별×교시별 Phase 구분
2. **시간표**: 일별×교시별 그리드
3. **연결 맵**: 세션 간 연결 관계
4. **정렬 매트릭스**: SLO-활동-평가 매핑

#### L2: 세션 파일 파싱 (차시 레벨)

`02_script/06_sessions/` 폴더의 각 `06_session_{NNN}.md`를 순회:
1. **세션 헤더**: 차시 번호, 제목, Phase, SLO
2. **차시 구조**: 도입/전개/정리 하위 구조에서 구조적 정보 추출
   - 전체 교수자 발화를 파싱할 필요 없음. 활동명, 발문, 형성평가, SLO 매핑만 추출
3. **발표자 노트**: 각 차시의 발표자 노트 섹션

→ 세션 파일이 ~200줄 수준으로 각 파일이 관리 가능한 크기

#### L3: 통합본 상단 파싱 (메타데이터)

`02_script/06_write_lecture_script.md`의 처음 ~100줄만 Read:
1. **메타데이터**: §메타데이터 테이블에서 키-값 추출 (단순 테이블 파싱)
2. **SLO**: §2 테이블에서 SLO 목록 추출

→ 차시별 스크립트 본문은 이 단계에서 **읽지 않는다**.

#### L4: 보조 JSON 파싱

1. `02_script/01_input_data.json` → teaching_model, script_settings
2. `01_outline/01_input_data.json` → tone_examples, lab_environment, keywords

#### 파싱 실패 시 fallback 체인

| 실패 시나리오 | fallback |
|-------------|----------|
| 세션 폴더(`06_sessions/`) 없음 | 통합본(`06_write_lecture_script.md`) 전체 파싱 (경고 출력) |
| 통합본도 없음 | 오류: "`/lecture-script`를 먼저 완료하세요." → 종료 |
| 아키텍처 파일(`05_arch_architecture.md`) 없음 | 통합본 상단에서 시간표·SLO 추출 (경고 출력) |
| 개별 세션 파싱 실패 | 해당 세션만 스킵, 경고 기록, 나머지 세션 계속 |

### K-2. AskUserQuestion 선택지 수 제약

AskUserQuestion은 선택지 2~4개 제한:
- PQ2: 4개 (Marp/Slidev/Gamma/reveal.js) — 사용자 질문
- 자동 추론 확인: 2개 (그대로 진행/세부 설정) — 게이트
- PQ4: 3개 (전면/부분/불릿) — 게이트 통과 시 조정 가능
- PQ5: 3개 (교육적/전문적/미니멀) — 게이트 통과 시 조정 가능
- PQ7: 3개 (dark/light/monokai) — 게이트 통과 + has_code_content 시에만
- ~~PQ1, PQ3, PQ6: v4에서 자동 추론으로 전환, 사용자 질문 불필요~~

### K-3. 강의 폴더 선택 시 엣지 케이스

교안 Phase 1과 동일한 패턴 적용:

| 상황 | 처리 |
|------|------|
| `lectures/` 없음 | 오류: "`/lecture-outline`을 먼저 실행하세요." → 종료 |
| 폴더 0개 | 동일 오류 |
| 폴더 1개 | 해당 폴더 + "기타 직접 입력" = 2개 선택지 |
| 폴더 2개 | 2개 폴더 + "기타 직접 입력" = 3개 선택지 |
| 폴더 3개+ | 최신 3개 + "기타 직접 입력" = 4개 선택지 |
| `02_script/06_sessions/` 폴더 없음 | fallback: `06_write_lecture_script.md` 전체 파싱 (경고 출력) |
| `02_script/06_write_lecture_script.md` 없음 | 오류: "`/lecture-script`를 먼저 완료하세요." → 종료 |
| `01_outline/05_arch_architecture.md` 없음 | 통합본 상단에서 시간표 추출 (경고 출력) |

### K-4. 도구별 레이아웃 제약 전달

Phase 4(writer-agent)에서 도구별 제약을 반영하려면, Phase 1에서 도구 선택 정보를 정확히 전달해야 한다:

| 도구 | 코드 하이라이트 | 레이아웃 | 이미지 삽입 | 수식 | 인터랙션 |
|------|-------------|---------|-----------|------|---------|
| Marp | Prism.js | 기본 마크다운 | 마크다운 이미지 | KaTeX | 제한적 |
| Slidev | Shiki (줄별) | Vue 컴포넌트 가능 | 마크다운+Vue | KaTeX/MathJax | 라이브 코딩, 클릭 |
| Gamma | 내장 | 블록 기반 | 드래그&드롭 | 제한적 | AI 추천 |
| reveal.js | highlight.js | HTML/CSS 자유 | HTML img | MathJax | 플러그인 확장 |

---

## L. 다음 단계 (Phase 2) 연계

Phase 1 완료 후 `01_input_data.json`이 생성되면, Phase 2(brainstorm-agent)의 입력이 된다.

슬라이드 기획 Phase 2 브레인스토밍 방향:
- **시각화 아이디어**: 각 개념을 어떤 시각적 형태로 표현할 것인가
- **레이아웃 패턴**: 유형별 레이아웃 변형 (2열, 3열, 전면 이미지 등)
- **인터랙션 요소**: 클릭 애니메이션, 단계별 공개, 빈칸 채우기 등
- **메타포 시각화**: `tone_examples`의 메타포를 슬라이드 시각 요소로 변환
- **색상/아이콘 체계**: `design_tone`에 맞는 시각 시스템

Phase 2 brainstorm-agent는 `slide_settings`와 `inherited.sessions`를 읽어 차시별 시각화 전략을 도출한다.
