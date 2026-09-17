# OpenHuman 분석 & 활용 가이드 (한국어)

> 이 문서는 OpenHuman 레포지토리를 직접 분석한 내용과, 이를 바탕으로 한
> 학습 · 활용 · 수익화 전략을 정리한 개인 연구 노트입니다.

| 항목 | 내용 |
| --- | --- |
| 작성일 | 2026-09-17 |
| 내 포크 | https://github.com/bmshin94/openhuman |
| 원본(업스트림) | https://github.com/tinyhumansai/openhuman |
| 공식 문서 | https://tinyhumans.gitbook.io/openhuman/ |
| 릴리즈(설치파일) | https://github.com/tinyhumansai/openhuman/releases/latest |
| 제품 페이지 | https://tinyhumans.ai/openhuman |
| 커뮤니티 | https://discord.tinyhumans.ai/ · https://www.reddit.com/r/tinyhumansai/ |
| 라이선스 | GNU GPL-3.0 |
| 상태 | Early Beta (공식 README 명시) |

---

## 1. 한 줄 요약

**OpenHuman = 내 컴퓨터에서 돌아가는 개인용 AI 비서 "본체"를 만드는 오픈소스 데스크탑 앱.**

ChatGPT 같은 채팅창이 아니라, 내 데이터를 기억하고 여러 AI 에이전트를 지휘하는
**사령부(오케스트레이터)** 에 가깝다.

- 비유: ChatGPT = 호텔 룸서비스 / OpenHuman = 우리 집 집사
- 구성: React 프론트엔드 + Rust 코어 + Tauri v2 데스크탑 셸
- 규모: Rust 파일 **3,540개**, TypeScript/TSX 파일 **1,821개** (직접 카운트)

---

## 2. 폴더 구조 (직접 확인한 내용)

```
openhuman/
├── app/                  🖥️  화면 — React + Vite + TypeScript
│   └── src/pages/            Brain / Channels / Flows / Skills / Settings ...
├── crates/               🧠  두뇌 — Rust
│   ├── openhuman-core/       실제 로직 (아래 도메인 참조)
│   ├── openhuman-app/        Tauri 데스크탑 셸
│   ├── openhuman-tui/        터미널 클라이언트
│   ├── openhuman-rpc/        RPC 계약 + 클라이언트
│   ├── openhuman-session/    로그인 / 세션 소유권
│   └── openhuman-embed/      다른 제품에 임베딩하는 라이브러리 파사드
├── vendor/               🔌  부품 — 서브모듈 16개 (별도 오픈소스)
├── gitbooks/             📚  공개 문서 (기능 설명 전부)
├── docs/                 📝  내부 문서 (이 파일 포함)
├── scripts/              🛠️  빌드 · 배포 · 테스트 스크립트 (100개+)
├── packages/             📦  배포 패키징 (Homebrew / deb / AUR / npm)
└── .claude/agents/       🤖  Claude Code 전문 에이전트 13종
```

### 2-1. Rust 코어 도메인 (`crates/openhuman-core/src/`)

`agent` `memory` `tools` `channels` `integrations` `inference` `voice`
`sandbox` `security` `skills` `flows` `cron` `mcp` `media` `medulla`
`threads` `wallet(web3)` `hosted` `hosting` `http_host` `runtime`

> 아키텍처 원칙(AGENTS.md): **비즈니스 로직은 Rust 코어가 소유**하고,
> 프론트엔드와 Tauri 셸은 표현 · 오케스트레이션만 담당한다.
> TypeScript에 코어 정책을 중복 구현하지 않는다.

### 2-2. vendor 서브모듈 16개

| 이름 | 역할 |
| --- | --- |
| `tinyagents` | 체크포인트 기반 에이전트 그래프 |
| `tinyflows` | 워크플로우 엔진 |
| `tinymemory` | 기억 저장소 |
| `tinychannels` | 메신저 채널 |
| `tinyjuice` | 토큰 압축 (TokenJuice) |
| `tinymcp` | MCP 클라이언트 |
| `tinyvoice` | 음성 (STT / TTS) |
| `tinywallet` | 크립토 지갑 |
| 그 외 | `tinybox` `tinybus` `tinyconnectors` `tinydocs` `tinyhosts` `tinyruntime` `tinyhumans-sdk` `motosan-ai-oauth` |

**핵심 포인트: 각 부품이 별도 오픈소스이므로, OpenHuman 전체를 쓰지 않고
필요한 부품만 떼어내 내 프로젝트에 쓸 수 있다.**

---

## 3. 핵심 기능

### 3-1. 🧠 기억하는 두뇌 (Memory Tree + Obsidian Wiki)

- 연결한 계정(Gmail / Notion / Slack / GitHub 등)을 **20분마다 자동 수집(auto-fetch)**
- 내 컴퓨터의 SQLite에 **점수 매긴 마크다운 트리**로 압축 저장
- 동시에 **Obsidian 볼트로 미러링** → 사람이 직접 열어보고 수정 가능
- 벡터DB 블랙박스가 아니라 **눈으로 읽을 수 있는 기억**이 차별점
- Karpathy의 "LLM Knowledgebase" 아이디어에서 영감

### 3-2. 🕸️ 오케스트레이터 (에이전트 지휘)

- 턴이 **루프가 아니라 체크포인트 그래프**로 실행 (`tinyagents`)
- 사람 승인을 기다리며 일시정지 가능, **재시작 후에도 중간부터 재개**
- 서브 에이전트를 3단계 깊이까지 생성, 막힌 에이전트는 **원인 리포트** 반환
- 인스턴스 간 통신은 Signal 프로토콜 E2E 암호화 (+ x402 결제)

### 3-3. 🔬 리서처 & 실행기

- 웹 검색(Exa 기반, 구독 포함) · 스크래퍼 · 실제 브라우저 조작
- 음성(STT는 프록시 / TTS는 Piper 로컬 가능)
- 이미지 · 영상 생성 (Seedream / SeedEdit / Seedance / Veo)
- **모델 라우팅**: 작업 성격별로 적절한 LLM 자동 선택

### 3-4. 그 외

- **Workflows**: AI가 자동화를 먼저 제안 → 비주얼 캔버스에서 검토 후 저장.
  스케줄 · 웹훅 · 채널 이벤트로 발동하며 재시작에도 살아남고 승인 게이트 통과.
- **메신저 채널 16종**: Telegram, Discord, Slack, WhatsApp, Signal, iMessage,
  IRC, Mattermost, DingTalk, QQ, Lark, Yuanbao, Linq, 네이티브 이메일(IMAP/SMTP)
- **TokenJuice**: 툴 출력을 압축해 토큰 최대 80% 절감
- **Privacy Mode**: 스위치 하나로 추론이 기기 밖으로 못 나가게 **Rust 코어에서 강제**
- **OS 키링 시크릿 저장**, 옵트인 샌드박싱, 승인 게이트
- **마스코트**: 말하고 반응하는 캐릭터 UI
- **Theme Studio**: 테마 5종 + 비주얼 에디터 (JSON 내보내기)

---

## 4. 핵심 질문 7개 정리

### Q1. 설치 및 사용법?

#### A. 그냥 쓰기 (권장)

| OS | 명령 / 방법 |
| --- | --- |
| macOS | `brew install --cask openhuman` |
| Windows | Releases에서 서명된 `.msi` 다운로드 후 실행 |
| Debian/Ubuntu | `sudo apt-get install -y --no-install-recommends ./OpenHuman_*_amd64.deb` |
| Arch | `yay -S openhuman-bin` |

> ⚠️ `curl ... | bash` / `irm ... | iex` 설치 스크립트는 **INSTALL.md에
> "서명 검증 없음(unverified)"이라고 명시**되어 있음. 네이티브 패키지 우선.
> 리눅스 AppImage는 Wayland 등에서 실패 사례가 있어 `.deb` 권장.

**사용 흐름**: 설치 → 로그인 → 계정 연결 → 20분 주기 자동 수집 →
Brain 화면에서 기억 확인 → 채팅으로 작업 지시 → Workflows로 자동화

#### B. 소스에서 빌드

선행 조건: Node.js 24+, pnpm 10.10.0, Rust 1.96.1 (rustfmt + clippy),
CMake, Ninja, ripgrep, 플랫폼별 데스크탑 빌드 의존성

```bash
git clone https://github.com/bmshin94/openhuman
cd openhuman
git submodule update --init --recursive   # ⚠️ 필수! 빠뜨리면 빌드 실패
pnpm install

pnpm dev            # 웹 UI만 (가장 가벼움, 시작 추천)
pnpm dev:app        # macOS 데스크탑
pnpm dev:app:win    # Windows 데스크탑

# 검증 명령
pnpm typecheck
pnpm lint
pnpm format:check
pnpm test
cargo check -p openhuman --lib
```

#### C. 서버(헤드리스) 배포

Rust 코어만 **포트 7788**에서 JSON-RPC 서버로 단독 실행 가능.
문서화된 4가지 경로: DigitalOcean 원클릭 / doctl 수동 / VPS + Docker Compose / Fly.io

```bash
openhuman-core serve
# 데스크탑 앱에서 원격 코어에 접속:
#   OPENHUMAN_CORE_RPC_URL=https://core.example.com/rpc
#   OPENHUMAN_CORE_TOKEN=...
```

용도: 멀티 디바이스 공용 코어, 로컬 툴체인 없는 테스터, **노트북을 꺼도
살아있어야 하는 장기 실행 크론 / 웹훅**

---

### Q2. 플러그인? 스킬? MCP? → **셋 다 아님**

OpenHuman은 플러그인이 아니라 **플러그인을 꽂는 본체(호스트/하네스)** 다.

```
        ┌─────────────────────────┐
        │   OpenHuman (본체/앱)    │
        └───────────┬─────────────┘
                    │ 꽂아서 사용
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
  MCP 서버      Skills 카탈로그    OAuth 커넥터
  (수천 개)      (약 9만 개)       (100+ 개)
```

- **MCP**: Smithery.ai + 공식 `registry.modelcontextprotocol.io` 두 레지스트리를
  병렬 검색·병합. 로컬 설치 시 stdio 서브프로세스로 실행, 배포형은 HTTP 연결.
  설치 목록은 `mcp_clients/mcp_clients.db`에 저장되고, 슈퍼바이저가 **약 60초마다
  프로브 + 서버별 지수 백오프로 재접속**. 비정상 결과는 Settings → Developer →
  Event Log의 `mcp` 배지에 기록되고 계속 죽어있으면 알림 발생.
- **Skills**: HermesHub / ClawHub / LobeHub 등을 합친 약 **9만 개 메타데이터 카탈로그**.
  부팅 시 백그라운드 수집, `~/.openhuman/skill-registry/cache.json`에 약 1시간 TTL 캐시.
  ⚠️ **중요: 인앱 스킬 런타임(QuickJS 샌드박스)은 제거됨.** 현재 Skills는
  코드 실행이 아니라 **둘러보고 설치하는 메타데이터 카탈로그**.
- **플러그인**: 그런 개념 자체가 없음.

#### 🔑 반대 방향도 가능 — 실용성 최고

```bash
openhuman-core mcp
```

OpenHuman이 **스스로 MCP 서버가 되어** Claude Desktop / Claude Code 등에
**읽기 전용 툴**(메모리 검색 · recall, Memory Tree 탐색, 선택적 웹검색)을 제공.

> **OpenHuman 전체를 쓰지 않고, "기억 저장소"로만 쓰면서 평소 작업은
> Claude Code로 하는 하이브리드 활용이 가능하다.** 가장 빨리 체감되는 활용법.

---

### Q3. API 토큰이 필요한가? → **선택 사항. 3가지 경로를 섞어 쓸 수 있음**

|  | 🅰️ 관리형 구독(기본) | 🅱️ BYOK(내 키) | 🅲 완전 로컬 |
| --- | --- | --- | --- |
| API 키 | 불필요 | 제공사별 1개 | 불필요 |
| 채팅 / 추론 | 포함 | 내 키 · 내 과금 | Ollama / LM Studio |
| 비전(이미지 인식) | 포함 | 모델 지원 시 | 비전 가능 모델 필요 |
| 임베딩 | 포함 | 제공사에 따라 | `bge-m3` 권장 |
| STT(음성인식) | 포함 | 내 키 | **로컬 엔진 없음** |
| TTS(음성합성) | 포함 | 내 키 | Piper 가능 |
| 웹 검색 | 포함(Exa, 키 불필요) | Exa / Tavily / Brave / Parallel 키 | 해당 없음 |
| 추론 데이터 외부 전송 | 있음 | 내 제공사로 | **없음** |

**혼합 가능**: 임베딩은 로컬 + 채팅은 내 Anthropic 키 + 비전은 관리형 — 동시 설정 OK.

`.env.example` 확인 결과: 키 35개 중 **API 키 항목은 전부 주석 처리된 선택 사항**.
필수 입력값은 없고 기본값은 동작 설정(`OPENHUMAN_MODEL`, `OPENHUMAN_TEMPERATURE`),
로컬 바이너리 경로(`OLLAMA_BIN`, `PIPER_BIN`), 내부 통신용(`JWT_TOKEN`,
`OPENHUMAN_CORE_PORT`) 위주.

구독 요금제(billing 문서): **Free $0 / Basic $19.99·월 / Pro $199.99·월**
(상위 요금제는 호출당 단가 50~90% 할인)

> ⚠️ 문서가 명시한 주의사항: 로컬 모델을 쓰더라도 **로그인 · 관리형 OAuth ·
> 결제 · 미팅 기능은 여전히 백엔드를 사용**한다. 추론이 절대 나가지 않는
> 하드 보장을 원하면 **Privacy Mode**를 켜야 한다(코어에서 강제).

---

### Q4. 왜 GitHub에서 유명할까?

1. **타이밍** — "로컬 AI 에이전트" 열풍기에, 터미널이 아닌 **GUI 데스크탑 앱**으로
   진입장벽을 크게 낮춰 출시.
2. **Karpathy 후광** — README가 Karpathy의 "LLM Knowledgebase" 트윗을 반복 인용하며
   "그 아이디어의 구현체"로 포지셔닝.
3. **README 마케팅** — Trendshift · Product Hunt 배지 다수, 경쟁사 비교표
   (Claude Cowork / OpenClaw / Hermes 대비), 스타 히스토리 그래프,
   컨트리뷰터 명예의 전당, **6개 언어 번역(한국어 포함)**.
4. **기능 총집합** — 기억 + 오케스트레이션 + 워크플로우 + 채널 16종 + 음성 +
   브라우저 + 미디어 생성 + 지갑. "이거 하나면 다 된다"는 인식.
5. **기여 문턱 최소화** — 초보자 가이드에 **AI에게 복붙할 프롬프트**까지 제공,
   기여자에게 굿즈 + Discord 특별 권한, **CLA 서명 요구 없음**.
6. **감성** — 마스코트 캐릭터, "인간을 위한 하네스" 슬로건,
   "AGI는 아니지만 의미 있는 한 걸음" 같은 겸손한 톤.

**출시 1주 내 GitHub 트렌딩 1위 9일 연속** (README 자체 주장)

#### 냉정한 평가

- 스타 수는 품질의 증명이 아니라 **마케팅 실력도 크게 반영**된다.
- 프로젝트 스스로 **"Early Beta, 거친 부분 있음"**이라고 명시.
- 경쟁사 비교표는 자체 제작이므로 당연히 유리하게 구성되어 있다
  (문서도 "각 벤더에서 직접 확인하라"고 덧붙임).

---

### Q5. 로컬 에이전트 구축에 도움이 될까? → **매우 많이 된다**

직접 에이전트를 만들 때 부딪히는 문제와, 이 레포가 가진 해답:

| 겪게 되는 문제 | OpenHuman의 해법 | 볼 곳 |
| --- | --- | --- |
| 작업 중 끊기면 처음부터? | 체크포인트 그래프 | `vendor/tinyagents` |
| 툴 정의 수작업이 귀찮음 | 스키마 자동 생성 | `crates/openhuman-core/src/tools/generated.rs`, `tools/schemas/` |
| AI가 위험한 동작을 하면? | 승인 게이트 | `gitbooks/features/approval-gate.md` |
| 툴 출력이 길어 토큰 폭발 | 압축(최대 80% 절감) | `vendor/tinyjuice` |
| 기억을 어떻게 저장? | 마크다운 트리 + SQLite | `crates/openhuman-core/src/memory/` |
| MCP 서버가 자꾸 죽음 | 60초 프로브 + 지수 백오프 | `crates/openhuman-core/src/mcp/` |
| API 키를 어디 저장? | OS 키링 | `gitbooks/features/os-keyring-and-secret-storage.md` |
| 비용 추적 / 재현 | 리플레이 가능한 실행 저널 + 호출별 원가 | `gitbooks/developing/agent-observability.md` |

추가로 배울 것: 레이어 경계 설계(`AGENTS.md`), 파일마다 `*_tests.rs`를 짝지어 두는
테스트 문화, e2e 커버리지 매트릭스 관리 방식.

#### 권장 학습 단계

1. `pnpm dev`로 웹 UI 실행 (가장 안전한 첫걸음)
2. `openhuman-core mcp`로 Claude Code에 기억 붙이기 ← **즉시 실용적**
3. `crates/openhuman-core/src/memory/` 읽으며 기억 구조 학습
4. `vendor/tinyagents` 하나만 떼어 내 프로젝트에 적용
5. 내 에이전트 직접 설계

---

### Q6. React / PHP로 만들 수 있을까?

#### React → **이미 React다**

`app/src/` 전체가 React + Vite + TypeScript (파일 1,821개).
실제 화면: `Brain.tsx` `Channels.tsx` `FlowsPage.tsx` `Skills.tsx`
`Settings.tsx` `Welcome.tsx` 등. **프론트엔드는 지금 당장 수정 가능.**

#### PHP → 가능하지만 "본체"는 부적합

| 필요 요소 | PHP의 현실 |
| --- | --- |
| 수 시간 상주하는 프로세스 | 요청-응답 모델이라 부자연스러움 |
| 실시간 토큰 스트리밍 | 가능하지만 번거로움 |
| 에이전트 수십 개 병렬 | 동시성 취약 |
| 재시작 후 이어서 실행 | DB에 상태 저장하면 가능 |
| 데스크탑 앱 배포 | 불가능 |

**Laravel 우회 설계** — 에이전트 루프를 "큐 작업 단위"로 쪼개는 것이 핵심:

```
React (프론트)
   ↕ WebSocket
Laravel + Reverb (실시간)
   ↕
Laravel Queue + Horizon (에이전트 단계를 작업으로 분할) ← 핵심
   ↕
MySQL (체크포인트 = 단계별 상태 저장)
   ↕
MCP 서버들 (별도 프로세스로 기동)
```

각 단계 완료 시 DB에 저장하고 다음 작업을 큐에 넣으면, 서버가 죽어도 이어서 실행된다.

#### 가장 현실적인 결론

전체 복제는 비추천 (Rust 3,540 + TS 1,821 파일 규모).

> **`openhuman-core`는 이미 JSON-RPC 서버(포트 7788)다.
> PHP로 다시 만들 필요 없이 PHP에서 그 API를 호출하면 된다.**
> 무거운 처리는 Rust가, UI와 비즈니스 로직은 React/PHP가 담당.
> 프로세스가 분리되므로 **GPL 전염도 발생하지 않는다.**

권장 MVP 경로: 기능 1개만 선정 → React + Laravel로 2~4주 MVP →
무거운 부분은 `openhuman-core` API 호출 또는 `vendor/tinymemory` 활용 → 확장

---

## 5. 레포에서 발견한 "빈칸" = 사업 기회

### 발견 1. 메신저 채널 16종 중 한국 서비스 0개

`crates/openhuman-core/src/channels/providers/` 실제 파일 목록:

```
dingtalk  discord  email_channel  imessage  irc  lark  linq
mattermost  qq  signal  slack  telegram  whatsapp  whatsapp_web  yuanbao
```

**중국 서비스는 4개(DingTalk · QQ · Lark · Yuanbao)나 있는데,
카카오톡 · 네이버웍스 · 라인은 전무하다.**

### 발견 2. 커넥터 100개+ 중에도 한국 서비스 0개

| 카테고리 | 지원 | 한국 서비스 |
| --- | --- | --- |
| 메일 / 캘린더 | Gmail, Outlook, Google Calendar, Apple Calendar | 없음 |
| 문서 / 저장소 | Google Docs, Drive, Notion, Dropbox, Airtable | 없음 |
| 개발 | GitHub, Linear, Jira, Figma | 해당 없음 |
| 커뮤니케이션 | Slack, Discord, Teams, Telegram, WhatsApp | 없음 |
| CRM / 세일즈 | Salesforce, HubSpot | 없음 (더존 · 이카운트) |
| 커머스 / 결제 | Stripe, Shopify | 없음 (토스 · 카페24 · 쿠팡윙 · 스마트스토어) |
| 프로젝트 관리 | Asana, Trello | 없음 (잔디 · 플로우) |
| 소셜 | Twitter/X, Spotify, YouTube | 없음 |

### 발견 3. 스킬 레지스트리를 내 서버로 바꿀 수 있다 (`.env.example` 332행)

```bash
SKILLS_REGISTRY_URL=https://example.com/registry.json   # 원격
SKILLS_REGISTRY_URL=/path/to/registry.json              # 로컬 (캐시 없이 즉시 반영)
SKILLS_LOCAL_DIR=/path/to/skills                        # 개발용 최우선 소스
```

→ **자체 스킬 카탈로그(유료 장터)를 직접 운영할 수 있다.**

### 발견 4. 리퍼럴 프로그램이 이미 내장

- 계정별 추천 코드 1개, 공유 시트 / 클립보드 공유 지원
- 대시보드 타일: 내 코드 · **총 적립액(USD)** · 대기 중 · 완료
- 상태: Joined(가입) → Completed(전환, 보상 지급) → Expired
- Discord 연동 시 사용량 마일스톤에 따라 커뮤니티 역할 해금
- ⚠️ 백엔드 로그인 세션 필수 (로컬 전용 세션에서는 동작하지 않음)

### 발견 5. 지갑 + 에이전트 간 결제(x402)

- 비수탁형 멀티체인 지갑 (EVM 6체인 + Bitcoin), 복구 구문에서 체인별 1계정 파생
- **서명 · 브로드캐스트가 전부 코어 내부에서 수행되어 개인키가 기기를 떠나지 않음**
- `prepare → confirm → execute` 3단 안전 흐름, 네이티브 전송 + 일부 토큰만 지원
- 스왑 · 브리지 · 임의 컨트랙트 호출은 별도 `web3` 모듈이며 에이전트 노출 범위 밖

---

## 6. 라이선스 브리핑 (GPL-3.0, AGPL 아님)

### 안전지대

| 하는 일 | 근거 |
| --- | --- |
| 서버에 올려 **SaaS로 판매** | GPL은 "배포" 시 소스 공개. SaaS는 배포가 아님 |
| **MCP 서버**를 따로 만들어 판매 | 완전 별개 프로그램 |
| **JSON-RPC로 호출**하는 앱 제작 | 프로세스 분리 |
| **설치 · 구축 대행** | 코드 수정 없음 |
| **강의 · 컨설팅 · 콘텐츠** | 무관 |
| 소프트웨어 자체를 **유료 판매** | GPL도 판매 자유 |

### 위험지대

| 하는 일 | 리스크 |
| --- | --- |
| 코드 수정 후 **설치파일 배포** | 수정 소스 전체 공개 의무 |
| **상용 앱에 코드 직접 포함** | 그 앱까지 GPL로 전염 |
| "OpenHuman" **이름 / 로고 사용** | 상표권은 라이선스와 별개 문제 |

### 황금 규칙

> ## **본체는 수정하지 않고, 옆에 붙여서 판다.**
> 별도 프로세스(MCP) · 별도 서버(호스팅) · 별도 지식(강의)
> = GPL 리스크 없음 + 내 지식재산 100% 보유

- CONTRIBUTING.md 확인 결과 **CLA(저작권 양도) 요구 없음** → 기여해도 권리 유지
- 실제 매출이 발생하면 **변호사 1회 검토 권장** (특히 상표 사용)

---

## 7. 수익화 아이디어 7개

### TIER 1 — 즉시 시작 가능 (자본 0원)

#### 아이디어 1. 한국어 콘텐츠 선점

| 항목 | 내용 |
| --- | --- |
| 근거 | 한국어 README는 이미 있으나(`docs/README.ko.md`) 한국 커뮤니티 · 자료는 없음 |
| 할 일 | 설치 후기 → 기능별 튜토리얼 → "로컬 AI 에이전트 만들기" 시리즈 |
| 채널 | 블로그(SEO) + 유튜브 + 인프런 / 클래스101 |
| 자본 | 0원 / 콘텐츠 1편당 2~3일 |
| 수익 시나리오 | 강의 5만원 × 200명 = 약 1,000만원 (1회 제작 후 반복 판매) |
| 부가효과 | **신뢰 자산** — 아이디어 3 · 4 · 5의 연료가 된다 |

킬러 콘텐츠 후보:
- "Claude Code에 **기억**을 붙이는 방법" (`openhuman-core mcp` 활용)
- "GitHub 1위 프로젝트 5,300개 파일 해부기"
- "데이터 유출 0%, 내 컴퓨터에서만 도는 AI 비서 만들기"

#### 아이디어 2. 리퍼럴 (병행용)

| 항목 | 내용 |
| --- | --- |
| 구조 | 추천 코드 공유 → 전환 시 USD 크레딧 적립 (Basic $19.99 / Pro $199.99) |
| 현실 | 단독 수익원으로는 부족. 아이디어 1의 부수입으로 취급 |
| 주의 | 추천 링크임을 반드시 밝힐 것 (신뢰가 최우선 자산) |

### TIER 2 — 1~3개월 (개발 필요, 수익성 최고)

#### 🏆 아이디어 3. 한국 특화 MCP 서버 판매 — **최우선 추천**

선정 이유 4가지:

1. **GPL 전염 없음** — 별도 프로세스, 내 코드는 내 소유
2. **시장 3배** — OpenHuman뿐 아니라 Claude · Cursor · Codex 사용자도 고객
3. **경쟁자 없음** — 위 "발견 1 · 2"에서 확인한 빈칸
4. **재고 · 배송 없음** — 한 번 만들면 무한 판매

| 우선순위 | 대상 | 시장 | 난이도 | 비고 |
| --- | --- | --- | --- | --- |
| 1 | 네이버 스마트스토어 / 쿠팡윙 | 셀러 수십만 | 중 | 주문 · 재고 · 문의 자동화 = ROI가 즉시 보임 |
| 2 | 카카오톡 채널(비즈메시지) | 거의 전 국민 | 중 | 채널 목록 최대 빈칸 |
| 3 | 국세청 홈택스 / 더존 | 세무사 · 사업자 | 상 | B2B 단가 최고 |
| 4 | 토스페이먼츠 / 카페24 | 쇼핑몰 | 중 | 정산 · 매출 자동 집계 |
| 5 | 잔디 / 플로우 / 네이버웍스 | 국내 기업 | 하~중 | 기업 영업으로 연결 쉬움 |

수익 모델: (A) 오픈소스 무료 + 유료 서포트 / (B) **월 구독 1.9~4.9만원(추천)** /
(C) 기업 라이선스 연 300~1,000만원

시나리오: 월 2.9만원 × 100명 = **월 약 290만원**

> 시작은 **네이버 스마트스토어 MCP 하나만.** 동시 다발 착수 금지.

#### 아이디어 4. 유료 스킬팩 / 워크플로우 템플릿 장터

| 항목 | 내용 |
| --- | --- |
| 근거 | `SKILLS_REGISTRY_URL`로 자체 레지스트리 운영 가능 (발견 3) |
| 파는 것 | 직군별 완성 패키지 (기억 구조 + 워크플로우 + 프롬프트 세트) |
| 상품 예시 | 세무사팩 / 쇼핑몰 CS팩 / 병원 예약팩 / 개발팀 PR 리뷰팩 / 부동산팩 |
| 가격 | 팩당 5~30만원 또는 월 구독 |
| 강점 | 코딩보다 **도메인 지식**이 핵심이라 모방이 어렵다 |
| 리스크 | ⚠️ 스킬 런타임(QuickJS)이 제거되어 **코드 실행 불가**. "설정 + 프롬프트 묶음"으로 설계해야 함 |

### TIER 3 — 3~6개월 (단가 최고, B2B)

#### 아이디어 5. 기업 온프렘 구축 대행 — 단가 1위

세일즈 포인트:

> "Privacy Mode를 켜면 데이터가 회사 밖으로 나가지 않습니다.
> 그리고 소스가 전부 공개되어 있어 저희 말을 믿지 않고 직접 검증하실 수 있습니다."

| 항목 | 내용 |
| --- | --- |
| 타겟 | 병원 · 법무법인 · 회계법인 · 금융 · 방산 · 제조 |
| 기술 근거 | Privacy Mode(코어 강제) + OS 키링 + 승인 게이트 + 샌드박싱 |
| 구축 경로 | 사내 서버에 `openhuman-core serve` + Ollama 로컬 모델 |
| 견적 예시 | 구축 500~2,000만원 + 유지보수 월 100~300만원 |
| 필요한 것 | 레퍼런스 1건 (첫 고객은 할인해서라도 확보) |
| 리스크 | ⚠️ Early Beta를 "완제품"으로 판매하면 위험. "검증 · 커스터마이징한 구성"으로 포지셔닝하고 SLA 범위를 계약서에 명시 |

GPL의 반전: 소스 공개는 약점이 아니라 **"감사 가능"이라는 무기**가 된다.

#### 아이디어 6. 매니지드 호스팅

| 항목 | 내용 |
| --- | --- |
| 근거 | `cloud-deploy.md`의 4가지 배포 경로 + 원격 코어 접속 지원 |
| 파는 것 | "노트북을 꺼도 24시간 도는 내 AI 비서" (크론 · 웹훅 상시 실행) |
| 가격 | 월 2~3만원 |
| 시나리오 | 3만원 × 100명 = 월 약 300만원 (서버비 제외 마진 60~70%) |
| GPL | SaaS는 소스 공개 의무 없음 |
| 리스크 | 🚨 **개인정보 이슈가 최대 난관.** 타인의 Gmail · Notion 데이터를 내 서버에 저장하는 구조이므로 개인정보보호법 검토 · 암호화 · 약관이 선행되어야 함. **준비 없이 착수 금지** |

### TIER 4 — 장기 (실험적)

#### 아이디어 7. 에이전트 결제 인프라 (x402)

근거: 발견 5의 지갑 구현 + README의 x402 / Signal E2E 언급.
"AI가 AI에게 일을 맡기고 자동 결제"라는 미개척 시장.
**단, 현재 시장이 형성되어 있지 않아 지금 착수는 비추천.** 2~3년 후 재검토.

### 비교표

| # | 아이디어 | 자본 | 난이도 | 수익 규모 | 속도 | GPL 안전 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 한국어 콘텐츠 | 0 | ★ | 중 | 빠름 | 안전 |
| 2 | 리퍼럴 | 0 | ★ | 소 | 빠름 | 안전 |
| 3 | **MCP 서버 판매** | 낮음 | ★★★ | 대 | 보통 | 안전 |
| 4 | 스킬팩 장터 | 낮음 | ★★ | 중 | 보통 | 안전 |
| 5 | **기업 온프렘** | 낮음 | ★★★★ | 최대 | 느림 | 안전 |
| 6 | 매니지드 호스팅 | 중간 | ★★★★ | 대 | 느림 | 안전(단 개인정보 이슈) |
| 7 | x402 결제 | 높음 | ★★★★★ | 미지 | 매우 느림 | 안전 |

---

## 8. 3개월 로드맵

```
1개월 — 신뢰 쌓기 (매출 0, 가장 중요한 단계)
  [ ] 직접 설치해 2주간 실사용 (안 써보고는 팔 수 없다)
  [ ] 블로그 / 유튜브 5편 발행
      첫 편 추천: "Claude Code에 기억 붙이기"
  [ ] 오픈소스 기여 1건 (한국어 문서 개선이 가장 쉬움)
  [ ] 카톡 오픈채팅 / Discord 커뮤니티 개설
        ↓ 사용자가 무엇을 원하는지 파악됨

2개월 — 제품 만들기
  [ ] MCP 서버 1개 개발 (네이버 스마트스토어 권장)
  [ ] 무료 베타 20명 모집 후 피드백 반영
  [ ] 스킬팩 1개 제작 (내가 가장 잘 아는 도메인)
        ↓ 첫 매출 발생

3개월 — 매출 만들기
  [ ] MCP 서버 유료 전환 (월 2.9만원)
  [ ] 강의 출시 (인프런 / 클래스101)
  [ ] 기업 상담 시작 (1~2개월 콘텐츠가 신뢰 근거가 된다)
```

**목표: 월 100~300만원 + 기업 파이프라인 2~3건**

---

## 9. 피해야 할 함정 5가지

| 함정 | 이유 |
| --- | --- |
| 본체를 수정해 재배포 | GPL 위반 리스크. "옆에 붙이기" 원칙 유지 |
| "OpenHuman 한국판" 같은 이름 사용 | 상표 문제. 자체 브랜드 필요 |
| 여러 아이디어 동시 착수 | 집중력 분산으로 전부 실패. 하나만 끝까지 |
| Early Beta를 완제품으로 판매 | 환불 · 평판 리스크. 한계를 먼저 밝히면 오히려 신뢰 획득 |
| 타인의 개인정보를 준비 없이 내 서버에 저장 | 개인정보보호법 위반. 법률 검토 없이 착수 금지 |

---

## 10. 최종 권고 — 지금 할 3가지

1. **2주간 직접 사용해보기** (설치: `brew install --cask openhuman` 또는 릴리즈 파일)
2. **"Claude Code에 기억 붙이기" 글 1편 작성** (`openhuman-core mcp` 활용, 자본 0원)
3. **네이버 스마트스토어 MCP 서버 개발** (GPL 안전 · 경쟁자 없음 · 시장 3배)

---

## 참고 링크

| 구분 | 주소 |
| --- | --- |
| 내 포크 | https://github.com/bmshin94/openhuman |
| 원본 레포 | https://github.com/tinyhumansai/openhuman |
| 릴리즈 | https://github.com/tinyhumansai/openhuman/releases/latest |
| 공식 문서 | https://tinyhumans.gitbook.io/openhuman/ |
| Discussions | https://github.com/tinyhumansai/openhuman/discussions |
| Discord | https://discord.tinyhumans.ai/ |
| Reddit | https://www.reddit.com/r/tinyhumansai/ |
| 제품 페이지 | https://tinyhumans.ai/openhuman |
| MCP 공식 레지스트리 | https://registry.modelcontextprotocol.io |
| Smithery (MCP) | https://smithery.ai |
| Ollama | https://ollama.com |

### 레포 내부 문서

| 문서 | 경로 |
| --- | --- |
| 기여 규칙 / 아키텍처 규약 | `AGENTS.md` (`CLAUDE.md`가 심볼릭 링크) |
| 기여 가이드 | `CONTRIBUTING.md`, `docs/CONTRIBUTING-BEGINNERS.md` |
| 설치 상세 | `INSTALL.md` |
| 한국어 README | `docs/README.ko.md` |
| 아키텍처 | `gitbooks/developing/architecture.md` |
| 에이전트 하네스 | `gitbooks/developing/architecture/agent-harness.md` |
| MCP & Skills | `gitbooks/features/integrations/mcp-and-skills.md` |
| 로컬 / BYOK 모델 | `gitbooks/features/model-routing/local-and-byok-models.md` |
| 클라우드 배포 | `gitbooks/features/cloud-deploy.md` |
| 과금 & 사용량 | `gitbooks/features/billing-and-usage.md` |
| 리퍼럴 & 보상 | `gitbooks/features/rewards-and-referrals.md` |
| 지갑 | `gitbooks/features/wallet.md` |
| 프라이버시 모드 | `gitbooks/features/privacy-mode.md` |

---

> ⚠️ 면책: 이 문서의 수익 금액은 **가정에 기반한 시나리오**이며 보장된 수치가 아닙니다.
> 라이선스 해석은 참고용이므로 실제 사업화 전에 법률 전문가의 검토를 받으시기 바랍니다.
> OpenHuman은 Early Beta 단계이므로 기능과 문서는 변경될 수 있습니다.
