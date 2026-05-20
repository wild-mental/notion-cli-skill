![AI Skills for Everyone](author/wildmental-bjpark.png)

# notion-cli 
> Skill for Cursor, Claude, Codex agents

**Notion CLI(`ntn`)로 지식베이스를 다룰 때, Agent가 같은 함정을 반복하지 않도록 가르치는 Agent Skill입니다. Cursor, Claude Code, Codex 모두 지원합니다.**

Notion 페이지를 로컬 마크다운으로 내려받고, 이미지를 풀고, 내부 링크를 연결하는 작업은 문서상으로는 단순해 보이지만 실제로는 **페이지 ID 형식, Notion 전용 마크다운, `file://` 이미지 참조, API hang** 같은 함정이 연속으로 기다립니다. 이 스킬은 그 시행착오를 **미리 문서화하고, Agent 행동 규칙으로 고정**합니다.

---

## 이 스킬이 해결하는 것

### 1. Agent가 Notion을 안전하게 다루게 한다

- 변경 전 **반드시 조회** (`ntn pages get`, `ntn doctor`)
- 원본 보존, append 우선, destructive action 금지
- 불확실하면 `-help` / 공식 문서 확인

### 2. 로컬 export의 정석 워크플로를 제공한다

```
ntn doctor → page_id 정규화 → BFS로 내부 페이지 탐색
→ ntn pages get → 이미지 로컬화 → <page> 상대 링크 변환 → 검증
```

- `ntn pages get` + `getSignedFileUrls` 조합 (Public API block children 회피)
- `assets/` 디렉터리 규칙, slug 충돌 처리, 순환 링크 방지
- 완료 후 **페이지 수 · 이미지 수 · 링크 변환 목록** 보고

### 3. Notion ↔ Local LLM 아키텍처를 명확히 한다

```
Notion (source of truth)
    ↕  Notion CLI (ntn)
    ↕  Local LLM Agent
    ↕  GitHub / Local Runtime / External Services
```

Notion은 협업·지식 저장소, LLM은 분석·생성 엔진, CLI는 연결 계층 — 역할을 섞지 않도록 경계를 정의합니다.

---

## 왜 skill 이 필요한가

| 흔한 시도 | 기대 | 실제 |
|-----------|------|------|
| URL 슬러그로 `ntn pages get` | 페이지 조회 | `400 validation_error` |
| `ntn pages get` 결과를 GitHub 미리보기 | 표준 마크다운 | `<columns>`, `<page>` 등 Notion 확장 문법 |
| `ntn api GET v1/blocks/.../children` | 블록·이미지 수집 | 장시간 hang / timeout |
| `file://` JSON의 `source` 필드 그대로 API 호출 | signed URL | 필드명은 **`url`** 이어야 함 |
| 한글 파일명 이미지 다운로드 | 로컬 저장 | `UnicodeEncodeError` |
| `~/.config/notion/`에서 토큰 찾기 | 인증 정보 | macOS는 Keychain **`notion-cli`** |

이 스킬은 위 패턴을 **"알아서 피한다"** 수준이 아니라, **조회 우선 → 안전한 export 경로 → 검증 체크리스트**까지 포함한 운영 규칙으로 정리해 둡니다.

---


## 빠른 시작

### 사전 요구사항

- [Notion CLI (`ntn`)](https://ntn.dev) 설치 및 로그인
- Cursor, Claude Code, 또는 Codex

```bash
curl -fsSL https://ntn.dev | bash
ntn login
ntn doctor   # Token valid, Default workspace 통과 확인
```

### 스킬 설치

**프로젝트에 포함된 경우** — 이 저장소를 clone하면 아래 경로에 스킬이 함께 옵니다.

| 도구 | 프로젝트 경로 | 개인 경로 |
|------|---------------|-----------|
| Cursor | `.cursor/skills/notion-cli/` | `~/.cursor/skills/notion-cli/` |
| Claude Code | `.claude/skills/notion-cli/` | `~/.claude/skills/notion-cli/` |
| Codex | `.agents/skills/notion-cli/` | `~/.agents/skills/notion-cli/` |

**개인 스킬로 쓰는 경우:**

```bash
# Cursor
mkdir -p ~/.cursor/skills/notion-cli
cp .cursor/skills/notion-cli/SKILL.md ~/.cursor/skills/notion-cli/

# Claude Code
mkdir -p ~/.claude/skills/notion-cli
cp .claude/skills/notion-cli/SKILL.md ~/.claude/skills/notion-cli/

# Codex
mkdir -p ~/.agents/skills/notion-cli
cp .agents/skills/notion-cli/SKILL.md ~/.agents/skills/notion-cli/
```

- **Cursor**: 설치 후 **Reload Window**를 한 번 실행하세요.
- **Claude Code**: 스킬 수정은 세션 중 live 반영됩니다. 세션 시작 후 새로 만든 top-level `.claude/skills/`는 재시작이 필요할 수 있습니다.
- **Codex**: 스킬 변경이 반영되지 않으면 Codex를 재시작하세요.

### 사용 방법

Notion 관련 작업을 요청하면 Agent가 스킬을 자동으로 적용합니다.

| 도구 | 수동 호출 |
|------|-----------|
| Cursor | `/notion-cli` |
| Claude Code | `/notion-cli` |
| Codex | `/skills` 또는 `$notion-cli` |

Claude Code 버전은 스킬 로드 시 `ntn doctor` 결과를 자동으로 주입합니다. Codex 버전은 작업 시작 시 `ntn doctor` 실행을 지시합니다.

**적용되는 요청 예시:**

- "이 Notion 페이지를 마크다운으로 다운로드해줘"
- "이미지 포함해서 로컬에 저장하고 내부 링크도 연결해줘"
- "Notion 페이지 내용을 읽고 요약해줘"
- "ntn으로 페이지 상태 확인해줘"

---

## 핵심 함정 요약

스킬 본문에 상세 규칙이 있으며, 아래는 가장 자주 막히는 지점입니다.

### 페이지 ID는 32자 hex만

```bash
# ✅
ntn pages get 305d03212bd4807d9be2f771c42a8cb6

# ❌ URL 슬러그 포함
ntn pages get AI-IT-305d03212bd4807d9be2f771c42a8cb6
```

### `ntn pages get` 출력 ≠ GFM

Notion 확장 마크다운(`<columns>`, `<page url>`, `:icon:`)은 **정상 export**입니다. GFM 변환은 웹/GitHub용으로 명시적으로 요청할 때만 합니다.

### 이미지는 `getSignedFileUrls` + `url` 필드

`file://` 뒤 JSON의 `source` 값을 API 요청 시 **`url`** 키로 전달해야 합니다. 한글 파일명은 URL 인코딩 후 요청합니다.

### macOS 토큰 위치

```bash
security find-generic-password -s "notion-cli" -w
```

---

## 권장 로컬 디렉터리 구조

Notion 페이지를 재귀 다운로드할 때 스킬이 따르는 레이아웃:

```
<project-root>/
├── <root-page>.md              # 루트 페이지
├── assets/                     # 루트 이미지
├── pages/
│   ├── <slug>.md               # 내부 링크된 하위 페이지
│   └── assets/<slug>/          # 하위 페이지 이미지
└── scripts/
    └── download_notion_page.py # 재귀 다운로드 스크립트 (선택)
```

---

## 완료 검증 체크리스트

로컬 export 작업 후 Agent는 아래를 확인합니다.

- [ ] `ntn doctor` 통과
- [ ] 모든 `<page url>` → `[text](relative.md)` 변환
- [ ] `file://` 잔여 0건
- [ ] 각 `.md`의 이미지 경로와 실제 파일 일치
- [ ] 재귀 탐색 페이지 수 = 저장된 `.md` 수

---

## 스킬 구성

```
.cursor/skills/notion-cli/SKILL.md   # Cursor용
.claude/skills/notion-cli/SKILL.md   # Claude Code용 (ntn doctor 자동 주입)
.agents/skills/notion-cli/SKILL.md   # Codex용
```

| 섹션 | 내용 |
|------|------|
| 기본 원칙 | 조회 우선, 원본 보존, 최소 권한 |
| 시행착오 및 알려진 함정 | 7가지 실무 함정과 해결책 |
| 로컬 다운로드 규칙 | BFS 탐색, 이미지 로컬화, 상대 링크 |
| API 스니펫 | `getSignedFileUrls`, 정규식, page_id 정규화 |

---

## 이런 분께 추천합니다

- Notion을 **팀 지식베이스**로 쓰고, Cursor, Claude Code, 또는 Codex Agent로 읽기·분석·갱신하려는 분
- `ntn pages get` 결과가 "깨진 마크다운"처럼 보여 **GFM 변환을 억지로 시도**하다 시간을 쓴 분
- 이미지·내부 링크 때문에 **로컬 export를 포기**했거나, 매번 처음부터 디버깅하는 분
- Agent에게 Notion 작업을 맡기되 **원본 훼손·잘못된 API 경로**를 피하고 싶은 분

---

## 참고

- [Notion CLI (`ntn`)](https://ntn.dev)
- Cursor Agent Skills: [Creating Skills](https://docs.cursor.com)
- Claude Code Skills: [Extend Claude with skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- Codex Agent Skills: [Agent Skills](https://developers.openai.com/codex/skills/)

---

## For LLMs

<!-- machine-oriented; skip prose formatting -->

```
skill=notion-cli
repo=notion-cli-skill
license=MIT
purpose=ntn/Notion CLI ops without repeated pitfalls: read-first, local markdown export, file:// images, recursive <page url> wiring

prereq:
  curl -fsSL https://ntn.dev | bash
  ntn login
  ntn doctor  # require Token valid + Default workspace

skill_paths:
  cursor.project=.cursor/skills/notion-cli/SKILL.md
  cursor.user=~/.cursor/skills/notion-cli/SKILL.md
  claude.project=.claude/skills/notion-cli/SKILL.md
  claude.user=~/.claude/skills/notion-cli/SKILL.md
  codex.project=.agents/skills/notion-cli/SKILL.md
  codex.user=~/.agents/skills/notion-cli/SKILL.md

install.repo=clone repo; skills ship in paths above
install.user.cursor=mkdir -p ~/.cursor/skills/notion-cli && cp .cursor/skills/notion-cli/SKILL.md ~/.cursor/skills/notion-cli/
install.user.claude=mkdir -p ~/.claude/skills/notion-cli && cp .claude/skills/notion-cli/SKILL.md ~/.claude/skills/notion-cli/
install.user.codex=mkdir -p ~/.agents/skills/notion-cli && cp .agents/skills/notion-cli/SKILL.md ~/.agents/skills/notion-cli/

invoke.cursor=/notion-cli
invoke.claude=/notion-cli
invoke.codex=/skills|$notion-cli

post_install.cursor=Reload Window
post_install.claude=live reload; restart if new top-level .claude/skills/ after session start
post_install.codex=restart if skill not detected

must_read=SKILL.md  # full rules, traps, snippets
workflow=ntn doctor > normalize_page_id(32hex) > BFS(<page url>) > ntn pages get > localize_images(getSignedFileUrls url=source) > replace_page_links(relative.md) > verify
rules=read-before-write; preserve source; append-over-overwrite; page_id=32hex-only; no ntn api GET block-children for download; Notion MD != GFM unless asked; macOS token=keychain service notion-cli
traps=slug-in-page-id=>400; source-field=>use url in getSignedFileUrls; korean-filename=>urllib.parse.quote path
verify=ntn doctor ok; no file:// left; all <page url>=>[text](relative.md); image paths exist; page count matches
```

---

## 라이선스

[MIT License](LICENSE)
