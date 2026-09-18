# Evonic 분석 & 활용 가이드 (한국어)

> 작성일: 2026-09-18
> 대상 저장소: <https://github.com/bmshin94/evonic>
> 원본(업스트림): <https://github.com/anvie/evonic>
> 공식 문서: <https://evonic.dev>

Claude Code 세션에서 진행한 Evonic 저장소 분석 대화를 정리한 문서입니다.

---

## 목차

1. [Evonic이란?](#1-evonic이란)
2. [쉽게 이해하기](#2-쉽게-이해하기)
3. [설치 및 사용법](#3-설치-및-사용법)
4. [플러그인? 스킬? MCP?](#4-플러그인-스킬-mcp)
5. [API 토큰이 필요한가?](#5-api-토큰이-필요한가)
6. [왜 GitHub에서 주목받나?](#6-왜-github에서-주목받나)
7. [로컬 에이전트 구축에 도움이 될까?](#7-로컬-에이전트-구축에-도움이-될까)
8. [React / PHP로 만들 수 있을까?](#8-react--php로-만들-수-있을까)
9. [수익화 아이디어](#9-수익화-아이디어)
10. [실행 로드맵](#10-실행-로드맵)

---

## 1. Evonic이란?

**에이전트 AI를 설계·배포·운영하는 올인원 플랫폼**입니다. 라이브러리가 아니라
웹 UI와 CLI를 갖춘 완제품 서버입니다.

| 항목 | 내용 |
|---|---|
| 버전 | v1.2.0 |
| 제작자 | Robin Syihab (anvie) |
| 언어 | Python 3.10+ (586개 파일, 약 156,000줄) + Go (Evonet 커넥터) |
| 웹 스택 | Flask + Jinja2 + Tailwind (템플릿 24개, 라우트 20개) |
| 라이선스 | AGPL-3.0 (비상업) / 상업용 별도 라이선스 (`COMMERCIAL.md`) |
| 상업 라이선스 문의 | robin@syihab.st |

### 디렉터리 구조

```
evonic/
├── app.py              # Flask 메인 서버 (33KB)
├── evonic              # CLI 래퍼 스크립트
├── routes/             # 웹 라우트 20개 (agents, models, skills, plugins,
│                       #   workplaces, scheduler, evaluation, rtk, safety_rules ...)
├── backend/
│   ├── agent_runtime/  # 핵심 런타임 (llm_loop, llm_tool_executor,
│   │                   #   memory_manager, explorer, summarizer, approval, cmp/)
│   ├── channels/       # telegram / discord / whatsapp(Node 사이드카)
│   ├── provider/       # codex_client, oauth_codex
│   ├── promptpurify/   # 프롬프트 인젝션 방어
│   └── plugin_*.py     # 플러그인 매니저 / 훅 / 핫리로드 / SDK
├── skills/             # 14개 (github, exa-search, kanban, explorer, subagent,
│                       #   scheduler, panel, obscura, direxplorer ...)
├── plugins/            # 11개 (mcp_client, agentapi, auto_improver, model-router,
│                       #   token_monitor, github_webhook, workflow_guard ...)
├── skillsets/          # 직군 템플릿 8개 (coder, devops, pentester, data_analyst,
│                       #   sysadmin, reverse_engineer, fullstack_dev, customer_service)
├── tools/              # 툴 정의 JSON 30개 (bash, runpy, read/write_file, patch,
│                       #   str_replace, sshc, describe_image, monitor ...)
├── evonet/             # Go 바이너리 — 원격 장비 연결 커넥터
├── evaluator/          # LLM 평가 엔진
├── improver/           # 자동 개선 파이프라인 (analyzer → generator → comparator)
├── models/             # SQLite 스키마 (chat, session_archive, llm_trace)
├── lib/envcrypt/       # 환경변수 암호화
├── scripts/            # build_training_dataset.py 등 유틸
└── docker/tools/       # bash/runpy 툴 실행용 샌드박스 이미지
```

### 3대 차별점

1. **Workplace — 어디서든 실행**
   - `local` (호스트 샌드박스) / `ssh` (원격 서버) / `tunnel` (Evonet 커넥터)
   - Evonet은 기기에서 **아웃바운드 WebSocket**으로 접속하므로 공인 IP, 포트포워딩,
     방화벽 규칙이 전부 불필요

2. **Agent-to-Agent 통신이 1급 프로토콜**
   - 멀티 에이전트 스웜, 계층형 오케스트레이션, P2P 협업
   - 에이전트별 Messaging ACL(화이트/블랙리스트) 지원

3. **휴리스틱 악성 행위 탐지**
   - 대량 파일 삭제, 권한 상승, 무단 원격 실행, 행동 드리프트를 **실행 전에** 차단
   - 의심 시 사람에게 에스컬레이션 (`backend/agent_runtime/approval.py`)
   - 별도 Injection Guard(`backend/promptpurify/`)로 프롬프트 인젝션 방어

### 주요 기능

| 기능 | 설명 |
|---|---|
| Agents | 툴 · KB · 격리 워크스페이스를 가진 독립 LLM 어시스턴트 |
| Models | OpenAI 호환 API면 모두 지원 (로컬/클라우드) |
| Skills | 툴 정의 + Python 백엔드 번들. load → context → execute 라이프사이클 |
| Plugins | 이벤트 기반 확장 (`turn_complete` 등 훅) |
| Knowledge Graph | `[[Doc Title]]` 위키링크 + force-directed 그래프 시각화 |
| KB Organizer | 엔티티 추출 · 중복 제거 · 위키링크 연결 자동 수행 서브 에이전트 |
| Memory Engine | Evomem(시맨틱 + 지식그래프), 미설치 시 SQLite FTS5 폴백 |
| Token Compressor | RTK 기반 컨텍스트 압축으로 LLM 비용 절감 |
| Training Data Archive | LLM 요청/응답을 바이트 단위로 저장 → SFT 데이터셋 생성 |
| Evaluation Engine | regex / 휴리스틱 / LLM 기반 자동 평가 |
| Scheduler | 크론 트리거, 반복 작업, 리마인더 |
| Channels | Telegram ✅ / WhatsApp ✅ / Discord ✅ / Slack 🔄예정 |
| Backup & Restore | 전체 설정 · KB · 데이터 백업 |

---

## 2. 쉽게 이해하기

Evonic = **"AI 직원 관리 회사 시스템"**

### 에이전트 = AI 직원 한 명

| 항목 | 쉬운 설명 |
|---|---|
| Concept | 성격/역할 설명서 (시스템 프롬프트) |
| Model | 두뇌 (어떤 LLM을 쓸지) |
| Tools | 쓸 수 있는 도구 (터미널, 파일, 웹검색) |
| Knowledge Base | 업무 매뉴얼 폴더 |
| Channels | 일하는 창구 (텔레그램, 디스코드, 웹) |
| Skills | 추가 특기 (깃허브, 칸반 등) |

핵심은 **코딩 없이 웹 화면에서 클릭으로 에이전트를 만든다**는 점입니다.

### Workplace = 출근하는 사무실

- `local` — 내 PC 안 격리된 도커 컨테이너
- `ssh` — 원격 서버로 출근
- `tunnel` — 내 기기가 **먼저 바깥으로 전화를 거는** 방식이라 방화벽 설정이 전혀 불필요

### 경쟁 제품과의 위치

| | 정체 | 웹 UI | 비유 |
|---|---|---|---|
| LangChain | 라이브러리 | ❌ | 레고 부품 상자 |
| AutoGPT | 실험 프로젝트 | △ | 장난감 로봇 |
| Dify / n8n | 워크플로우 툴 | ✅ | 순서도 그리기 |
| **Evonic** | **완제품 플랫폼** | ✅ | **AI 직원 회사** |

> 포지션 요약: "Dify인데, 실제 서버에 SSH로 들어가 명령을 실행하는 개발자용 버전"

---

## 3. 설치 및 사용법

### 원클릭 설치

```bash
curl -fsSL https://evonic.dev/install.sh | bash
```

`install.sh`가 수행하는 6단계:

1. `git`, `python3` 확인 (**Python 3.9+ 필수**, README 권장 3.10+)
2. `venv` + `ensurepip` 모듈 확인
3. 최신 태그로 `git clone --depth 1`
4. `.venv` 생성 후 `requirements.txt` 설치
5. **evomem 메모리 엔진 바이너리 자동 프로비저닝** (실패 시 FTS5 폴백)
6. `evonic`을 PATH에 등록 (`.bashrc` / `.zshrc` 수정)

### 수동 설치

```bash
git clone https://github.com/anvie/evonic
cd evonic
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
chmod +x ./evonic

./evonic setup      # 대화형 초기 설정 마법사
./evonic pass       # 관리자 비밀번호 설정
./evonic start      # http://localhost:8080
```

### 도커 샌드박스 (권장)

```bash
docker build -t evonic-sandbox:latest docker/tools/
```

빌드하지 않으면 `bash` / `runpy` 툴이 **호스트에서 직접 실행**됩니다.
도커를 못 쓰면 `.env`에 `sandbox_enabled=0`.

### 자주 쓰는 CLI

```bash
./evonic status                  # 서버 상태
./evonic doctor                  # 진단 (문제 생기면 우선 실행)
./evonic restart                 # 데몬 재시작
./evonic backup / restore        # 백업 · 복원
./evonic clear-sandbox           # 샌드박스 컨테이너 정리

./evonic model add gpt4o --provider openai --api-key "sk-..." --base-url "https://api.openai.com/v1"
./evonic agent add dev_bot --name "개발봇" --skillset coder
./evonic skill add path/to/skill.zip
./evonic plugin install path/to/plugin.zip
./evonic workplace create --name "prod-server" --type remote
```

---

## 4. 플러그인? 스킬? MCP?

**셋 중 어느 것도 아닙니다. Evonic은 그것들을 담는 *플랫폼*입니다.**

Claude Code에 설치하는 확장이 아니라, 별도로 실행하는 독립 서버입니다.

| 구분 | Claude Code | Evonic |
|---|---|---|
| 정체 | CLI 에이전트 도구 | 웹 서버 + 에이전트 런타임 |
| 실행 | `claude` | `./evonic start` → :8080 |
| 스킬 | `.claude/skills/*.md` | `skills/` (`skill.json` + `tools.json` + Python 백엔드) |
| 플러그인 | 마켓플레이스 | `plugins/` (이벤트 훅 기반) |
| MCP | 클라이언트 | 클라이언트 (플러그인 형태) |

### Evonic 내부 3층 구조

**① Tools** — `tools/*.json` 30개. LLM function calling 스키마 그대로.

**② Skills** — 툴 + Python 백엔드 번들:

```
skills/hello_world/
├── skill.json      # id, name, version, tools_file, variables
├── tools.json      # LLM에 노출할 함수 스키마
├── SYSTEM.md       # 스킬 활성 시 추가되는 프롬프트
└── backend/tools/hello_world.py   # 실제 구현
```

load → context → execute 라이프사이클이라 미사용 시 시스템 프롬프트에 올라가지
않아 토큰이 절약됩니다.

**③ Plugins** — 이벤트 기반 확장 (`"events": ["turn_complete"]` 등).

### MCP 지원 (양방향)

- **들어오는 방향**: `plugins/mcp_client` — 외부 MCP 서버를 stdio로 연결.
  기본 프리셋이 **Claude Code**입니다:

  ```json
  {"claude-code": {"command": "claude", "args": ["mcp", "serve"], "enabled": true}}
  ```

  → Evonic 에이전트가 Claude Code를 MCP 서버로 호출할 수 있습니다.

- **나가는 방향**: `plugins/agentapi` — Evonic 에이전트를 **OpenAI 호환 REST API**로
  외부에 노출. Bearer 토큰 인증 + 쿼터 + 모델 스코핑 지원.

---

## 5. API 토큰이 필요한가?

### ① LLM API 키 — 사실상 필수 (단, 무료 경로 있음)

`.env.example` 발췌:

```bash
LLM_BASE_URL=https://openrouter.ai/api/v1
LLM_API_KEY=your-api-key-here
LLM_MODEL=qwen/qwen-3.5-4B
ANTHROPIC_API_KEY=...        # improver 모듈용 (선택)
```

**완전 무료 구성 (로컬 LLM):**

```bash
LLM_BASE_URL=http://localhost:11434/v1
LLM_API_KEY=ollama           # 아무 값이나 가능
LLM_MODEL=qwen2.5:14b
```

### ② Evonic 자체 토큰 — 대부분 자동 생성

| 용도 | 설명 |
|---|---|
| 관리자 비밀번호 | `./evonic pass` → `ADMIN_PASSWORD_HASH` |
| Flask SECRET_KEY | 세션 암호화. setup에서 생성 |
| AgentAPI Bearer 토큰 | 외부 API 제공 시 발급 |
| Evonet 자격증명 | 바이너리에 사전 임베드 (페어링 불필요) |
| Cloudflare Turnstile | 로그인 봇 차단 (선택) |

### 보안 체크리스트

- [ ] `.env`는 커밋 금지 (`.gitignore`에 이미 포함됨)
- [ ] `lib/envcrypt/`로 환경변수 암호화 가능
- [ ] `shared/db/session_archive.db`에는 **전체 프롬프트와 대화**가 저장됨 — 민감 데이터로 취급
- [ ] `.env.example` 기본값 `DEBUG=1` → **운영 배포 시 반드시 `0`**
- [ ] 기본값 `HOST=0.0.0.0` → 외부 노출 주의. 로컬 전용이면 `127.0.0.1`

---

## 6. 왜 GitHub에서 주목받나?

**업스트림(anvie/evonic) 실측 지표 (2026-09 기준)**

| 지표 | 수치 |
|---|---|
| ⭐ Stars | 327 |
| 🍴 Forks | 68 |
| 📝 Commits | 2,415 |
| 👀 Watchers | 3 |

LangChain·Dify급 메가히트는 아니지만, **fork/star 비율이 약 20%** 로 높습니다.
별만 누르는 게 아니라 **실제로 가져다 쓰는 비율**이 높다는 신호입니다.

### 주목받는 이유

1. **개인 프로젝트 치고 완성도가 매우 높음** — Python 15.6만 줄, CHANGELOG 130KB
2. **Evonet 터널의 독창성** — 방화벽 없이 원격 실행. 경쟁 프레임워크에 거의 없음
3. **안전성을 전면에 내세움** — 휴리스틱 탐지 + 인젝션 가드 + 사람 승인 플로우
4. **파인튜닝 데이터 수집** — 실제 에이전트 동작을 SFT 데이터셋으로 전환
5. **셀프호스팅 + 데이터 주권** — 로컬 LLM 수요 급증 흐름과 맞음

### 단점 / 리스크

- 사실상 1인 개발 → 버스 팩터 1
- **AGPL-3.0** → 클로즈드 SaaS 불가 (성장 제약 요인)
- 커뮤니티·한국어 자료 부족
- Flask + Jinja 기반이라 UI가 모던 SPA 대비 투박

---

## 7. 로컬 에이전트 구축에 도움이 될까?

**매우 도움이 됩니다. 이 프로젝트 최대 강점입니다.**

| 요소 | 상태 |
|---|---|
| 로컬 LLM 지원 | ✅ Ollama, llama.cpp, LM Studio (OpenAI 호환이면 전부) |
| 클라우드 의존성 | ✅ 0% — 오프라인 동작 가능 |
| 데이터 저장 | ✅ 전부 로컬 SQLite |
| 메모리 엔진 | ✅ Evomem 로컬 바이너리 / FTS5 폴백 |
| 샌드박스 | ✅ 로컬 도커 |
| 웹 UI | ✅ localhost:8080 |

### 완전 오프라인 구성 레시피

```bash
ollama pull qwen2.5:14b

# .env
LLM_BASE_URL=http://localhost:11434/v1
LLM_API_KEY=local
LLM_MODEL=qwen2.5:14b
EVONIC_MEMORY_ENGINE=fts5     # evomem 바이너리 없이도 동작
HOST=127.0.0.1
DEBUG=0

docker build -t evonic-sandbox:latest docker/tools/
./evonic start
```

### 코드 자체가 학습 교재

`backend/agent_runtime/` 는 에이전트를 직접 만들 때 참고할 실전 구현체입니다.

| 파일 | 배울 수 있는 것 |
|---|---|
| `llm_loop.py` | 툴콜 루프 설계와 종료 조건 |
| `llm_tool_executor.py` | 병렬 툴 실행 + 타임아웃 관리 |
| `active_context.py` | 컨텍스트 윈도우 초과 방지 전략 |
| `summarizer.py` | 장문 대화 요약 압축 |
| `memory_manager.py` | 장기 기억 저장/회수 |
| `explorer.py` | 서브에이전트 파일시스템 탐색 |
| `approval.py` | 위험 작업 사람 승인 플로우 |
| `concurrency.py` | 동시성 처리 |
| `cmp/` (RTK) | 토큰 압축 알고리즘 |

### 주의사항

- 7B 이하 소형 모델은 tool calling 정확도가 낮음 → 최소 14B, 권장 32B
- RAM 16GB 최소 / 32GB 권장
- GPU 없으면 응답 지연이 큼

---

## 8. React / PHP로 만들 수 있을까?

### 케이스 A: React로 프론트엔드만 새로 제작 — **강력 추천**

`plugins/agentapi`가 이미 OpenAI 호환 REST를 제공합니다.

```
[React / Next.js 프론트]  ←→  [Evonic AgentAPI]  ←→  [에이전트 런타임]
      직접 제작                   기존 활용             기존 활용
```

```js
const res = await fetch('http://server:8080/v1/chat/completions', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${TOKEN}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    model: 'gpt-4-assistant',   // MODEL_AGENT_MAP 매핑값
    messages: [{ role: 'user', content: '안녕!' }],
    stream: true,
  }),
});
```

### 케이스 B: PHP 프론트/래퍼 — 가능

```php
$res = Http::withToken($token)->post('http://server:8080/v1/chat/completions', [
    'model' => 'gpt-4-assistant',
    'messages' => [['role' => 'user', 'content' => $msg]],
]);
```

워드프레스 플러그인으로 감싸면 기존 PHP 사이트에 AI 상담 기능을 붙일 수 있습니다.

### 케이스 C: Evonic 자체를 React/PHP로 포팅 — **비권장**

| 항목 | 현실 |
|---|---|
| 코드량 | Python 156,000줄 + Go |
| 필요 생태계 | tiktoken, opencv, paramiko, APScheduler, discord.py — PHP 대체제 부실 |
| 예상 기간 | 1인 기준 최소 1~2년 |
| 얻는 이득 | 사실상 없음 |
| 라이선스 | AGPL-3.0 전염 — 포팅해도 소스 공개 의무 |

PHP의 요청-응답 모델은 **장시간 실행 + 상태 유지 + 스트리밍**이 필요한 에이전트
워크로드와 궁합이 나쁩니다 (ReactPHP/Swoole로 가능하지만 난이도 급상승).

### 권장 아키텍처

```
┌─────────────────────────────┐
│  React / Next.js (직접 제작)  │  UI · 결제 · 랜딩
└──────────┬──────────────────┘
           │ REST (OpenAI 호환)
┌──────────▼──────────────────┐
│  Evonic AgentAPI (기존)      │  인증 · 쿼터 · 모델 매핑
└──────────┬──────────────────┘
┌──────────▼──────────────────┐
│  Evonic 런타임 (기존)         │  툴 · 메모리 · 샌드박스
└──────────┬──────────────────┘
┌──────────▼──────────────────┐
│  Ollama / OpenRouter        │
└─────────────────────────────┘
```

---

## 9. 수익화 아이디어

### 먼저: AGPL-3.0 라이선스 주의

AGPL은 **네트워크로 서비스만 제공해도** 수정 소스 공개 의무가 발생합니다.

| 수익 모델 | AGPL 안전성 | 비고 |
|---|---|---|
| 구축 / 컨설팅 | 🟢 안전 | 코드가 아닌 노동 판매 |
| 고객사 온프레미스 설치 | 🟢 안전 | 소스는 해당 고객에게만 제공 |
| 교육 / 강의 / 전자책 | 🟢 안전 | 코드 배포 아님 |
| 오픈소스 SaaS | 🟡 가능 | 수정분 전체 공개 감수 |
| 클로즈드 SaaS | 🔴 불가 | **상업 라이선스 필요** (robin@syihab.st) |

### 티어 1 — 즉시 시작 가능 (초기비용 ≒ 0)

**① AI 에이전트 구축 대행 (최우선 추천)**

```
베이직   150만원 : 설치 + 에이전트 1개 + 텔레그램 연동
스탠다드 400만원 : 에이전트 3개 + KB 구축 + 직원 교육
프리미엄 800만원~: 멀티에이전트 스웜 + 커스텀 툴 개발
+ 월 유지보수 30~80만원
```

`skillsets/` 템플릿 8종 덕분에 셋업 속도가 빠른 것이 경쟁력.
타겟: 쇼핑몰, 병원, 학원, 부동산, 법률사무소.
시작법: 지인 업체 1곳 무료 구축 → 사례집 확보 → 영업.

**② 온프레미스 구축 (데이터 반출 금지 업종)**

- 병원 / 법률 / 회계 / 방산 / 공공기관
- Ollama + Evonic 완전 오프라인 구성
- 단가 1,000만 ~ 5,000만원 (GPU 서버 포함 시), 경쟁자 적고 마진 높음

**③ 교육 콘텐츠**

- 온라인 강의: "로컬 AI 에이전트 완전정복"
- 전자책 / 노션 템플릿 / 유료 커뮤니티
- 유튜브를 상단 깔때기로 활용

### 티어 2 — 개발 필요 (3~6개월)

**④ 업종 특화 SaaS (버티컬)**

예: "미용실 전용 AI 실장" — 예약 확인, 노쇼 방지, 시술 상담, 재방문 유도.
월 99,000원 × 100개 매장 = 월 990만원.
`scheduler`, `channels`, `tools/create_booking.json`이 이미 존재.
⚠️ 클로즈드 SaaS면 상업 라이선스 필요 (또는 매장별 개별 설치로 우회).

**⑤ 스킬 / 플러그인 판매**

- 한국 특화: 네이버 / 카카오 / 쿠팡 / 배민 연동
- 커머스: 스마트스토어 주문·재고 관리
- 세무: 홈택스 연동, 매출 리포트
- 개당 5~20만원 또는 월 구독
- `skills/hello_world/`를 복제해 빠르게 제작 가능

**⑥ React 프리미엄 UI 판매**

- 테마 $49~99, 화이트라벨 $299~
- AgentAPI만 호출하므로 백엔드 수정 없음 → AGPL 회피 가능

### 티어 3 — 하이리스크 하이리턴

**⑦ 파인튜닝 데이터 비즈니스 (가장 강력한 해자)**

```
도메인 에이전트 운영 → build_training_dataset.py로 JSONL 추출
→ 소형 모델(Qwen 7B 등) 파인튜닝
→ "GPT-4의 1/50 비용으로 유사 성능" 전용 모델 확보
→ 모델 판매 또는 초저원가 SaaS 운영
```

데이터는 복제 불가능해 시간이 지날수록 격차가 벌어짐.
⚠️ 고객 대화 학습은 **반드시 사전 동의** 필요.

**⑧ 에이전트 호스팅 서비스 (Vercel for Agents)**

모니터링 · 백업 · 스케일링 · 토큰 비용 대시보드. 월 $29~299.
⚠️ 상업 라이선스 필요.

**⑨ Evonet 기반 원격 IT 관리 AI**

고객사 PC에 Evonet 바이너리 설치 → 방화벽 설정 없이 AI가 원격 진단·정리·리포트.
대당 월 3만원 × 100대 = 월 300만원. 경쟁자가 거의 없는 영역.

### 한국 시장 특화

| 아이디어 | 설명 | 난이도 |
|---|---|---|
| 카카오톡 채널 연동 스킬 | Slack은 로드맵에 있으나 카톡은 부재 | ⭐⭐⭐ |
| 네이버 생태계 스킬 | 스마트스토어 / 플레이스 / 블로그 자동화 | ⭐⭐⭐ |
| 공공기관 납품 | 조달청 등록 + 온프레미스, 고단가 | ⭐⭐⭐⭐ |
| AI 바우처 사업 | 정부지원금 기반 중소기업 AI 도입 대행 | ⭐⭐ |
| 한글(HWP) 문서 처리 스킬 | 국내 수요 확실 | ⭐⭐⭐ |

---

## 10. 실행 로드맵

```
1개월차 — 숙달
  · 로컬 설치, 에이전트 3개 제작, 스킬 1개 직접 개발
  · backend/agent_runtime/ 코드 정독

2개월차 — 레퍼런스 확보
  · 지인 업체 1곳 무료 구축 (후기 + 사례 확보)
  · 과정을 블로그 / 영상으로 기록

3~4개월차 — 유료 전환
  · 사례집 기반 영업 → 유료 고객 2~3곳
  · 한국 특화 스킬 2~3개 제작 및 판매

5~6개월차 — 확장
  · 버티컬 SaaS MVP 또는 온프레미스 고단가 영업 집중
```

### 핵심 원칙

1. 초기에는 **구축 대행**으로 현금흐름과 시장 감각을 먼저 확보
2. SaaS를 목표한다면 **상업 라이선스 문의부터** (robin@syihab.st)
3. "Evonic을 판다"가 아니라 **"고객 문제를 푼다"** 로 접근

---

## 참고 링크

- 이 저장소: <https://github.com/bmshin94/evonic>
- 업스트림 원본: <https://github.com/anvie/evonic>
- 공식 문서: <https://evonic.dev>
- 설치 스크립트: <https://evonic.dev/install.sh>
- 상업 라이선스 문의: robin@syihab.st
