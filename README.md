# OpenClaw Multi-Agent Orchestration OS — Setup Guide

> **결정론적(Deterministic) 멀티 에이전트 제어 시스템**  
> 1명의 총괄 오케스트레이터(`Main`) + 6개의 전문 서브 에이전트가 협업하는 엔터프라이즈급 개발/문서 자동화 프레임워크

---

## 📖 개념 및 설계 철학

### 왜 멀티 에이전트인가?

단일 LLM에게 "앱 만들어줘"라고 하면 환각, 누락, 일관성 붕괴가 발생한다. 이 시스템은 하나의 거대한 프롬프트 대신 **7개의 전문화된 에이전트**가 각자 맡은 역할만 수행하고, `Main` 오케스트레이터가 이들을 **결정론적 상태 머신**으로 통제한다.

### 핵심 설계 원칙

| 원칙 | 설명 |
|------|------|
| **포인터 기반 통신** | 에이전트 간 데이터는 값(Value)이 아닌 파일 경로(Pointer)로만 전달. Main의 컨텍스트 윈도우 보호. |
| **서브 에이전트 간 직접 소통 금지** | 모든 통신은 Main을 경유. 서브 에이전트끼리 대화하지 않는다. |
| **정량 평가 기반 루프** | Architect가 테스트 기준을 설계 → Forge가 구현 → Verify가 기계적으로 검증. LLM의 주관적 평가 배제. |
| **서킷 브레이커** | 동일 태스크 3회 연속 실패 시 자동 잠금(LOCKED). 무한 재시도로 인한 토큰 소진 방지. |
| **상태 기반 라우팅** | Main의 모든 판단은 `project_state.json` 파일에 기록된 상태값에만 의존. 대화 흐름에 휩쓸리지 않음. |

### 에이전트 토폴로지

```
                          ┌─────────────┐
                          │   👤 사용자   │
                          └──────┬──────┘
                                 │
                          ┌──────▼──────┐
                          │  🏠 Main    │  ← deepseek-v4-flash
                          │ 오케스트레이터 │     모든 진입점, 라우팅, 상태 관리
                          └──┬──┬──┬──┬─┘
                             │  │  │  │
        ┌────────┬───────────┘  │  │  └──────────┬──────────┐
        │        │              │  │              │          │
   ┌────▼──┐ ┌──▼───┐   ┌─────▼┐ ┌▼──────┐  ┌───▼───┐ ┌───▼───┐
   │🏗 Arch │ │🎨 Ill │   │🔨 Frg│ │✅ Ver │  │🎬 Dir │ │✍️ Scr │
   │ 설계   │ │ 이미지 │   │ 구현  │ │ 검증  │  │ 리서치 │ │ 문서화 │
   └───────┘ └──────┘   └──────┘ └───────┘  └───────┘ └───────┘
   v4-flash   G 2.5-fl   v4-pro    v4-flash   G 2.5-fl  Qwen 3.5
```

---

## 🗂️ 파일 구조

```
openclaw_setup_dev/
├── README.md                    ← 본 파일
├── .gitignore
│
├── AGENTS.md                    ← 토폴로지 · 통신 프로토콜 · 라우팅 규칙
├── SOUL.md                      ← 통제 철학 · 서킷 브레이커 · Critical Review
├── TOOLS.md                     ← 인프라 명세 · API 매핑 · 툴 매트릭스
│
├── agents/                      ← 서브 에이전트 명세 (6종)
│   ├── architect.md             ← 시스템 설계 · UI 텍스트 기획 · 평가 스키마
│   ├── forge.md                 ← 코드 구현 · 통합
│   ├── verify.md                ← 정량 검증 · exec 실행 · 디버깅
│   ├── illustrator.md           ← 이미지/에셋 생성 (Gemini)
│   ├── director.md              ← 멀티모달 리서치 (Skills 기반)
│   └── scribe.md                ← 마크다운 → HTML/문서 변환
│
└── state/
    ├── project_state.json       ← 전역 프로젝트 상태 트리 (빈 초기값)
    └── README.md                ← 상태 스키마 레퍼런스
```

---

## 🚀 새 PC에서 설치하기

### 0. 사전 요구사항

| 항목 | 최소 버전 | 확인 커맨드 |
|------|----------|------------|
| Node.js | v22+ | `node --version` |
| Python | 3.10+ | `python3 --version` |
| Git | any | `git --version` |
| WSL2 (Windows) | any | `wsl --status` |

### 1. OpenClaw Gateway 설치

```bash
# npm으로 글로벌 설치
npm install -g openclaw

# 설치 확인
openclaw --version
```

### 2. Repository 클론

```bash
# 워크스페이스 생성
mkdir -p ~/.openclaw/workspace
cd ~/.openclaw/workspace

# 이 레포 클론
git clone git@github.com:steviieparker-collab/openclaw_setup_dev.git .

# 또는 HTTPS (GitHub CLI 인증 필요)
# gh repo clone steviieparker-collab/openclaw_setup_dev .
```

### 3. 환경변수(.env) 설정

```bash
# ~/.openclaw/.env 파일 생성
cat > ~/.openclaw/.env << 'ENVEOF'
# API Keys
DEEPSEEK_API_KEY=sk-your-deepseek-key
DEEPSEEK_PRO_API_KEY=sk-your-deepseek-pro-key
GEMINI_API_KEY=your-gemini-key
QWEN_API_KEY=your-qwen-key

# GitHub
GIT_HUB_TOKEN=ghp_your_token
ENVEOF

# 권한 제한
chmod 600 ~/.openclaw/.env
```

### 4. 프로젝트 디렉토리 구조 생성

```bash
cd ~/.openclaw/workspace

# 프로젝트 템플릿 디렉토리 (Architect가 프로젝트 시작 시 자동 생성하지만, 수동 가능)
mkdir -p projects
mkdir -p memory/threads
mkdir -p skills
```

### 5. OpenClaw Gateway 설정 및 시작

```bash
# Gateway 설정 (최초 1회)
openclaw gateway config

# Gateway 시작
openclaw gateway start

# 상태 확인
openclaw gateway status
```

### 6. 에이전트 등록

OpenClaw Gateway가 실행 중인 상태에서 각 에이전트를 등록한다:

```bash
# Main 에이전트 등록 (기본값)
openclaw agent create main \
  --model deepseek/deepseek-v4-flash \
  --fallback google/gemini-2.5-flash \
  --workspace ~/.openclaw/workspace

# 서브 에이전트 등록 (6종)
# Architect
openclaw agent create architect \
  --model deepseek/deepseek-v4-flash \
  --fallback google/gemini-2.5-flash \
  --workspace ~/.openclaw/workspace \
  --system-prompt "$(cat ~/.openclaw/workspace/agents/architect.md)"

# Forge
openclaw agent create forge \
  --model deepseek/deepseek-v4-pro \
  --fallback google/gemini-2.5-flash \
  --workspace ~/.openclaw/workspace \
  --system-prompt "$(cat ~/.openclaw/workspace/agents/forge.md)"

# Verify
openclaw agent create verify \
  --model deepseek/deepseek-v4-flash \
  --fallback google/gemini-2.5-flash \
  --workspace ~/.openclaw/workspace \
  --system-prompt "$(cat ~/.openclaw/workspace/agents/verify.md)"

# Illustrator
openclaw agent create illustrator \
  --model google/gemini-2.5-flash \
  --workspace ~/.openclaw/workspace \
  --system-prompt "$(cat ~/.openclaw/workspace/agents/illustrator.md)"

# Director
openclaw agent create director \
  --model google/gemini-2.5-flash \
  --fallback google/gemini-3.1-flash-lite \
  --workspace ~/.openclaw/workspace \
  --system-prompt "$(cat ~/.openclaw/workspace/agents/director.md)"

# Scribe
openclaw agent create scribe \
  --model qwen/qwen3.5-flash \
  --fallback qwen/qwen3.5-plus \
  --workspace ~/.openclaw/workspace \
  --system-prompt "$(cat ~/.openclaw/workspace/agents/scribe.md)"
```

> **참고:** `openclaw agent create`의 정확한 CLI 옵션은 Gateway 버전에 따라 다를 수 있다. `openclaw agent create --help`로 확인할 것. 또는 Gateway 설정 UI에서 직접 등록 가능.

---

## 🔄 작동 흐름 (Phase 0 → 4)

### PHASE 0: 기획
- **담당:** Main (사용자와 직접 대화)
- Main이 요구사항 분석, 프로젝트 범위 확정
- `state/project_state.json`에 프로젝트 엔트리 생성
- 사용자 승인 후 Phase 1 진입

### PHASE 1: 설계
- **담당:** Architect (+ Director, Illustrator 선택적)
- 1) Director가 필요 시 리서치 수행 → `RESEARCH/`
- 2) Architect가 시스템 설계, 노드 분해, UI 텍스트 명세, 평가 스키마 → `PLAN/`
- 3) Illustrator가 필요 시 UI 목업 생성 → `ASSETS/`
- 사용자 설계 리뷰 승인 후 Phase 2 진입

### PHASE 2: 구현
- **담당:** Forge (Main이 노드별로 순차 할당)
- `PLAN/` 명세서 기반 단일 노드씩 구현 → `CODE/`
- 각 노드 검증 완료 후 다음 노드 진행
- 선행 노드 출력이 다음 노드 입력으로 파이프라인 연결

### PHASE 3: 검증
- **담당:** Verify
- `PLAN/03_test_spec.md`의 정량적 기준으로 코드 실행 및 평가 → `REPORT/`
- **통과:** 다음 노드로
- **실패(retry < 3):** `fix_suggestion`과 함께 Forge에게 재할당
- **실패(retry ≥ 3):** 서킷 브레이커 발동 → LOCKED → 사용자 개입 요청

### PHASE 4: 통합
- **담당:** Forge (통합) + Verify (E2E 테스트)
- 노드 결합, 파일 I/O 제거, 디버깅 코드 정리
- 최종 통합 E2E 테스트
- 사용자 최종 승인 → COMPLETE

---

## 📡 통신 프로토콜 (JSON Envelope)

모든 서브 에이전트는 작업 완료 시 자연어 대신 아래 JSON으로만 Main에게 응답한다:

```json
{
  "agent_id": "forge",
  "task_id": "TASK_102",
  "status": "SUCCESS",
  "summary": "JWT 인증 모듈 구현 완료",
  "artifact_paths": ["projects/sample/CODE/auth/jwt_handler.py"],
  "evaluation": {
    "spec_path": "projects/sample/PLAN/03_auth_test_spec.md",
    "pass_conditions": {"test_exit_code": 0, "lint_errors": 0},
    "results": {"test_exit_code": 0, "lint_errors": 0},
    "all_passed": true
  },
  "error_log": null,
  "fix_suggestion": null,
  "next_recommended_action": "pass"
}
```

**핵심:** `artifact_paths`는 파일 경로만 담는다. Main은 코드/문서 내용을 직접 읽지 않고, 포인터만 다음 에이전트에게 전달한다.

---

## 🛡️ 서킷 브레이커 (무한 루프 방지)

- 동일 태스크의 `retry_count`가 **3회**에 도달하면:
  - 태스크 상태가 `LOCKED`로 변경됨
  - 해당 에이전트 호출 중단
  - 사용자에게 "TASK_ID [X] 잠금. 수동 개입 필요" 보고
- 사용자가 명시적으로 잠금을 해제할 때까지 절대 재시도하지 않음

---

## 📁 프로젝트별 폴더 구조

각 프로젝트는 `projects/<project_name>/` 아래 다음 구조로 관리된다:

```
projects/<project_name>/
├── PLAN/              ← Architect (설계서, 명세서, 평가 스키마)
├── CODE/              ← Forge (구현된 소스코드)
├── REPORT/            ← Verify (검증 결과 리포트)
├── ASSETS/            ← Illustrator (생성된 이미지/에셋)
├── RESEARCH/          ← Director (리서치 결과, 추출 데이터)
├── OUTPUT/            ← Scribe (최종 HTML/문서)
│   ├── draft/
│   └── final/
├── state/             ← 프로젝트별 로컬 상태
├── data/              ← 런타임 데이터
└── docs/              ← 참고 문서
```

---

## ⚡ 빠른 시작: 새 프로젝트 만들기

### 사용자 → Main

```
"프로젝트를 하나 시작하자. FastAPI로 유저 인증 + CRUD API를 만들 거야."
```

### Main의 응답 흐름

1. 요구사항 비판적 검토 (SOUL.md Critical Review 11체크포인트)
2. `state/project_state.json`에 프로젝트 엔트리 생성
3. Phase 0 기획 노트 작성 → 사용자 승인 대기

### 사용자 → Main

```
"진행해."
```

### Main → Architect 호출

```
sessions_spawn(
  agent="architect",
  task="[TASK_ID: TASK_101] [PROJECT: projects/auth_api] [PHASE: DESIGN] ..."
)
```

### Architect → PLAN/ 파일 생성 → Main에게 JSON Envelope 응답

### Main → 사용자에게 설계 리뷰 요청

### ... Phase 순차 진행

---

## 🔧 문제 해결

| 문제 | 해결 |
|------|------|
| Gateway 시작 실패 | `openclaw gateway logs` 확인, Node.js 버전 체크 |
| API 키 인증 실패 | `.env` 파일 경로 및 키 값 확인 |
| 에이전트가 JSON 대신 자연어 응답 | 해당 에이전트의 system prompt에 `agents/*.md`가 제대로 로드되었는지 확인 |
| exec 타임아웃 | Verify의 타임아웃 기본값은 10초. 복잡한 테스트는 `timeout` 값 증가 |
| 서킷 브레이커 발동 | `project_state.json`에서 해당 태스크의 `locked_reason` 확인. 문제 해결 후 `locked_at: null`로 수동 해제 |

---

## 📚 참조

- **OpenClaw Docs:** https://docs.openclaw.ai
- **OpenClaw GitHub:** https://github.com/openclaw/openclaw
- **이 레포:** https://github.com/steviieparker-collab/openclaw_setup_dev
