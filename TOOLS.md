# TOOLS.md — Infrastructure & Environment Specifications

본 파일은 Main 및 서브 에이전트들이 작업을 수행하는 물리적 환경, 허용된 도구, API 접근 권한 및 보안 경계를 정의한다.

## 1. 시스템 환경

| 항목 | 값 |
|------|-----|
| OS | WSL2, Ubuntu (Linux 6.6.87.2-microsoft-standard-WSL2, x64) |
| Gateway | OpenClaw (node v22.22.1) |
| Python | /usr/bin/python3 |
| Node | v22.22.1 |
| 패키지 매니저 | pip3, npm |
| Git | GitHub: github.com/steviieparker-collab/ |

## 2. 작업 디렉토리

```
workspace_dev/                  ← Main 오케스트레이터 메인 워크스페이스
├── agents/                     ← 서브 에이전트 .md 명세 파일
│   ├── architect.md
│   ├── forge.md
│   ├── verify.md
│   ├── illustrator.md
│   ├── director.md
│   └── scribe.md
├── projects/                   ← 모든 프로젝트 (프로젝트별 하위 폴더)
│   └── <project_name>/
│       ├── PLAN/               ← Architect 산출물
│       ├── CODE/               ← Forge 산출물
│       ├── REPORT/             ← Verify 산출물
│       ├── ASSETS/             ← Illustrator 산출물
│       ├── RESEARCH/           ← Director 산출물
│       ├── OUTPUT/             ← Scribe 산출물
│       │   ├── draft/
│       │   └── final/
│       ├── state/              ← 프로젝트별 로컬 상태
│       ├── data/               ← 런타임 데이터
│       └── docs/               ← 참고 문서
├── state/                      ← 전역 상태
│   └── project_state.json      ← Main 전용 상태 트리
├── memory/                     ← 사용자 장기 기억 및 일일 로그
├── skills/                     ← ClawHub 스킬 (Director용)
├── data/                       ← 공용 데이터
├── codes/                      ← 자체 제작 스크립트
├── .learnings/                 ← 학습 기록
├── .env                        ← 환경변수 (API 키)
├── AGENTS.md                   ← 토폴로지/프로토콜 (Main 제어용)
├── SOUL.md                     ← 통제 철학/서킷 브레이커 (Main 제어용)
└── TOOLS.md                    ← 본 파일 (인프라 명세)
```

## 3. Environment Variables & API Mapping

코드에 API 키를 하드코딩하는 것을 엄격히 금지한다. 모든 에이전트는 `.env` 파일의 환경변수만 참조한다.

| 에이전트 | 모델 | 환경변수 |
|---------|------|---------|
| Main | deepseek/deepseek-v4-flash | `DEEPSEEK_API_KEY` |
| Architect | deepseek/deepseek-v4-flash | `DEEPSEEK_API_KEY` |
| Forge | deepseek/deepseek-v4-pro | `DEEPSEEK_PRO_API_KEY` |
| Verify | deepseek/deepseek-v4-flash | `DEEPSEEK_API_KEY` |
| Illustrator | google/gemini-2.5-flash | `GEMINI_API_KEY` |
| Director | google/gemini-2.5-flash | `GEMINI_API_KEY` |
| Scribe | qwen/qwen3.5-flash | `QWEN_API_KEY` |

**Fallback 체인 (API 장애 시):**
- DeepSeek 계열 (Main/Architect/Verify): `deepseek/deepseek-v4-flash` → `google/gemini-2.5-flash`
- Forge: `deepseek/deepseek-v4-pro` → `google/gemini-2.5-flash`
- Director: `google/gemini-2.5-flash` → `google/gemini-3.1-flash-lite`
- Illustrator: `google/gemini-2.5-flash` (fallback 없음 — 이미지 생성 대체 모델 없음)
- Scribe: `qwen/qwen3.5-flash` → `qwen/qwen3.5-plus`

## 4. Allowed Tools Matrix (에이전트별 툴 매트릭스)

| 툴 | Main | Architect | Forge | Verify | Illustrator | Director | Scribe |
|----|------|-----------|-------|--------|-------------|----------|--------|
| `sessions_spawn` (서브에이전트 호출) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `read` / `write` / `edit` (파일 I/O) | ✅ (state만) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `exec` (코드 실행) | ❌ | ❌ | ❌ | ✅ (타임아웃 10s) | ❌ | ❌ | ❌ |
| `web_search` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `web_fetch` | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ |
| `pdf` | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ |
| `image` (분석) | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | ❌ |
| `image_generate` (생성) | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| `memory_search` / `memory_get` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `cron` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

**규칙:**
- ✅ = 해당 에이전트가 사용 가능
- ❌ = 사용 금지. 시도 시 작업 실패 처리
- 파일 I/O는 반드시 자신에게 할당된 프로젝트 폴더 내로 제한 (`../` 경로 탈출 시도 금지)

## 5. Verify Agent 실행 제약

`Verify`가 `exec`를 사용하여 코드를 실행할 때:
- **타임아웃:** 10,000ms (초과 시 프로세스 kill 및 `TIMEOUT` 상태 반환)
- **작업 디렉토리:** 프로젝트 루트 (`projects/<name>/`)
- **실행 커맨드 예시:**
  ```bash
  cd projects/<name> && timeout 10 python3 CODE/<task>/<node>.py
  ```
- **Lint 체크 예시:**
  ```bash
  python3 -m flake8 projects/<name>/CODE/<task>/ --max-line-length=120
  ```
- 테스트 실행 전 반드시 `PLAN/03_*_test_spec.md`를 읽고 pass_conditions를 파싱할 것

## 6. Director Skills 참조

Director는 다음 Skill 파일들을 task에 따라 로드하여 사용한다:

| Skill | 경로 | 용도 |
|-------|------|------|
| `document-extraction` | `skills/document-extraction/SKILL.md` | PDF/이미지 → 마크다운 추출 |
| `media-analysis` | `skills/media-analysis/SKILL.md` | 영상/오디오/회의록 → 핵심 요약 |
| `market-research` | `skills/market-research/SKILL.md` | 웹 검색/트렌드 조사 → 리포트 |
| `data-curation` | `skills/data-curation/SKILL.md` | 다중 소스 데이터 수집 → 통합 |

## 7. 사용 가능한 Python 패키지 (주요)

- `pyepics` (EPICS CA 통신)
- `yfinance` (주식 정보)
- `requests` (HTTP)
- `python-docx`, `openpyxl`, `python-pptx` (문서 처리)
- `markitdown` (`~/.venv/markitdown/`)
- `flake8` (린트)

## 8. SSH / 원격 접속

- 포항가속기 내부망: EPICS IOC (192.168.1.x)
- ngrok: 외부망 터널링

## 9. 프로젝트 목록 (기존)

`workspace_dev/projects/` 하위:
- `coachflow`
- `irongate`
- `KSU_remote_control`
- `runcrew`
- `SRF_postmortem`
- `triathineer`

## 10. Red Lines (인프라 측면)

- **`rm -rf` 금지:** 파일 덮어쓰기/수정만 허용. 삭제 시 `.bak` 백업 후 수행.
- **워크스페이스 외부 접근 금지:** 모든 파일 I/O는 `workspace_dev/` 내부로 제한.
- **외부 발송 금지:** 이메일, 메시징, API POST 등 외부 액션은 사용자 승인 필수.
- **Windows ↔ WSL 경로 변환:** 필요 시 `wslpath` 사용.
- **cron 작업:** WSL 내 systemd 또는 직접 스크립트 실행.
