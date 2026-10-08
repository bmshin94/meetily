# 📊 Meetily 전수조사 분석 보고서 (한국어)

> 작성일: 2026-10-08
> 분석 대상 버전: **v0.4.1** (커밋 `be1c7de`)
> 분석 브랜치: `claude/friendly-tesla-g389l3`

---

## 🔗 GitHub 저장소 주소

| 구분 | 주소 |
|---|---|
| 🍴 **내 포크 (작업 저장소)** | https://github.com/bmshin94/meetily |
| 🌱 **원본 저장소 (upstream)** | https://github.com/Zackriya-Solutions/meetily |
| 📦 **릴리즈 (설치파일 다운로드)** | https://github.com/Zackriya-Solutions/meeting-minutes/releases/latest |
| 🌐 공식 웹사이트 | https://meetily.ai |
| 💎 PRO 버전 | https://meetily.ai/pro/ |
| 🏢 Enterprise | https://meetily.ai/enterprise/ |
| 💬 Discord | https://discord.gg/crRymMQBFH |
| 👥 Reddit 커뮤니티 | https://www.reddit.com/r/meetily/ |
| 🎬 데모 영상 | https://youtu.be/6FnhSC_eSz8 |

### 참고한 외부 프로젝트
- Whisper.cpp — https://github.com/ggerganov/whisper.cpp
- Screenpipe — https://github.com/mediar-ai/screenpipe
- transcribe-rs — https://crates.io/crates/transcribe-rs
- Parakeet ONNX — https://huggingface.co/istupakov/parakeet-tdt-0.6b-v3-onnx

---

## 1️⃣ 이게 뭐하는 프로젝트인가?

### 한 줄 요약
> **내 컴퓨터 안에서만 돌아가는 AI 회의 비서.**
> 회의 녹음 → 실시간 자동 받아쓰기(STT) → AI 회의록 자동 작성까지, 인터넷 없이 전부 로컬 처리.

| 항목 | 내용 |
|---|---|
| 정식 이름 | Meetily (Privacy-First AI Meeting Assistant) |
| 현재 버전 | v0.4.1 |
| 라이선스 | **MIT** (상업적 이용 / 수정 / 재배포 모두 허용) |
| 지원 OS | macOS (Apple Silicon), Windows x64, Linux (소스 빌드) |
| 코드 규모 | Rust 약 **47,000줄** + Next.js/React 프론트엔드 |
| 특징 | TrendShift 등재 프로젝트 (유명 오픈소스) |

### 핵심 구조 — 서버가 필요 없는 단일 데스크톱 앱

```
┌──────────────────────────────────────────────────────┐
│          Meetily.app  (실행파일 하나!)                │
│                                                       │
│  ┌─────────────────┐      ┌─────────────────────┐   │
│  │   화면 (UI)     │ ←──→ │  두뇌 (Rust Core)    │   │
│  │  Next.js 14     │Tauri │  - 오디오 캡처        │   │
│  │  React 18 + TS  │ IPC  │  - Whisper 받아쓰기   │   │
│  │  Tailwind       │      │  - SQLite 저장        │   │
│  └─────────────────┘      │  - LLM 요약 호출      │   │
│                           └─────────────────────┘   │
└──────────────────────────────────────────────────────┘
        ↑ 전부 로컬! 클라우드 전송 0% (사용통계 옵션 제외)
```

---

## 2️⃣ 폴더별 전수조사 결과

### 🥇 `/frontend` — **앱 본체 전부** (이름은 frontend지만 실제론 앱 전체)

#### `frontend/src/` — 화면 (React / Next.js)
```
src/
├── app/
│   ├── page.tsx                  ← 메인 녹음 화면
│   ├── settings/page.tsx         ← 설정 (API키, 모델 선택)
│   ├── meeting-details/          ← 회의 상세 (전사본 + 요약)
│   ├── notes/[id]/               ← 노트 편집 페이지
│   └── _components/
│       ├── SettingsModal.tsx
│       ├── TranscriptPanel.tsx   ← 실시간 자막 패널
│       └── StatusOverlays.tsx
├── components/
│   ├── Sidebar/SidebarProvider.tsx  ← 전역 상태관리 (Context API)
│   ├── BlockNoteEditor/             ← Notion 스타일 블록 에디터
│   ├── AISummary/                   ← AI 요약 뷰
│   ├── ImportAudio/                 ← 기존 음성파일 가져오기
│   ├── TranscriptRecovery/          ← 앱 크래시 시 복구 기능
│   ├── DatabaseImport/              ← 구버전 DB 마이그레이션
│   ├── onboarding/steps/            ← 첫 실행 온보딩 마법사
│   └── ui/                          ← shadcn/ui + Radix 컴포넌트
├── hooks/        ← 커스텀 훅
├── services/     ← Tauri invoke 래퍼
└── contexts/
```

#### `frontend/src-tauri/src/` — 🧠 두뇌 (Rust)
```
src-tauri/src/
├── lib.rs                   (868줄)  ← ⭐ 심장. Tauri 커맨드 200개+ 등록
├── lib_old_complex.rs      (2437줄)  ← 리팩토링 전 구버전 (미사용)
│
├── audio/                             ← 🎙️ 오디오 엔진 (가장 복잡)
│   ├── pipeline.rs         (1116줄)  ← 믹싱 + VAD. 핵심 중의 핵심
│   ├── recording_commands.rs(1697줄) ← 녹음 시작/정지/일시정지
│   ├── import.rs           (1323줄)  ← 외부 오디오 파일 import
│   ├── retranscription.rs  (1054줄)  ← 다른 모델로 재전사
│   ├── decoder.rs           (869줄)  ← mp3/m4a/flac 디코딩 (Symphonia)
│   ├── vad.rs               (779줄)  ← 음성 구간 감지 (Silero VAD)
│   ├── audio_processing.rs  (739줄)  ← 노이즈 제거, 라우드니스 정규화
│   ├── incremental_saver.rs (475줄)  ← 체크포인트 저장 (크래시 복구)
│   ├── ffmpeg_mixer.rs      (513줄)  ← FFmpeg 사이드카 믹싱
│   ├── device_monitor.rs    (443줄)  ← 녹음 중 장치 변경 감지
│   ├── devices/
│   │   ├── discovery.rs              ← 장치 목록 조회
│   │   └── platform/
│   │       ├── windows.rs            ← WASAPI 루프백
│   │       ├── macos.rs              ← ScreenCaptureKit
│   │       └── linux.rs              ← ALSA / PulseAudio
│   ├── capture/
│   │   ├── microphone.rs             ← 마이크 스트림
│   │   ├── system.rs                 ← 시스템 소리 스트림
│   │   └── core_audio.rs    (445줄)  ← macOS 전용
│   └── transcription/worker.rs(608줄)← 전사 워커 스레드
│
├── whisper_engine/                    ← 🗣️ Whisper 받아쓰기
│   ├── whisper_engine.rs   (1806줄)  ← 모델 로드/추론/GPU 자동감지
│   ├── commands.rs          (568줄)  ← 다운로드, 삭제, 검증
│   └── parallel_processor.rs(480줄)  ← 긴 회의 병렬 처리
│
├── parakeet_engine/                   ← ⚡ NVIDIA Parakeet (ONNX, 초고속)
│   ├── parakeet_engine.rs  (2016줄)
│   └── model.rs             (503줄)
│
├── summary/                           ← 📝 AI 요약 엔진
│   ├── service.rs          (1110줄)  ← 요약 오케스트레이션
│   ├── llm_client.rs        (928줄)  ← ⭐ 7개 LLM 공급자 추상화
│   ├── processor.rs         (920줄)  ← 청킹 + 맵리듀스
│   ├── summary_engine/
│   │   ├── model_manager.rs (851줄)  ← 내장 GGUF 모델 관리
│   │   ├── sidecar.rs       (670줄)  ← llama-helper 프로세스 제어
│   │   └── models.rs        (493줄)  ← 모델 카탈로그
│   └── templates/                    ← 회의록 템플릿 시스템
│
├── database/                          ← 🗄️ SQLite (sqlx)
│   ├── manager.rs                    ← 커넥션 풀, WAL
│   └── repositories/                 ← meeting / transcript / summary / setting
│
├── ollama/ openai/ anthropic/ groq/ openrouter/   ← LLM 공급자별 클라이언트
├── notifications/                     ← 알림 + 방해금지(DND) 연동
├── analytics/              (572줄)   ← PostHog 사용통계 (끌 수 있음)
├── tray.rs                 (414줄)   ← 메뉴바/트레이 아이콘
├── onboarding.rs                     ← 첫 실행 플로우
└── api/api.rs             (1383줄)   ← 프론트 ↔ DB 브릿지 커맨드
```

#### `frontend/src-tauri/migrations/` — DB 스키마 진화 기록
```
20250916  초기 스키마 (meetings, transcripts, settings...)
20250920  OpenRouter API 키 추가
20251006  오디오 싱크 필드
20251010  Ollama 엔드포인트
20251101  요약 백업
20251105  PRO 라이선스 + 커스텀 OpenAI
20251110  유예기간 / speaker 필드 (화자분리 준비)
20251223  회의 노트
20251229  Gemini API 키 (최신)
```
→ 지금도 활발하게 개발 중인 프로젝트임을 보여준다.

#### `frontend/src-tauri/templates/` — 🎯 회의록 템플릿 (커스터마이즈 포인트!)
제공 템플릿 7종:
`standard_meeting` / `daily_standup` / `project_sync` / `retrospective` /
`sales_marketing_client_call` / `psychatric_session` / `README`

```json
// standard_meeting.json
{
  "name": "Standard Meeting Notes",
  "sections": [
    { "title": "Summary",       "instruction": "한 단락 요약",  "format": "paragraph" },
    { "title": "Key Decisions", "instruction": "주요 결정사항", "format": "list" },
    { "title": "Action Items",  "instruction": "담당자/기한 포함",
      "item_format": "| **Owner** | Task | Due | 근거 구간 | 타임스탬프 |" },
    { "title": "Discussion Highlights", "format": "paragraph" }
  ]
}
```
→ **JSON 하나 추가하면 나만의 회의록 양식 완성** (수익화 핵심 포인트)

### 🥈 `/llama-helper` — 내장 LLM 실행기 (별도 바이너리)
```toml
name = "llama-helper"
llama-cpp-2 = "=0.1.146"
[features] metal / cuda / vulkan
opt-level = "s"   # 로딩 속도 위해 사이즈 최적화
```
**왜 따로 있나?** → Ollama 설치 안 한 사람도 앱만 설치하면 바로 로컬 AI 요약이 되게 하려고.
Tauri 사이드카로 실행되는 작은 llama.cpp 래퍼.

내장 모델 카탈로그:
| 모델 | 용도 |
|---|---|
| `gemma3:1b` (Q8_0) | 빠름 |
| `gemma3:4b` (Q4_K_M) | 균형 |
| `qwen3.5:2b` (Q4_K_M) | 균형 |
| `qwen3.5:4b` (Q4_K_M) | 고품질 |

### 🥉 `/backend` — ⚠️ 레거시 보관소 (사용 금지!)
```
backend/
├── app/main.py              ← 옛날 FastAPI 서버
├── docker-compose.yml       ← 옛날 Docker 구성
├── Dockerfile.server-{cpu,gpu,macos}
├── whisper-custom/server/   ← 옛날 독립 whisper-server (C++)
├── debug_cors.py            ← 🚨 인증 없는 개방 CORS (보안 위험)
└── clean_start_backend.sh
```
CLAUDE.md에 **"archived and unsupported"** 로 명시. 현재는 전부 Rust로 흡수됨.
→ **이 폴더는 무시할 것. 보안상 사용 금지.**

### 📄 `/docs` — 문서 + 스크린샷/GIF
`BUILDING.md`, `building_in_linux.md`, `GPU_ACCELERATION.md`, `architecture.md` + 이미지 33MB

### 🔧 `/scripts` & `/.github`
```
scripts/
├── generate-update-manifest-github.js  ← 자동 업데이트 매니페스트 생성
├── test-update-locally.js
└── inject_transcript.py                ← 테스트용 전사본 주입

.github/workflows/
├── build-{macos,windows,linux}.yml     ← OS별 CI 빌드
├── build-devtest.yml, build-test.yml
├── pr-main-check.yml
└── release.yml                         ← 릴리즈 자동화
```
→ GitHub Actions로 3개 OS 자동 빌드 + 릴리즈 완비.

---

## 3️⃣ 기술 스택 전체

| 레이어 | 기술 |
|---|---|
| 셸 | **Tauri 2.6.2** (Rust, Electron보다 10배 가벼움) |
| UI | Next.js 14 + React 18 + TypeScript 5.7 |
| 스타일 | Tailwind 3.4 + shadcn/ui + Radix UI + framer-motion |
| 에디터 | BlockNote 0.36 + TipTap + Remirror (Notion 스타일) |
| 오디오 캡처 | `cpal`(패치) + ScreenCaptureKit(mac) / WASAPI(win) / ALSA(linux) |
| 오디오 처리 | `nnnoiseless`(RNNoise), `ebur128`(라우드니스), `rubato`(리샘플), `realfft`, `ringbuf` |
| 음성감지 | **Silero VAD** (`silero_rs`) |
| STT | **whisper-rs 0.13.2** + **ONNX Runtime(ort)** for Parakeet |
| 디코딩 | `symphonia` (aac/mp4/mp3/flac/ogg/wav) + `ffmpeg-sidecar` |
| DB | **SQLite** via `sqlx 0.8` (WAL 모드) |
| LLM | `llama-cpp-2` (내장) + `reqwest` (클라우드) |
| 비동기 | Tokio full + rayon + crossbeam + dashmap |
| 분석 | `posthog-rs` (옵트아웃 가능) |

---

## 4️⃣ 오디오 파이프라인 — 이 프로젝트의 진짜 기술력

```
      🎤 마이크              🔊 시스템 소리(상대방 목소리)
         │                        │
         └──────────┬─────────────┘
                    ▼
        ┌───────────────────────────┐
        │  AudioMixerRingBuffer     │  ← 두 스트림 속도/도착시간이 달라서
        │  (50ms 윈도우 정렬)       │     링버퍼로 싱크 맞춤
        └───────────┬───────────────┘
                    ▼
         ┌──────────┴──────────┐
         ▼                     ▼
┌─────────────────┐   ┌──────────────────────┐
│ 🎵 녹음 경로     │   │ 🗣️ 전사 경로          │
│ ProfessionalMixer│   │ Silero VAD 필터링     │
│ - RMS 기반 더킹  │   │ - 말소리만 추출        │
│ - 클리핑 방지    │   │ - 침묵/숨소리 버림     │
│ - EBU R128 정규화│   │ → Whisper 부하 70%↓   │
└────────┬────────┘   └──────────┬───────────┘
         ▼                       ▼
   WAV 파일 저장            실시간 자막 출력
   + 체크포인트 저장         (Tauri 이벤트)
```

**용어 설명**
- **더킹(ducking)**: 내가 말할 때 시스템 소리를 자동으로 줄여 내 목소리가 묻히지 않게 하는 방송 기법
- **VAD**: Voice Activity Detection. "지금 사람이 말하는 중인가?" 판별 → 침묵 버려서 AI 부하 절감
- **클리핑 방지**: 두 소리를 합칠 때 음량이 터지는 현상 방지
- **EBU R128**: 방송 표준 라우드니스 정규화

---

## 5️⃣ 지원하는 AI 공급자 (7종)

`summary/llm_client.rs` 의 `LLMProvider` enum 기준:

| 공급자 | API 키 | 비용 | 프라이버시 |
|---|:---:|:---:|:---:|
| **BuiltInAI** (llama-helper) | ❌ | 무료 | 🔒🔒🔒 완전 로컬 |
| **Ollama** (localhost:11434) | ❌ | 무료 | 🔒🔒🔒 완전 로컬 |
| **CustomOpenAI** (자체 엔드포인트) | 선택 | 자체 | 🔒🔒 내 인프라 |
| OpenAI | ✅ | 유료 | ☁️ 클라우드 |
| Claude (Anthropic) | ✅ | 유료 | ☁️ 클라우드 |
| Groq | ✅ | 유료/무료티어 | ☁️ 클라우드 |
| OpenRouter | ✅ | 모델별 | ☁️ 클라우드 |

**STT(받아쓰기)는 100% 로컬 고정** — Whisper 또는 Parakeet. API 키 불필요.

### Whisper 모델 선택 가이드
| 모델 | 용량 | 속도 | 정확도 | 추천 |
|---|---|---|---|---|
| tiny | 75MB | ★★★ | ★★ | 테스트용 |
| base | 142MB | ★★★ | ★★★ | 개발용 |
| small | 466MB | ★★ | ★★★★ | 가벼운 실사용 |
| medium | 1.5GB | ★ | ★★★★★ | 실사용 추천 |
| **large-v3-turbo** | 1.6GB | ★★ | ★★★★★ | **가성비 최고** |
| large-v3 | 3.1GB | ☆ | ★★★★★★ | 최고 품질 |
| `-q5_1`/`-q5_0` | 약 40%↓ | 더 빠름 | 거의 동일 | 용량 절약 |

---

## 6️⃣ 어떨 때 쓰는가?

### ✅ 완벽하게 맞는 상황
| 상황 | 이유 |
|---|---|
| 💼 고객사 미팅 / NDA 걸린 회의 | 데이터가 외부로 안 나감 |
| ⚖️ 법률 상담, 의료/심리 상담 | GDPR/개인정보보호법 리스크 회피 |
| 🏢 보안 엄격한 회사 (금융/방산/공공) | 클라우드 SaaS 금지 환경 |
| ✈️ 오프라인 환경 (비행기, 폐쇄망) | 인터넷 0으로 동작 |
| 💸 비용 절감 | Otter.ai·Fireflies 월 $20~30 → **$0** |
| 🎓 강의 / 인터뷰 녹취 | 긴 음성도 병렬 처리 |
| 🎙️ 기존 녹음파일 정리 | Import & Enhance 기능 |

### ❌ 안 맞는 상황
- 회의 자동 참석/자동 입장 봇 필요 (PRO 전용)
- 화자 분리(누가 말했는지) 필수 → 커뮤니티판 미지원 (DB에 `speaker` 필드만 준비)
- 모바일(iOS/Android) → 데스크톱만 지원
- 저사양 PC → Whisper medium/large 구동 어려움

---

## 7️⃣ 숨어있는 좋은 기능들 (README에 없는 것들)

| 기능 | 파일 | 설명 |
|---|---|---|
| 💾 크래시 복구 | `incremental_saver.rs` | 녹음 중 앱이 죽어도 체크포인트로 복구 |
| 🔌 장치 변경 감지 | `device_monitor.rs` | 녹음 중 에어팟 뺐다 꽂아도 안 끊김 |
| 🔄 재전사 | `retranscription.rs` | 녹음 다시 안 하고 다른 모델로 재변환 |
| 📥 파일 가져오기 | `import.rs` | 기존 mp3/m4a 넣어도 회의록 생성 |
| ⚡ 병렬 처리 | `parallel_processor.rs` | 긴 회의를 CPU 코어 나눠 동시 처리 |
| 🔇 방해금지 연동 | `notifications/` | 회의 중 알림 자동 차단 |
| 🖥️ 트레이 아이콘 | `tray.rs` | 메뉴바에서 바로 녹음 시작 |
| 🗂️ DB 마이그레이션 | `database/commands.rs` | 구버전 데이터 자동 이전 |
| 🔄 자동 업데이트 | `scripts/generate-update-manifest-github.js` | Tauri updater 연동 |

---

## 8️⃣ 프로젝트의 진화 역사 (코드에 흔적이 남아있음)

```
[1세대] 복잡한 3단 구조
  Next.js UI ←HTTP→ Python FastAPI ←HTTP→ whisper-server(C++) + Docker 3개
  흔적: /backend 폴더, backend/docker-compose.yml, frontend/API.md

        ↓ "설치가 너무 어렵다" 피드백

[2세대] Rust로 통합
  Tauri 앱 하나 = UI + 오디오 + Whisper + DB + LLM
  흔적: lib_old_complex.rs(2437줄), audio/core-old.rs

        ↓ "파일 하나가 너무 크다" 리팩토링

[3세대] 현재 v0.4.1
  모듈 계층화 완료 (audio/devices/platform/...)
  + Parakeet 추가 + 내장 LLM(llama-helper) + 템플릿 시스템
```
→ **"왜 마이크로서비스를 모놀리식으로 합쳤는가"** 라는 실전 설계 결정을 코드로 볼 수 있는 교보재.

재밌는 흔적: `package.json`에 아직 `"main": "electron/main.js"` 가 남아있음
→ 옛날엔 Electron이었다가 Tauri로 갈아탄 증거.

---

## 9️⃣ 설치 및 사용법

### 🟢 Case A. 그냥 쓰고 싶다 (초보 / 추천)

#### 🍎 macOS (Apple Silicon)
```
1. https://github.com/Zackriya-Solutions/meeting-minutes/releases/latest 접속
2. meetily_0.4.1_aarch64.dmg 다운로드
3. dmg 열고 → Meetily를 Applications로 드래그
4. ⚠️ 첫 실행: 우클릭 → "열기" (Gatekeeper)
   안 되면: 시스템 설정 → 개인정보 보호 및 보안 → "확인 없이 열기"
5. 권한 2개 허용:
   ✅ 마이크
   ✅ 화면 기록 (← 시스템 소리 캡처에 필수!)
6. 시스템 소리 녹음하려면 BlackHole 설치:
   brew install blackhole-2ch
   → 시스템 설정 > 사운드 > 출력을 "Multi-Output Device"로
```

#### 🪟 Windows
```
1. Releases에서 x64-setup.exe 다운로드 → 실행
2. ⚠️ Windows Defender 경고 → "추가 정보" → "실행"
3. 🔧 필수 조건: AVX2 지원 CPU (AVX-512는 불필요)
   - 인텔: 2013년 Haswell 이후 / AMD: 2017년 Ryzen 이후
   - 확인법: CPU-Z → Instructions 항목에 AVX2
4. 시스템 소리는 WASAPI 루프백으로 자동 캡처 (별도 설치 불필요)
※ 패키지 설치판은 Vulkan 빌드. CUDA는 소스 빌드 필요.
```

#### 🐧 Linux
```bash
git clone https://github.com/Zackriya-Solutions/meeting-minutes
cd meeting-minutes/frontend
pnpm install --frozen-lockfile
./build-gpu.sh
```

### 🔵 Case B. 개발하고 싶다

```bash
# ===== 사전 준비물 =====
# 1) Rust 1.77+
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 2) Node.js 20+ & pnpm
npm install -g pnpm

# 3) OS별 추가
#  macOS:   xcode-select --install
#  Windows: Visual Studio Build Tools (C++ 워크로드)
#  Linux:   sudo apt install cmake llvm libomp-dev \
#             libwebkit2gtk-4.1-dev libasound2-dev pkg-config

# ===== 실행 =====
git clone https://github.com/bmshin94/meetily
cd meetily/frontend
pnpm install

pnpm run tauri:dev      # 개발 모드 (핫리로드) — 가장 많이 쓸 명령

./clean_run.sh          # info 로그
./clean_run.sh debug    # debug 로그
./clean_run.sh trace    # trace 로그

RUST_LOG=app_lib::audio=debug ./clean_run.sh   # 오디오만 디버깅
```

#### GPU별 빌드 명령
```bash
pnpm run tauri:dev:metal     # 🍎 맥 (M시리즈)
pnpm run tauri:dev:coreml    # 🍎 맥 (CoreML 추가 가속)
pnpm run tauri:dev:cuda      # 🟢 엔비디아
pnpm run tauri:dev:vulkan    # 🔴 AMD / 🔵 인텔
pnpm run tauri:dev:hipblas   # 🔴 AMD ROCm (리눅스)
pnpm run tauri:dev:openblas  # CPU 최적화
pnpm run tauri:dev:cpu       # 순수 CPU

pnpm run tauri:build         # 프로덕션 빌드 (설치파일 생성)
./clean_build.sh             # 맥
clean_build_windows.bat      # 윈도우
```

#### 코드 수정 포인트 치트시트
| 뭘 바꾸고 싶나? | 어디로 |
|---|---|
| 메인 화면 UI | `frontend/src/app/page.tsx` |
| 설정 화면 | `frontend/src/app/settings/page.tsx` |
| **회의록 양식 추가** | `frontend/src-tauri/templates/*.json` ← **제일 쉬움** |
| 프롬프트 수정 | `src-tauri/src/summary/templates/defaults.rs` |
| 새 LLM 공급자 | `src-tauri/src/summary/llm_client.rs` (enum 추가) |
| 오디오 믹싱 | `src-tauri/src/audio/pipeline.rs` |
| Whisper 모델 추가 | `src-tauri/src/whisper_engine/whisper_engine.rs:1077` |
| 새 Tauri 커맨드 | `src-tauri/src/lib.rs:612` (`generate_handler!`) |
| DB 스키마 | `src-tauri/migrations/` 에 새 .sql 파일 |

### 🟣 실제 사용 흐름
```
① 첫 실행 → 온보딩 마법사
     └ Whisper 모델 다운로드 (large-v3-turbo 추천, 1.6GB)
     └ AI 공급자 선택
        ├ BuiltInAI : 아무것도 안 해도 됨, 모델만 받으면 끝
        ├ Ollama    : ollama serve + ollama pull qwen2.5:7b
        └ 클라우드  : API 키 입력
② 설정 → 오디오 장치 선택 (마이크 + 시스템)
③ 🔴 Record 버튼 → 회의 이름 입력
④ 회의 중 → 실시간 자막, ⏸️ 일시정지 가능
⑤ ⏹️ Stop
⑥ "Generate Summary" → 템플릿 선택 → 1~3분 대기
⑦ 📄 회의록 완성 → BlockNote 에디터에서 수정 → Markdown 복사
```

---

## 🔟 "플러그인? 스킬? MCP?" — 정체 규명

### 정답: **전부 아니다. 독립 데스크톱 애플리케이션이다.**

| 종류 | 정체 | Meetily? |
|---|:---:|---|
| 🔌 플러그인 | 다른 앱에 끼워넣는 확장 (VSCode/크롬 확장) | ❌ |
| 🎓 스킬 | Claude에게 주는 지침서 폴더 (SKILL.md + 스크립트) | ❌ |
| 🔗 MCP 서버 | AI가 도구를 쓰게 해주는 표준 프로토콜 서버 | ❌ |
| 📦 **독립 앱** | 더블클릭하면 실행되는 완성품 프로그램 | ✅ **이것** |

→ Notion이나 Slack처럼 **그냥 설치해서 쓰는 앱**.

### 💡 하지만 직접 변환 가능!

#### 변환 A: MCP 서버로 감싸기 ⭐⭐⭐⭐⭐ (가장 유망)
```typescript
// meetily-mcp-server (신규 제작)
// Tauri 앱의 SQLite를 직접 읽고, Rust 엔진을 CLI로 호출
tools: [
  search_meetings,          // 회의 키워드 검색
  get_meeting_transcript,   // 전체 전사본
  get_action_items,         // 미완료 액션아이템
  transcribe_audio_file,    // 로컬 전사 (프라이버시 보장)
  summarize_with_template,  // 템플릿 적용 요약
]
```
→ Claude에게 **"지난 달 김부장님과의 회의 결정사항 찾아서 정리해줘"** 가 가능해진다.

#### 변환 B: Claude Skill로 만들기 ⭐⭐⭐⭐
```
.claude/skills/meeting-notes/
├── SKILL.md                 ← "회의록 작성 전문가" 지침
├── templates/*.json         ← Meetily 템플릿 그대로 재활용 (MIT라 합법)
└── scripts/parse_transcript.py
```

#### 변환 C: CLI 도구로 추출 ⭐⭐⭐
```bash
meetily transcribe meeting.mp3 --model large-v3-turbo --lang ko
meetily summarize transcript.txt --template retrospective
```

---

## 1️⃣1️⃣ API 토큰을 사용해야 하나?

### 결론: **안 써도 전부 다 된다.**

```
🎙️ 받아쓰기 (STT)
  → Whisper / Parakeet, 100% 로컬 → 🔑 API 키 아예 사용 안 함

📝 AI 요약
  → 선택 1: BuiltInAI (llama-helper) 🔑 불필요
  → 선택 2: Ollama (localhost)       🔑 불필요
  → 선택 3: 클라우드                 🔑 필요 (선택)
```

### 조합별 비교
| 조합 | API 키 | 월 비용 | 품질 | 속도 | 프라이버시 |
|---|:---:|:---:|:---:|:---:|:---:|
| Whisper + BuiltInAI | ❌ | ₩0 | ★★★ | 느림 | 🔒🔒🔒 |
| **Whisper + Ollama** (qwen2.5:7b) | ❌ | **₩0** | ★★★★ | 보통 | 🔒🔒🔒 |
| Whisper + Groq | ✅ | 거의 무료 | ★★★★ | 매우 빠름 | ☁️ |
| Whisper + Claude | ✅ | $3~15/1M토큰 | ★★★★★★ | 빠름 | ☁️ |
| Whisper + OpenAI | ✅ | $2.5~10/1M | ★★★★★ | 빠름 | ☁️ |
| Whisper + CustomOpenAI | 선택 | 자체 | 자유 | 자유 | 🔒🔒 |

**실제 비용 감각**: 1시간 회의 ≈ 10,000단어 ≈ 15,000토큰 (청킹으로 실제 2~3배)
→ Claude Sonnet 1회 요약 약 $0.1~0.3 (150~450원). 하루 2회면 월 1~2만원.
→ Ollama 쓰면 **0원**.

### API 키 저장 위치
```sql
-- settings 테이블 (로컬 SQLite)
groqApiKey, openaiApiKey, anthropicApiKey,
ollamaApiKey, openRouterApiKey, geminiApiKey
```
> ⚠️ 확인한 범위에선 **평문 저장**. OS 키체인(keyring) 연동으로 개선 필요 → 제품화 전 필수 과제.

### 분석(Analytics) 체크
```
analytics/analytics.rs → PostHog로 사용통계 전송 (기본 ON)
  전송: 이벤트명, 앱 버전, 세션ID, meeting_id
  전송 안 함: 전사본 내용, API 키 (SENSITIVE_ANALYTICS_KEYS로 필터)
  끄기: 설정에서 disable_analytics
```
→ 회의 내용은 안 나가지만, 완전 오프라인 원하면 끄는 것 권장.

---

## 1️⃣2️⃣ AI 에이전트 구축에 도움이 되는가?

### 결론: **매우 도움 된다. 특히 "로컬 우선 에이전트" 교보재로 최고.**

### 훔쳐올 수 있는 에이전트 패턴 7개

| # | 패턴 | 파일 | 왜 중요한가 |
|:-:|---|---|---|
| 1 | **멀티 LLM 공급자 추상화** | `summary/llm_client.rs` (928줄) | 공급자 7개를 하나의 인터페이스로. 모델별 예외처리(`reasoning_effort` 미지원 감지 후 재시도)까지 실전 노하우 포함 |
| 2 | **사이드카 프로세스로 로컬 LLM** | `summary_engine/sidecar.rs` (670줄) + `/llama-helper` | LLM 크래시해도 앱 안 죽음 / 메모리 회수 / GPU 피처 별도 컴파일 |
| 3 | **맵리듀스 청킹** | `summary/processor.rs` (920줄) | 컨텍스트 윈도우 초과 문제 해결. 오버랩 청킹 + 계층 요약 |
| 4 | **구조화된 출력 (템플릿)** | `summary/templates/` | 프롬프트를 코드가 아닌 **데이터(JSON)** 로 관리 → 비개발자도 수정 가능 |
| 5 | **작업 취소** | `CancellationToken` 사용처 | 긴 작업 중단. 없으면 UX 최악 |
| 6 | **진행률 스트리밍** | `app.emit("transcript-update", ...)` | "AI가 지금 뭐하는지" 실시간 표시 |
| 7 | **리소스 인지 스케줄링** | `whisper_engine/parallel_commands.rs` | `get_system_resources` / `check_resource_constraints` / `calculate_optimal_workers`. 로컬 에이전트는 사용자 자원을 공유하므로 필수 |

### 만들 수 있는 에이전트 구성
```
[Meetily (전사+저장)] ──MCP──┐
[캘린더 MCP] ─────────────────┤
[Slack MCP] ──────────────────├──→ 🧠 Claude 에이전트
[Notion MCP] ─────────────────┘       ├→ "어제 회의 액션아이템 Slack에 올려"
                                      ├→ "다음주 회의 안건 자동 생성"
                                      ├→ "내가 약속한 거 뭐 있지?"
                                      └→ "김부장님 요구사항 전체 타임라인"
```

### 추천 학습 루트
```
Week 1  templates/*.json       → 프롬프트 엔지니어링을 데이터로
Week 2  llm_client.rs           → 멀티 공급자 추상화
Week 3  processor.rs            → 청킹 / 맵리듀스
Week 4  sidecar.rs + llama-helper → 로컬 LLM 통합
Week 5  pipeline.rs             → 실시간 스트림 처리
Week 6  나만의 MCP 서버 제작    → 실전
```

---

## 1️⃣3️⃣ React나 PHP로 만들 수 있는가?

### ⚛️ React → **UI는 이미 React다** ✅
```
frontend/src/  ← 전부 React 18 + Next.js 14 + TypeScript
```
```tsx
import { invoke } from '@tauri-apps/api/core';
import { listen } from '@tauri-apps/api/event';

await invoke('start_recording', { meeting_name: '팀 회의' });
await listen<TranscriptUpdate>('transcript-update', (e) => {
  setTranscripts(prev => [...prev, e.payload]);
});
```

**React만으로 못 하는 것**
| 기능 | 왜 안 되는가 |
|---|---|
| 🔊 시스템 소리 캡처 | 브라우저 보안상 OS 오디오 접근 불가 (화면공유 API로 탭 소리만, 제한적) |
| 🗣️ Whisper 로컬 실행 | WASM은 10~50배 느림, 모델 3GB 브라우저 로드 비현실적 |
| ⚡ GPU(Metal/CUDA) | 브라우저는 WebGPU만, whisper.cpp 사용 불가 |
| 💾 파일 시스템 | 샌드박스 제한 |
| 🖥️ 트레이 아이콘 | 네이티브 영역 |

### 🐘 PHP → **일부만 가능**
| 할 수 있는 것 | 못 하는 것 |
|---|---|
| ✅ 웹 대시보드 (회의 목록/검색) | ❌ 실시간 오디오 캡처 |
| ✅ 업로드된 오디오 받기 | ❌ 브라우저에서 Whisper |
| ✅ LLM API 호출 (요약) | ❌ GPU 가속 로컬 추론 |
| ✅ 팀 공유/권한/결제 | ❌ 저지연 스트리밍 처리 |
| ✅ Whisper CLI `exec()` 호출 | ❌ (성능/동시성 문제 큼) |

### 추천 설계 3안

#### 🅰️ 포크 개선형 ⭐⭐⭐⭐⭐ (최우선 추천)
```
Meetily 그대로 포크 → React UI 한국화 + 템플릿 추가
  수정: frontend/src/**            (React)
  수정: src-tauri/templates/*.json (JSON — 쉬움)
  유지: Rust 엔진                  (안 건드림)
👍 2~4주면 "한국형 Meetily" 출시 가능 / Rust 거의 안 배워도 됨
```

#### 🅱️ 하이브리드 (React + PHP) ⭐⭐⭐⭐ (수익화 최적)
```
┌──────────────────────────┐        ┌─────────────────────┐
│  🖥️ 데스크톱 (Meetily)    │        │  🌐 웹 (PHP/Laravel) │
│  = 녹음 + 로컬 전사       │ ─HTTPS→│  = 팀 공유 대시보드  │
│  (민감 음성은 로컬에!)    │ 암호화 │  = 검색/권한/결제    │
│  Rust + React            │  전사본 │  = PHP + React SPA   │
└──────────────────────────┘        └─────────────────────┘
👍 "음성은 절대 안 올라감, 텍스트만" = 프라이버시 셀링포인트
👍 PHP/Laravel로 SaaS 수익모델(구독/팀/결제) 구현
```

#### 🅲 웹 온리 ⭐⭐ (비추천)
```
React + PHP만으로 → 음성을 서버 업로드 → 서버에서 Whisper
❌ 핵심 가치(프라이버시) 상실 / GPU 서버 비용 폭발 / 레드오션
```

---

## 1️⃣4️⃣ 유튜브 강의 영상 제작 가능한가?

### 결론: **가능. 소재 퀄리티가 아주 좋다.**

### 법적 검토
```
라이선스: MIT ✅
├─ 코드 리뷰/강의 OK
├─ 수정해서 보여주기 OK
├─ 상업적 이용 OK (유튜브 수익창출 포함)
├─ 유료 강의 판매 OK
└─ 조건: 저작권 고지 + MIT 라이선스 사본 유지

⚠️ 지킬 것
├─ 원작자 크레딧 명시 (Zackriya Solutions)
├─ "내가 만들었다" 주장 금지
└─ 데모 영상은 본인/동의받은 음성만 (초상권/음성권)
```

### 추천 커리큘럼

#### 트랙 1: 입문자용 (조회수)
| # | 제목 | 길이 | 훅 |
|:-:|---|:-:|---|
| 1 | 월 3만원 내던 회의록 앱, 공짜로 만들었다 | 12분 | 비용 비교 |
| 2 | 내 노트북에서 AI 받아쓰기 (인터넷 OFF) | 15분 | 랜선 뽑는 연출 |
| 3 | 회의록 양식 커스터마이징 (JSON 하나로) | 10분 | 코딩 거의 없음 |
| 4 | Ollama로 100% 무료 AI 요약 세팅 | 18분 | 0원 |
| 5 | Mac/Windows 시스템 소리 녹음 완전정복 | 14분 | 모두의 고충 |

#### 트랙 2: 개발자용 (유료강의화)
| # | 제목 | 길이 | 가치 |
|:-:|---|:-:|---|
| 1 | Tauri 2.0 실전: Electron 안녕 | 25분 | 희소 주제 |
| 2 | Rust 오디오 파이프라인 해부 (링버퍼+VAD) | 35분 | ⭐ 국내 자료 거의 없음 |
| 3 | whisper-rs로 로컬 STT 붙이기 | 30분 | 실용 |
| 4 | LLM 공급자 7개를 하나의 인터페이스로 | 28분 | ⭐ 에이전트 필수 |
| 5 | 사이드카 패턴: 로컬 LLM 안전하게 돌리기 | 32분 | ⭐⭐ 고급 |
| 6 | 긴 문서 요약: 맵리듀스 청킹 구현 | 26분 | ⭐ 에이전트 필수 |
| 7 | GitHub Actions로 3-OS 자동 빌드 | 24분 | DevOps |
| 8 | **Meetily를 MCP 서버로 만들기** | 40분 | 🔥 최신 트렌드 |

#### 트랙 3: 비즈니스 (전환율)
```
1. 오픈소스로 월 500만원 만드는 법 (MIT 활용법)
2. 한국형 AI 회의록 SaaS 만들기 1편 - 기획
3. 2편 - React 프론트 한국화
4. 3편 - Laravel로 팀 공유 백엔드
5. 4편 - 결제 붙이고 런치하기
```

### 제작 팁
```
✅ 비주얼 훅
   - 랜선 뽑고/비행기모드 → "인터넷 없는데 되네?"
   - Activity Monitor로 GPU 사용률 보여주기
   - Otter.ai 요금제 vs 0원 비교 샷
   - 터미널 로그 흐르는 거 (./clean_run.sh debug)

✅ 차별화 (한국 유튜브에 거의 없는 주제)
   - Tauri 2.x 한국어 강의 → 거의 없음
   - Rust 실시간 오디오 → 거의 없음
   - 로컬 LLM 통합 패턴 → 거의 없음
   - MCP 서버 제작 → 생기는 중 (선점 기회)

⚠️ 주의
   ❌ /backend 폴더 가르치지 말 것 (레거시 + CORS 취약)
   ❌ API 키 노출 주의 (블러 필수)
   ❌ 실제 회사 회의 녹음 사용 금지 (NDA/초상권)
   ✅ 더미 회의 미리 녹음 (본인 목소리 2인 롤플레이)
   ✅ 버전 명시 ("v0.4.1 기준")
```

---

## 1️⃣5️⃣ 💰 수익화 아이디어 (상세)

### 전체 지도
```
            수익 잠재력 ↑
    높음 │  ③산업특화판        ②MCP SaaS
         │  (법무/의료)        (에이전트 연동)
         │
         │        ①한국형 Meetily
    중간 │        (로컬라이즈 + 템플릿)
         │
         │  ⑤유튜브/강의    ④구축대행
    낮음 │  ⑥템플릿마켓
         └──────────────────────────────→
            쉬움              어려움   난이도
```

### 🥇 ① 한국형 Meetily

**인사이트**: 원본은 영어권 중심. 한국 시장에 구조적 공백이 있다.

| 문제 | 한국 상황 |
|---|---|
| 🇰🇷 한국어 전사 정확도 | Whisper 한국어는 영어보다 떨어짐 → 후처리 필요 |
| 🗣️ 한국어 특유 문제 | 존댓말, 직급("부장님"), 외래어 혼용("어그리합니다") |
| 📝 한국식 회의록 양식 | 품의/결재/주간보고 양식이 서양식과 완전 다름 |
| 💬 비즈니스 용어 | "ASAP", "FYI", "리소스 태우다", "컨펌" |
| 🏢 협업툴 | Notion보다 카카오워크 / 네이버웍스 / 잔디 |

**로드맵**
```
Phase 1 (2주) — 로컬라이즈 [쉬움]
├─ UI 한글화 (frontend/src/** — React만)
├─ 한국식 템플릿 10종 JSON 추가
│   주간업무보고 / 임원보고 / 품의서초안 / 고객상담일지 / 프로젝트킥오프
│   스프린트회고 / 1on1면담 / 채용면접평가 / 영업미팅 / 교육강의노트
└─ 한국어 프롬프트 튜닝 (defaults.rs)

Phase 2 (3주) — 정확도 개선 [중간]
├─ 한국어 전사 후처리 (사용자 사전: 회사명/인명/제품명 치환)
├─ 직급 자동 정규화 ("김부장님" → "김OO 부장")
├─ 숫자/날짜 한글 정규화 ("삼천만원" → "30,000,000원")
└─ 외래어 표준화 ("어그리" → "동의")

Phase 3 (3주) — 통합 [중간]
├─ 카카오워크 / 네이버웍스 webhook 전송
├─ 네이버 / 구글 캘린더 연동
└─ 한글(HWP) / 엑셀 내보내기  ← 한국에선 매우 중요

Phase 4 (2주) — 배포 [쉬움]
├─ 코드 서명 (Apple Developer $99/년, Windows EV 인증서)
├─ 자동 업데이트 (이미 구현됨)
└─ 랜딩페이지 + 결제
```

**가격 설계**
| 플랜 | 가격 | 내용 |
|---|---|---|
| 🆓 Free | ₩0 | Whisper base/small, 템플릿 3종, 월 10회 |
| ⭐ Pro | ₩9,900/월 | 전 모델, 템플릿 전체, 무제한, HWP 내보내기 |
| 👥 Team | ₩7,900/인/월 (5인+) | 팀 공유, 관리자, 통합 연동 |
| 🏢 Enterprise | 별도 견적 | 온프레미스, 커스텀 템플릿, 전담 지원 |

**예상(보수적)**: 6개월차 월 ₩99만 → 1년차 월 ₩600만 (연 7,200만)

**리스크**: 원본팀 PRO의 한국 진출 → "HWP + 카카오워크" 특화로 차별화 /
네이버 클로바노트 무료 → "클라우드 업로드 안 함"으로 차별화

---

### 🥈 ② Meetily MCP 서버 — "내 회의 기억을 AI에게" ⭐ 최우선 추천

**인사이트**: 지금 AI 에이전트 시장에서 가장 부족한 게 **개인/조직의 맥락(context)**.

```
현재 Claude/ChatGPT: "제가 당신의 회의 내용은 알 수 없습니다"

MCP 연결 후:         "지난 분기 회의 12건을 분석했습니다.
                     예산 관련 결정이 3번 뒤집혔고,
                     김부장님이 약속한 7건 중 2건만 완료됐네요."
```

**아키텍처**
```
┌─────────────────────────────────────────────────────────┐
│  🖥️ 사용자 PC                                            │
│  ┌──────────────┐    ┌─────────────────────────────┐    │
│  │  Meetily     │    │  meetily-mcp-server (NEW)   │    │
│  │  (녹음/전사) │───→│  - SQLite 직접 읽기          │    │
│  │  meetily.db  │←───│  - Whisper CLI 호출          │    │
│  └──────────────┘    │  - 로컬 임베딩 검색(RAG)     │    │
│                      └──────────┬──────────────────┘    │
│                                  │ MCP (stdio)          │
│                      ┌───────────▼────────────┐          │
│                      │ Claude Desktop         │          │
│                      │ Claude Code / Cursor   │          │
│                      └────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
          ※ 회의 내용은 PC를 절대 안 떠남
```

**도구(tools) 설계**
```typescript
// 조회 계열
search_meetings(query, date_range, participants?)
get_meeting(meeting_id)
get_transcript(meeting_id, with_timestamps?)
get_summary(meeting_id)
list_action_items(status: "open"|"done"|"all", owner?)
semantic_search(query)              // 로컬 임베딩 RAG ⭐

// 분석 계열
find_decisions(topic)               // "예산" 관련 결정 전부
track_commitment(person)            // "김부장이 약속한 것들"
detect_contradictions(topic)        // 결정이 뒤집힌 지점 찾기
meeting_stats(date_range)

// 실행 계열
transcribe_file(path, model, lang)
generate_summary(meeting_id, template)
create_followup_agenda(meeting_id)
export_report(format: "md"|"docx"|"hwp")
```

**킬러 유즈케이스 (영업 포인트)**
```
"지난 3개월 회의에서 '출시일' 언급된 거 다 찾아서 타임라인으로"
"내가 약속해놓고 안 한 거 뭐 있어?"
"A안 vs B안 논쟁이 어떻게 결론났는지 근거 타임스탬프랑 같이"
"이번 주 회의 전부 요약해서 주간보고 초안 써줘"
"신입한테 프로젝트 히스토리 브리핑 문서 만들어줘"
"같은 안건이 몇 번 반복됐는지 세줘"  ← 회의 문화 개선
```

**가격**
| 플랜 | 가격 | 내용 |
|---|---|---|
| 🆓 OSS | ₩0 | 기본 MCP 서버 오픈소스 공개 (마케팅용) |
| ⭐ Pro | $15/월 | 시맨틱 검색(RAG), 고급 분석, 다중 클라이언트 |
| 👥 Team | $12/인/월 | 팀 지식베이스 통합, 권한 관리 |
| 🏢 Enterprise | 견적 | 온프레미스, SSO, 감사로그 |

**유망한 이유**
```
✅ MCP는 폭발적 성장 중 (Claude/Cursor/Windsurf 등 지원)
✅ "로컬 프라이버시 + 에이전트" 조합은 경쟁자 거의 없음
✅ 기술 장벽 낮음 (TypeScript MCP 서버 + SQLite 읽기)
✅ Meetily가 데이터 수집을 이미 해줌 → "두뇌"만 만들면 됨
✅ React/TS 스킬로 바로 시작 가능
```

**개발 기간**: MVP 2주 + RAG 2주 + 분석/리포트 3주 = **7주면 유료 출시**

---

### 🥉 ③ 산업 특화판 — "컴플라이언스가 돈이다"

**인사이트**: 클라우드 금지 산업은 대안이 없어서 비용 지불 의사가 높다.

| 산업 | 규제 | 왜 Meetily가 답인가 | 객단가 |
|---|---|---|---|
| ⚖️ 법무/법률 | 변호사법, 비밀유지의무 | 상담내용 외부 전송 불가 | 💰💰💰💰💰 |
| 🏥 의료/심리 | 의료법, 개인정보보호법 | 환자정보 로컬 필수 (`psychatric_session.json` 이미 존재) | 💰💰💰💰💰 |
| 🏦 금융/보험 | 전자금융감독규정, 금소법 | 상담 녹취 의무 + 외부전송 금지 | 💰💰💰💰💰 |
| 🏛️ 공공/국방 | 국가계약법, 보안업무규정 | 망분리 환경 (인터넷 자체 없음) | 💰💰💰💰 |
| 🔬 R&D/제조 | 영업비밀, 특허 | 기술 논의 유출 방지 | 💰💰💰💰 |
| 👔 HR/인사 | 근로기준법, 개인정보 | 면담/징계 기록 민감 | 💰💰💰 |

**특화 작업 (예: 법무 버전)**
```
① 템플릿: 상담일지 / 사건회의 / 증인신문요지 / 내부케이스리뷰
② 용어 사전: 법률 용어 전사 교정 ("기소유예", "항소심", "준항고")
③ 컴플라이언스 기능 ← 💰 돈 받는 부분
   ├─ 🔐 전사본/DB 암호화 (AES-256, 현재 평문 → 개선)
   ├─ 📋 감사 로그 (누가 언제 열람)
   ├─ ⏱️ 보존기간 자동 관리 + 자동 파기
   ├─ 🙈 자동 비식별화 (이름/주민번호/전화번호 마스킹)
   ├─ 🔑 OS 키체인 연동 (API키 평문저장 문제 해결)
   ├─ 📄 규정 준수 리포트 자동 생성
   └─ ✍️ 전자서명/타임스탬프 (무결성 증명)
④ 배포: 온프레미스 설치 + 망분리 환경 지원
```

**가격 (B2B)**
```
🏢 사이트 라이선스:   ₩300만 ~ 1,500만 / 년
🔧 구축/커스터마이즈: ₩500만 ~ 3,000만 (일회성)
🛟 연간 유지보수:     라이선스의 20%
🎓 교육/온보딩:       ₩200만 ~

목표: 년 5개 사이트 × ₩800만 = ₩4,000만
     + 구축 3건 × ₩1,500만 = ₩4,500만
     → 연 매출 8,500만원
```

**리스크**: 영업 사이클 3~12개월 / 레퍼런스 1호 확보가 난관
→ 지인·소규모 업체부터 무료 파일럿 권장 / ISMS-P 등 보안 인증 요구 가능

---

### 🏅 ④ 구축 대행 서비스 — "제일 빨리 돈 되는 것"

**인사이트**: 오픈소스의 영원한 진리 — "공짜지만 설치가 어렵다."

실제 설치 허들:
```
❌ BlackHole 설치 + Multi-Output Device 설정 (맥)
❌ 화면 기록 권한 (왜 필요한지 이해 못 함)
❌ AVX2 CPU 확인 (윈도우)
❌ Whisper 모델 선택 (7종 선택지)
❌ Ollama 설치 + 모델 pull
❌ GPU 빌드 (CUDA 툴체인)
❌ 한국어 정확도 튜닝
```

**상품화**
| 상품 | 가격 | 소요 | 내용 |
|---|---|---|---|
| 🔧 개인 원격설치 | ₩15만 | 1시간 | 설치+모델+권한+템플릿 1개 |
| 🏢 소규모 팀 (5~20인) | ₩150~300만 | 1~2주 | 전원 설치, 템플릿 커스텀, 교육 |
| 🎯 템플릿 제작 | ₩30만/개 | 2일 | 업무 양식 분석 → JSON 설계 + 프롬프트 튜닝 |
| ⚡ GPU 최적화 | ₩50만 | 3일 | CUDA/Vulkan 빌드 + 벤치마크 |
| 🛟 연간 유지보수 | ₩100만/년 | - | 업데이트, 장애대응, Q&A |

**장점**
```
✅ 초기 투자 ₩0 / 기술 리스크 ₩0
✅ 당장 시작 가능 (크몽/숨고/탈잉 등록)
✅ 고객 피드백 → ①②③ 제품 기획 재료
✅ 레퍼런스 축적 → B2B 영업 자산
```
**전략**: ④로 현금흐름 만들며 ②(MCP SaaS) 개발 → ④ 고객이 ②의 첫 고객

---

### 🎖️ ⑤ 유튜브 + 온라인 강의

```
1️⃣ 애드센스         월 ₩30~200만 (구독 1만명 기준)
2️⃣ 유료강의         ₩8만 × 300명 = ₩2,400만 (1회 제작, 반복 판매)
3️⃣ 코드 템플릿      "한국형 Meetily 스타터킷" ₩15만 × 100 = ₩1,500만
4️⃣ 멤버십           월 ₩9,900 × 200명 = 월 ₩198만
5️⃣ 기업 교육        "사내 로컬 AI 구축" ₩300만/회
6️⃣ 컨설팅 유입      영상 → DM → ③④로 연결
```
**블루오션 근거**: "Tauri 강의", "Rust 오디오", "MCP 서버 만들기" 한국어 자료 거의 없음

---

### 🎗️ ⑥ 템플릿 마켓플레이스

**인사이트**: Meetily 템플릿은 그냥 JSON 파일 → 누구나 만들고 거래 가능.

```json
// 예: 투자유치_IR미팅.json
{
  "name": "VC 미팅 기록",
  "sections": [
    { "title": "투자사 정보", "format": "list" },
    { "title": "제기된 질문/우려", "format": "list" },
    { "title": "우리가 약속한 자료", "item_format": "| 자료 | 기한 | 담당 |" },
    { "title": "다음 단계", "format": "list" },
    { "title": "느낌/온도감", "format": "paragraph" }
  ]
}
```

**판매 모델 (Gumroad / Lemon Squeezy)**
```
"스타트업 팩" (10종)     $19
"영업/세일즈 팩" (8종)   $19
"HR/채용 팩" (8종)       $19
"의료상담 팩" (6종)      $29
"법무 팩" (6종)          $29
"전체 번들" (50종)       $79
→ 제작비 거의 0원, 무한 복제, 마진 95%+
```

**확장**: 웹 마켓플레이스 (React + PHP/Laravel)
크리에이터 70% / 플랫폼 30%, 평점·리뷰, 미리보기, 앱 내 "템플릿 스토어" 연동

---

### 🗺️ 실행 플랜 (React + PHP 기반)

```
┌─ Month 1 ──────────────────────────────────────┐
│ ✅ Meetily 설치 + 완전 숙달                     │
│ ✅ 한국식 템플릿 10종 제작 (⑥ 판매 준비)        │
│ ✅ 크몽/숨고에 "AI 회의록 구축" 등록 (④)        │
│ ✅ 유튜브 입문 영상 3편 (⑤)                     │
│ 🎯 목표: ₩50~150만                              │
└────────────────────────────────────────────────┘
┌─ Month 2-3 ────────────────────────────────────┐
│ ✅ 포크 → React UI 한글화 (①)                   │
│ ✅ meetily-mcp-server MVP (②) ← TypeScript      │
│ ✅ 유튜브 개발자 트랙 시작 (⑤)                  │
│ ✅ ④ 고객 3~5건 처리하며 니즈 파악              │
│ 🎯 목표: 월 ₩200~400만                          │
└────────────────────────────────────────────────┘
┌─ Month 4-6 ────────────────────────────────────┐
│ ✅ MCP Pro 유료 출시 (②) — $15/월               │
│ ✅ Laravel 팀 대시보드 (🅱️ 하이브리드 구조)      │
│ ✅ 템플릿 마켓 웹 오픈 (⑥)                      │
│ ✅ B2B 파일럿 1곳 확보 (③)                      │
│ 🎯 목표: 월 ₩500~800만                          │
└────────────────────────────────────────────────┘
┌─ Month 7-12 ───────────────────────────────────┐
│ ✅ 한국형 Meetily 정식 출시 (①)                 │
│ ✅ 산업 특화판 1종 완성 (③)                     │
│ ✅ 유료 강의 런치 (⑤)                           │
│ 🎯 목표: 월 ₩1,000만+ / 연 1억                  │
└────────────────────────────────────────────────┘
```

### ⚠️ 수익화 법적 체크리스트
```
✅ MIT 라이선스 고지 유지 (LICENSE.md 포함 배포)
✅ "Zackriya Solutions" 원작 크레딧 명시
✅ Whisper(MIT) / whisper.cpp(MIT) / Parakeet(NVIDIA 라이선스 확인 필요)
✅ 상표권 주의 → "Meetily"는 원작 브랜드. 내 제품은 다른 이름으로
   (❌ "Meetily Korea"   ✅ "회의노트" + "Powered by Meetily")
✅ 녹음 동의 고지 기능 필수
   ⚠️ 통신비밀보호법: 제3자간 대화 녹음은 불법
✅ 개인정보처리방침 작성 (B2B 필수)
✅ /backend 레거시 코드는 배포에서 제거 (CORS 취약점)
✅ API 키 평문저장 → OS 키체인 전환 (제품화 전 반드시)
```

---

## 1️⃣6️⃣ ⚠️ 주의할 점 (리스크 정리)

| 심각도 | 항목 | 내용 |
|:---:|---|---|
| 🔴 | `/backend` 폴더 | 레거시 + 인증 없는 CORS. **사용 금지** |
| 🟠 | API 키 평문 저장 | SQLite `settings` 테이블에 평문. 키체인 전환 필요 |
| 🟡 | PostHog 분석 | 기본 ON (민감키는 필터링됨, 끌 수 있음) |
| 🟡 | `lib_old_complex.rs` (2437줄) | 미사용 데드 코드 |
| 🟡 | `core-old.rs`, `recording_saver_old.rs` | 리팩토링 잔재 |
| 🟡 | macOS 시스템 소리 | BlackHole 가상 오디오 장치 별도 설치 필요 |
| 🟡 | Windows 설치판 | AVX2 지원 CPU 필수 (구형 CPU 불가) |
| 🟡 | 모델 용량 | large-v3 ~3GB, GGUF LLM 1~3GB 다운로드 |
| 🟡 | 화자분리 없음 | 커뮤니티판 미지원 (PRO 예정) |
| 🔵 | `vs_buildtools.exe` (4.4MB) | 바이너리 설치파일이 레포에 커밋돼 있음 |
| 🔵 | `package.json` → `"main": "electron/main.js"` | Electron 흔적 (실제론 Tauri) |

---

## 1️⃣7️⃣ 📌 최종 결론

> **Meetily는 "MIT 라이선스로 자유롭게 쓸 수 있는, 기술적으로 아주 잘 만들어진
> 로컬 AI 회의록 앱"이며, 동시에 다음 3가지 가치를 가진다.**

### ① 실용 가치
- 회의/강의/인터뷰 녹취를 **월 0원**으로 무한히 (Otter.ai 연 $240 절약)
- 회의록 작성 시간 **30분 → 1분**
- Action Item이 담당자 + 기한 + 근거 타임스탬프까지 자동 표로 정리

### ② 학습 가치 (최고급 교보재)
| 배울 수 있는 것 | 위치 |
|---|---|
| Tauri 2.x 실전 패턴 | `lib.rs` 커맨드 200개 등록 구조 |
| Rust 비동기 설계 | `Arc<RwLock<T>>`, `AtomicBool`, mpsc 채널 |
| 실시간 오디오 DSP | 링버퍼, RMS 더킹, VAD, 리샘플링 |
| 로컬 LLM 통합 | 사이드카 프로세스 + GGUF 모델 관리 |
| 멀티 LLM 추상화 | 7개 공급자를 하나의 인터페이스로 |
| 크로스플랫폼 오디오 | WASAPI / ScreenCaptureKit / ALSA |
| GPU 가속 피처 플래그 | Metal/CUDA/Vulkan/HIP Cargo features |
| Next.js ↔ Rust 브릿지 | invoke + event 양방향 |
| sqlx 마이그레이션 운영 | 10개 마이그레이션 진화 과정 |
| 3-OS CI/CD | GitHub Actions 빌드 매트릭스 |

### ③ 비즈니스 가치
- MIT = 상업적 이용/수정/재배포 자유 (저작권 표기만 유지)
- 원본 팀이 이미 PRO($)/Enterprise로 수익화 성공 → **검증된 시장**
- 니치 공략 가능: "한국 시장 최적화", "산업 특화", "MCP 에이전트 연동"

### 🎯 추천 우선순위
```
1순위  ② Meetily MCP 서버    — 기술난이도 대비 임팩트 최대, 7주면 출시
2순위  ④ 구축 대행 서비스     — 당장 현금흐름, 초기투자 0원
3순위  ① 한국형 Meetily       — React 스킬 100% 활용, 2~4주
4순위  ⑤ 유튜브 + 강의        — 블루오션, 복리 효과
5순위  ⑥ 템플릿 마켓          — 마진 95%, 크로스셀
6순위  ③ 산업 특화판          — 최고 수익이지만 영업 사이클 길다
```

---

*이 문서는 `bmshin94/meetily` 저장소 v0.4.1(`be1c7de`) 코드 전수조사 결과를 정리한 것입니다.*
