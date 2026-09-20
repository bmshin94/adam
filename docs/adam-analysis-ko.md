# Adam 전수조사 & 활용 분석 (한국어 정리)

> 작성일: 2026-09-20
> 대상 레포지토리: **https://github.com/bmshin94/adam**
> 원본(업스트림): **https://github.com/sqliteai/adam** — ⭐ 123 / 🍴 17 / MIT License
> 관련 프로젝트: https://github.com/sqliteai/sqlite-vector · https://github.com/sqliteai/sqlite-memory
> 의존 서브모듈: [llama.cpp](https://github.com/ggerganov/llama.cpp) · [whisper.cpp](https://github.com/ggerganov/whisper.cpp) · [miniaudio](https://github.com/mackron/miniaudio) · [mbedtls](https://github.com/Mbed-TLS/mbedtls) · [curl](https://github.com/curl/curl)

---

## 목차

1. [Adam이란?](#1-adam이란)
2. [폴더 전수조사](#2-폴더-전수조사)
3. [핵심 동작 원리](#3-핵심-동작-원리)
4. [설치 및 사용법](#4-설치-및-사용법)
5. [플러그인? 스킬? MCP?](#5-플러그인-스킬-mcp)
6. [API 토큰 필요 여부](#6-api-토큰-필요-여부)
7. [왜 주목받는가](#7-왜-주목받는가)
8. [로컬 에이전트 구축 활용도](#8-로컬-에이전트-구축-활용도)
9. [React / PHP 연동 방법](#9-react--php-연동-방법)
10. [수익화 아이디어](#10-수익화-아이디어)
11. [로드맵 & 리스크](#11-로드맵--리스크)

---

## 1. Adam이란?

**C언어로 작성된 임베디드 AI 에이전트 라이브러리.**

`#include "adam.h"` 한 줄로 프로그램 안에 완전한 에이전트 루프(툴 콜링, 메모리, 세션, 음성, 스트리밍, 구조화 출력)를 내장한다.

- **클라우드 LLM**: Anthropic, OpenAI, Google Gemini, Groq, Together, xAI
- **로컬 LLM**: llama.cpp를 통한 GGUF 모델 (API 키 불필요)
- **플랫폼**: macOS, Linux, Windows, iOS, Android, WASM(브라우저)
- **규모**: `src/` + `extensions/` 약 15,500줄 C 코드, 테스트 161개(ASan/UBSan)

### 한 줄 비유
> AI 에이전트 기능을 전부 C 라이브러리 하나로 압축해서, 아무 프로그램에나 붙일 수 있게 만든 레고 블록.

---

## 2. 폴더 전수조사

### `src/` — 라이브러리 본체 (심장부)

| 파일 | 크기 | 역할 |
|---|---|---|
| `adam.c` | 77KB | 메인 엔진. `adam_run()` 에이전트 루프 |
| `adam.h` | 55KB | 공개 API 전체 (이것만 include) |
| `adam_json.c` | 51KB | 3사(Anthropic/OpenAI/Gemini) JSON 포맷 직접 변환 |
| `adam_tools.c` | 34KB | 내장 툴 13종 구현 |
| `adam_stream.c` | 31KB | SSE 실시간 토큰 스트리밍 |
| `adam_local.c` | 30KB | llama.cpp 연동 (로컬 GGUF 추론) |
| `adam_research.c` | 20KB | 자율 리서치 모드 (다회 반복 + 보고서 합성) |
| `adam_memory.c` | 19KB | 장기기억 (BM25 + 벡터 하이브리드 검색) |
| `adam_voice.c` | 19KB | 음성 파이프라인 (STT → 에이전트 → TTS) |
| `adam_session.c` | 17KB | 세션 저장/복원 (UUIDv7 키) |
| `adam_evolve.c` | 14KB | 진화 루프 (자가개선: 실행→채점→전략수정→재시도) |
| `adam_http.c` | 14KB | HTTP 추상화 |
| `adam_audio.c` | 13KB | 오디오 I/O (miniaudio) |
| `adam_net_apple.m` | 12KB | macOS/iOS NSURLSession 백엔드 |
| `adam_net_curl.c` | 8KB | Linux libcurl 백엔드 |
| `adam_stt_local.c` | 8KB | whisper.cpp 로컬 STT |
| `adam_cache.c` | 7KB | LRU 응답 캐시 (모델+히스토리 해시 키) |
| `arena.c` | 6KB | 아레나 할당자 (반복마다 통째 해제 = 누수 0 설계) |
| `jsmn.h` | 12KB | 경량 JSON 토크나이저 |
| `main.c` | 47KB | CLI 채팅 앱 (슬래시 명령어) |

### `extensions/` — DB 확장 (킬러 기능)

| 경로 | 설명 |
|---|---|
| `extensions/sqlite/` | SQLite 확장. `.load adam` |
| `extensions/postgres/` | PostgreSQL 확장. `CREATE EXTENSION adam;` (Dockerfile 포함) |
| `extensions/src/` | 공용 구현 (`adam_ext_chat.c`, `adam_ext_config.c`, `adam_ext_schema.c`, `adam_ext_session.c`, `adam_ext_ctx.c`) |

**제공 SQL 함수**

| 함수 | 설명 |
|---|---|
| `adam_config(key, val)` | provider / api_key / model 설정 (영구 저장) |
| `adam(msg)` | 무상태 1회성 채팅 |
| `adam_ask(msg)` | **SQL 인식 에이전트** — 스키마 읽고, 쿼리 실행, 멀티턴 |
| `adam_sql(question)` | 자연어 → SQL 생성 (실행 안 함) |
| `adam_create_session()` / `adam_get_session()` / `adam_clear_session()` | 세션 관리 |

```sql
SELECT adam_ask('How many users signed up last month?');
-- → "47 users signed up last month."
```

### `packages/` — 언어 바인딩

| 경로 | 대상 |
|---|---|
| `packages/swift/` | Swift Package (iOS 14+, macOS 11+) — `Package.swift`가 릴리스 xcframework 참조 |
| `packages/android/` | Gradle 빌드 → AAR |
| `packages/flutter/` | pub.dev 패키지 `sqlite_adam` (Dart 3.10+ / Flutter 3.38+) |

### `examples/` — 예제 22개

`simple-conversation`, `tool-calling`, `local-model`, `local-vision`, `google-gemini`, `image-generation`, `sqlite-query`, `structured-json`, `memory`, `sessions`, `streaming`, `voice`, `multi-agent`, `evolution`, `research`, `filesystem-sandbox`, `guardrails`, `response-cache`, `thread-pool`, `telegram`, `wasm-chat`, `full-agent`

### 기타

| 경로 | 설명 |
|---|---|
| `modules/` | git 서브모듈 8개 (llama.cpp, whisper.cpp, miniaudio, sqlite, sqlite-vector, sqlite-memory, mbedtls, curl) |
| `test/` | `test_adam.c`, `test_memory.c`, `test_vision.c`, `test_chat.c`, `test_live.c`, 음성 테스트 2종 + 테스트 이미지 |
| `Makefile` | 31KB. 전 플랫폼 빌드 (`all` `deps` `test` `chat` `talk` `vision` `wasm` `extension` `xcframework` `aar`) |
| `API.md` | 26KB — 함수/타입/콜백 전체 레퍼런스 |
| `ARCHITECTURE.md` | 37KB — 내부 동작 원리 (에이전트 루프, 메모리 모델, SQLite 스키마, JSON 와이어 포맷, 아레나 할당자 등) |
| `.github/workflows/` | `main.yml` (전 플랫폼 빌드/테스트/릴리스), `flutter-package.yml` |

---

## 3. 핵심 동작 원리

### 에이전트 루프 (`adam_run()`)

```
시스템 프롬프트 조립 (정체성 + 지시문 + 부트스트랩 파일 + 기억 + 날짜)
  → 가드레일 체크 (on_before_send)
  → 응답 캐시 확인
  → LLM 디스패치 (mock | 로컬 llama.cpp | 원격 HTTP)
  → 가드레일 체크 (on_after_receive)
  → tool_calls 있으면? → 툴 실행 → 결과 append → 루프 반복
  → 텍스트면? → 최종 응답 반환 → 세션 자동저장 → 기억 추출
```

### 내장 툴 13종

| 툴 | 함수 | 인자 |
|---|---|---|
| 웹 fetch | `adam_tool_web_fetch` | `{"url","method"}` |
| 웹 검색 | `adam_tool_web_search` | `{"query","count"}` (BRAVE_API_KEY 필요) |
| HTTP POST | `adam_tool_http_post` | `{"url","body"}` |
| 파일 읽기 | `adam_tool_file_read` | `{"path","max_bytes"}` |
| 파일 쓰기 | `adam_tool_file_write` | `{"path","content","append"}` |
| 디렉터리 목록 | `adam_tool_list_directory` | `{"path"}` |
| 셸 실행 | `adam_tool_shell_exec` | `{"command","timeout"}` |
| 계산기 | `adam_tool_calculator` | `{"expression"}` |
| SQL 쿼리 | `adam_tool_sql_query` | `{"sql"}` |
| 기억 검색 | `adam_tool_memory_search` | `{"query","limit"}` |
| 기억 추가 | `adam_tool_memory_add` | `{"text","context","source"}` |
| 리서치 | `adam_tool_research` | `{"question","instructions","max_iterations"}` |
| 서브 에이전트 | `adam_tool_agent` | `{"message"}` |

> 파일/디렉터리/셸 툴은 `adam_settings_allow_dir()`로 허용한 경로에서만 동작 (샌드박스).

### 기능 게이트 (컴파일 타임 축소)

`ADAM_NO_CURL` · `ADAM_NO_LOCAL` · `ADAM_NO_PTHREADS` · `ADAM_NO_SQLITE` · `ADAM_NO_VOICE` · `ADAM_NO_FILESYSTEM` · `ADAM_NO_SHELL`

---

## 4. 설치 및 사용법

### A. 소스 빌드

```bash
git clone --recursive https://github.com/sqliteai/adam.git   # --recursive 필수
cd adam
make deps        # llama.cpp / whisper.cpp (+ Linux: mbedtls, curl) — 10~30분
make all         # libadam.a + adam 실행파일
make test        # 161개 테스트 (ASan + UBSan)

make chat                              # 클라우드 API 대화
make chat GGUF=models/model.gguf       # 로컬 모델 대화
make talk                              # 음성 에이전트
make talk LOCAL=1                      # 완전 로컬 음성 에이전트
make vision GGUF=... MMPROJ=...        # 로컬 비전
```

필요: C 컴파일러(clang/gcc), make, cmake, git. 디스크 여유 5~10GB 권장.

### B. C 코드에서 사용

```c
#include "adam.h"

int main(void) {
    adam_init();

    adam_settings_t *s = adam_create_settings();
    adam_settings_set_provider(s, ADAM_API_ANTHROPIC,
                               getenv("ANTHROPIC_API_KEY"),
                               "claude-sonnet-4-20250514");

    adam_history_t *h = adam_history_create();
    adam_run_result_t r = adam_run(s, h, "What is the capital of France?");
    printf("%s\n", r.final_response);

    adam_run_result_free(&r);
    adam_history_destroy(h);
    adam_settings_destroy(s);
    adam_cleanup();
}
```

### C. SQLite 확장 (가장 간편)

```bash
make extension   # → dist/adam.dylib | .so | .dll
```

```sql
.load adam
SELECT adam_config('provider', 'anthropic');
SELECT adam_config('api_key', 'sk-ant-...');
SELECT adam_ask('이번 주 신규 가입자 수는?');
```

### D. 모바일 / 웹

```bash
make xcframework   # iOS/macOS
make aar           # Android
make wasm          # adam.js + adam.wasm (브라우저)
```
```bash
dart pub add sqlite_adam   # Flutter
```

### E. CLI 슬래시 명령어

`/image` `/clear` `/history` `/stream` `/identity` `/instructions` `/tools` `/sandbox` `/cache` `/db` `/memory` `/session` `/sessions` `/json` `/research` `/evolve` `/tts` `/speak` `/talk` `/telegram` `/status` `/help` `/quit`

---

## 5. 플러그인? 스킬? MCP?

**셋 다 아님. Adam은 "C 라이브러리(SDK) + DB 확장"이다.**

| 구분 | 해당 여부 | 이유 |
|---|---|---|
| Claude Code 플러그인 | ❌ | 플러그인은 `.claude/plugins/` 기반 설정/JS |
| Skill | ❌ | 스킬은 `SKILL.md` 마크다운 지침서 |
| MCP 서버 | ❌ | MCP는 JSON-RPC로 툴을 "제공"하는 쪽. Adam은 툴을 "호출"하는 쪽 |
| 라이브러리 / SDK | ✅ | `libadam.a` 링크 |
| DB 확장 | ✅ | SQLite `.load` / PostgreSQL `CREATE EXTENSION` |
| CLI 도구 | ✅ (부가) | `adam` 실행파일 |

### 포지션

```
MCP 서버      = 에이전트에게 도구를 제공하는 쪽
Claude Code   = 에이전트를 사용하는 완성품 앱
Adam          = 에이전트를 만드는 재료 (라이브러리)
```

> Adam을 stdio JSON-RPC로 감싸면 MCP 서버로 만들 수 있음 (기본 제공 X → 수익화 아이디어 #6).

---

## 6. API 토큰 필요 여부

| 모드 | 토큰 | 비용 |
|---|---|---|
| 클라우드 (Anthropic/OpenAI/Gemini/Groq/xAI/Together) | 필요 | 종량제 |
| **로컬 GGUF (llama.cpp)** | **불필요** | **0원** |
| 웹 검색 툴 | `BRAVE_API_KEY` | 무료 티어 있음 |
| 로컬 STT/TTS (whisper.cpp + 시스템 TTS) | 불필요 | 0원 |
| Mock 모드 | 불필요 | 0원 |

### 키 설정 4가지

```bash
export ANTHROPIC_API_KEY=sk-ant-...        # 1) 환경변수
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env # 2) .env (예제들이 사용)
```
```c
adam_settings_set_provider(s, ADAM_API_ANTHROPIC, "sk-ant-...", "model"); // 3) 코드
```
```sql
SELECT adam_config('api_key', 'sk-ant-...');  -- 4) SQL (DB에 영구 저장)
```

> ⚠️ `adam_config`로 넣은 키는 DB 파일에 저장된다. 파일 권한 확인 필수, git 커밋 금지.

### 내장 모델 가격표 (API.md 기준)

| 모델 | 컨텍스트 | 입력 $/Mtok | 출력 $/Mtok |
|---|---|---|---|
| claude-opus-4 | 200K | 15.0 | 75.0 |
| claude-sonnet-4 | 200K | 3.0 | 15.0 |
| claude-haiku-4 | 200K | 0.8 | 4.0 |
| gpt-4o | 128K | 2.5 | 10.0 |
| gpt-4o-mini | 128K | 0.15 | 0.60 |
| gemini-2.5-flash | 1M | 0.15 | 0.60 |
| gemini-1.5-flash | 1M | 0.075 | 0.30 |

비용 절감 수단: 내장 LRU 응답 캐시, 토큰 추정 API, 히스토리 요약(LLM 압축).

---

## 7. 왜 주목받는가

**팩트: ⭐123 / 🍴17** — 초대형 인기는 아니고, 니치에서 제대로 주목받는 단계.

1. **C로 만든 AI 에이전트**라는 희소성 (임베디드/게임엔진/모바일에서 사실상 유일한 선택지)
2. **에이전트를 SQL 안에 넣는 발상** — `SELECT adam_ask(...)`. DB 자체가 에이전트가 됨
3. **SQLite AI 팀 제작** (sqlite-vector, sqlite-memory 제작사) — 신뢰도
4. **기능 밀도** — 툴콜링 + 메모리 + 세션 + 음성 + 비전 + 스트리밍 + 멀티에이전트 + 리서치 + 진화루프 + 가드레일 + 스레드풀 + 샌드박스를 15,000줄에
5. **크로스플랫폼 범위** — macOS/Linux/Windows/iOS/Android/WASM
6. **로컬·클라우드 동일 인터페이스** — 코드 수정 없이 전환
7. **문서 퀄리티** — API.md 26KB + ARCHITECTURE.md 37KB, 내부 알고리즘까지 공개
8. **엔지니어링 품질** — 161 테스트 + ASan/UBSan, 아레나 할당자, 기능 게이트

---

## 8. 로컬 에이전트 구축 활용도

**결론: 최상위급으로 적합.**

| 항목 | Adam | Python 계열 프레임워크 |
|---|---|---|
| 런타임 | 불필요 (네이티브) | Python/Node 필요 |
| 메모리 | 수 MB | 수백 MB |
| 시작 속도 | 즉시 | 수 초 |
| 서버 | 불필요 | 대체로 필요 |
| 오프라인 | 완전 가능 | 제한적 |
| 모바일 내장 | 가능 | 사실상 불가 |
| 브라우저 | WASM 가능 | 불가 |

### 완전 로컬 스택

```
[마이크] → whisper.cpp (로컬 STT)
          → llama.cpp + GGUF (로컬 LLM)
          → Adam 에이전트 루프 + 내장 툴
          → SQLite 장기기억 (BM25 + 벡터)
          → 시스템 TTS → [스피커]

비용 0원 / 인터넷 불필요 / 데이터 100% 온디바이스
```
```bash
make talk LOCAL=1
```

### 안전장치
- 파일시스템 샌드박스: `adam_settings_allow_dir()`
- 가드레일: `on_before_send` / `on_after_receive` 콜백
- 기능 게이트: `ADAM_NO_SHELL` 정의 시 셸 툴 자체가 컴파일에서 제외

### 학습 가치
`adam.c`의 `adam_run()`과 `ARCHITECTURE.md`가 "에이전트 내부 구조" 교과서 역할을 함.

### 단점
- 빌드 무거움 (`make deps` 10~30분, 디스크 수 GB)
- C 수동 메모리 관리
- 로컬 모델은 GPU 없으면 느림
- 생태계가 작아 막히면 소스를 직접 읽어야 함

---

## 9. React / PHP 연동 방법

### React

| 방법 | 설명 | 평가 |
|---|---|---|
| **WASM** | `make wasm` → `adam.js` import. 레포에 `examples/wasm-chat/` 예제 있음 | 브라우저 단독 가능. **API 키 노출 위험 → 프록시 필수** |
| **Node + FFI** | `ffi-napi` 또는 N-API 애드온으로 `libadam` 호출 → React는 `fetch('/api/chat')` | 키 안전, 성능 good |
| **Node + SQLite 확장** ⭐ | `better-sqlite3`의 `db.loadExtension('./adam')` → `SELECT adam_ask(?)` | **가장 간단. C 코드 0줄** |

```js
// 추천: better-sqlite3 + SQLite 확장
const Database = require('better-sqlite3');
const db = new Database('app.db');
db.loadExtension('./adam');
db.prepare("SELECT adam_config('api_key', ?)").run(process.env.ANTHROPIC_API_KEY);
const answer = db.prepare("SELECT adam_ask(?) AS r").get(question).r;
```

### PHP

| 방법 | 설명 | 평가 |
|---|---|---|
| **SQLite3::loadExtension** ⭐ | `$db->loadExtension('adam.so')` → `SELECT adam_ask(:q)` | 가장 쉬움. `php.ini`의 `sqlite3.extension_dir` 설정 필요 |
| **PostgreSQL 확장 + PDO** ⭐ | DB에 `CREATE EXTENSION adam;` 후 `SELECT adam_ask(?)` | 서버 앱/워드프레스에 최적 |
| **PHP FFI** | `FFI::cdef`로 `libadam.so` 직접 호출 | CLI엔 OK, FPM 환경에선 비권장 |

```php
<?php
$db = new SQLite3('app.db');
$db->enableExceptions(true);
$db->loadExtension('adam.so');

$db->querySingle("SELECT adam_config('provider','anthropic')");
$db->querySingle("SELECT adam_config('api_key','" . getenv('ANTHROPIC_API_KEY') . "')");

$stmt = $db->prepare("SELECT adam_ask(:q) AS answer");
$stmt->bindValue(':q', $_POST['question'], SQLITE3_TEXT);
echo $stmt->execute()->fetchArray()['answer'];
```

```php
<?php // PostgreSQL 버전
$pdo  = new PDO('pgsql:host=localhost;dbname=mydb', $user, $pass);
$stmt = $pdo->prepare("SELECT adam_ask(?)");
$stmt->execute([$question]);
echo $stmt->fetchColumn();
```

> **핵심**: SQLite/PG 확장 방식이면 `adam.so` 하나로 React·PHP·Python·Ruby 전부 커버된다.

---

## 10. 수익화 아이디어

### 티어 1 — 진입장벽 낮음 (즉시 착수 가능)

#### ① 자연어 DB 분석 SaaS ⭐ 1순위 추천
- **타겟**: 개발자 없는 소상공인, 마케터, 쇼핑몰 운영자
- **제품**: DB 연결 → 한국어 질문 → 답변 + 차트 (`adam_ask`가 스키마 읽고 SQL 실행)
- **가격**: Free(월 50쿼리) / ₩29,000(월 1,000쿼리) / ₩99,000(무제한 + API)
- **스택**: PHP or Node + SQLite/PG 확장 + React 대시보드
- **MVP**: 2~3주
- **차별점**: 경쟁사는 "SQL 생성"까지, Adam은 **실행 + 해석 + 멀티턴**까지
- **리스크**: 읽기 전용 DB 계정 강제 필수

#### ② 완전 오프라인 AI 앱 (프라이버시 특화)
- **타겟**: 의료/법률/금융 종사자, 기자, 프라이버시 민감층
- **제품**: 인터넷 없이 100% 온디바이스로 도는 AI 노트/상담 앱
- **가격**: 앱 1회 구매 ₩15,000 (구독 아님) 또는 모델팩 인앱결제
- **스택**: Flutter (`sqlite_adam` pub.dev 패키지 활용)
- **MVP**: 4~6주
- **차별점**: "데이터가 기기를 떠나지 않음" + **서버비 0원 → 마진 90%+**
- **리스크**: 앱 용량(모델 2~4GB), 구형 기기 성능

#### ③ 텔레그램/메신저 봇 SaaS
- 레포에 `examples/telegram/` 완성 예제 존재 (텍스트 + 음성 + 이미지 + 툴 + 메모리)
- **가격**: 봇 1개당 월 ₩19,000 / 기업형 ₩99,000
- **MVP**: 1~2주
- **차별점**: 장기기억 내장 → 단골 고객 문맥 기억

### 티어 2 — 기술력 필요, 마진 큼

#### ④ 임베디드 / 키오스크 AI 솔루션
- **타겟**: 무인매장, 키오스크 제조사, 스마트팩토리, 로봇 스타트업
- C 라이브러리라 라즈베리파이·산업용 보드에 그대로 탑재 — **Adam만의 영역**
- **가격**: SI 프로젝트 ₩3,000만~1억 / 유지보수 월 ₩200만
- **차별점**: 경쟁사는 인터넷+서버 필요, 이쪽은 오프라인 단독 동작
- **리스크**: 영업 사이클 3~6개월

#### ⑤ 언어 바인딩 제작 & 유지보수
- 현재 Swift / Android / Flutter만 존재 → **빈 자리가 기회**
- 미개척: **PHP 확장**, Python 휠, Node N-API, Ruby gem, .NET, Rust crate, Go cgo
- **수익 모델**: GitHub Sponsors / 듀얼 라이선스(기업용 ₩100만/년) / SLA 지원 월 ₩50만 / 통합 컨설팅 건당 ₩500만~
- **블루오션**: PHP 확장 — 워드프레스 생태계(웹의 약 40%)에 "로컬 AI 플러그인"은 거의 없음

#### ⑥ MCP 서버 래퍼 상품화
- Adam을 stdio JSON-RPC로 감싸 Claude Code / Cursor에서 사용
- **가격**: 개인 무료 / 팀 $20/인/월 / 엔터프라이즈 온프레미스
- **차별점**: 기존 MCP 서버 대부분 Node 기반 → 네이티브라 빠르고 가벼움
- **MVP**: 2~3주

### 티어 3 — 콘텐츠/교육 (자본 0원)

#### ⑦ "에이전트 내부구조 완전정복" 교육
`ARCHITECTURE.md` 37KB를 커리큘럼 소스로 사용

1. 에이전트 루프란? (`adam_run` 분해)
2. 툴 콜링 구현 (JSON 스키마 → 실행 → 결과 주입)
3. 메모리 시스템 (BM25 + 벡터 하이브리드)
4. 아레나 할당자로 메모리 누수 0 만들기
5. 3사 API 포맷 통합
6. 진화 루프 = 자가개선 AI
7. C에서 WASM까지 크로스플랫폼 빌드

- 온라인 강의 ₩99,000 × 500명 ≈ ₩4,950만 / 전자책 ₩25,000 / 유튜브 / 기업교육 1일 ₩300만

#### ⑧ 파생 오픈소스로 인지도 구축
`adam-php`, `adam-mcp`, `adam-docker`, `adam-ui`(React 컴포넌트), `adam-ko`(한국어 프리셋)
→ 스타 확보 → 컨설팅/채용/투자 연결

---

## 11. 로드맵 & 리스크

### 추천 로드맵

| 기간 | 할 일 |
|---|---|
| **1~2개월차** | `make all` 성공 + 예제 22개 실행 / SQLite 확장 빌드 후 PHP·Node 호출 테스트 / 아이디어 3개로 좁히고 지인 인터뷰 10명 |
| **3~4개월차** | MVP — **① 자연어 DB 분석 SaaS** 또는 **③ 텔레그램 봇** (기존 React/PHP 스택으로 가능, `adam_ask`가 핵심 로직 담당) |
| **5~6개월차** | 결제 붙이고 유료 전환 / 개발 과정 콘텐츠화(⑦) / 반응 좋으면 임베디드 B2B(④)로 확장 |

### 리스크 & 대응

| 리스크 | 대응 |
|---|---|
| API 키 비용 폭탄 | 사용량 제한 + 내장 응답 캐시 활성화 + 저가 모델(Gemini Flash 등) |
| DB 파괴 위험 | **읽기 전용 계정 강제** + 가드레일 콜백 필수 |
| 라이선스 | Adam은 MIT(상업 이용 가능). 단 llama.cpp / whisper.cpp 등 서브모듈 라이선스 별도 확인 |
| 프로젝트 성숙도 | ⭐123, 초기 단계 → 소스 직접 수정할 각오 필요 |
| 개인정보보호법 | 로컬 모드가 오히려 세일즈 포인트 |
| 빌드 난이도 | `--recursive` 클론 필수, 디스크 5~10GB, CI 캐시 활용 |

---

## 부록: 참고 링크

| 항목 | URL |
|---|---|
| 이 레포 (포크) | https://github.com/bmshin94/adam |
| 원본 Adam | https://github.com/sqliteai/adam |
| API 레퍼런스 | https://github.com/sqliteai/adam/blob/main/API.md |
| 아키텍처 문서 | https://github.com/sqliteai/adam/blob/main/ARCHITECTURE.md |
| SQLite 확장 | https://github.com/sqliteai/adam/tree/main/extensions/sqlite |
| PostgreSQL 확장 | https://github.com/sqliteai/adam/tree/main/extensions/postgres |
| 예제 모음 | https://github.com/sqliteai/adam/tree/main/examples |
| Flutter 패키지 | https://pub.dev/packages/sqlite_adam |
| sqlite-vector | https://github.com/sqliteai/sqlite-vector |
| sqlite-memory | https://github.com/sqliteai/sqlite-memory |
| llama.cpp | https://github.com/ggerganov/llama.cpp |
| whisper.cpp | https://github.com/ggerganov/whisper.cpp |
