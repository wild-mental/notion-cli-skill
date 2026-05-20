---
name: notion-cli
description: Operates on Notion pages and documents via Notion CLI (ntn). Covers read-first workflows, local markdown export with images, recursive internal-page download, and relative link wiring. Use when the user mentions ntn, Notion CLI, downloading Notion pages to markdown, file:// images, or internal page links. Do not use for generic Notion API work without ntn.
---

# Notion CLI + Local LLM Operating Rules

## 역할

Notion 워크스페이스의 페이지·문서·데이터베이스를 사람이 쓰는 협업 공간으로 두고,
로컬 LLM Agent는 이를 읽고 분석하고 갱신한다.

## 시작 시 확인

작업 시작 전 `ntn doctor`를 실행해 인증·워크스페이스 상태를 확인한다. 실패 시 `curl -fsSL https://ntn.dev | bash && ntn login` 안내.

---

# 기본 원칙

1. 먼저 조회하고 이후 변경한다.
2. 원본 문서는 보존한다.
3. 덮어쓰기보다 추가/확장 방식을 우선한다.
4. 변경 결과에는 항상 요약을 남긴다.
5. 불확실한 명령은 `-help` 또는 공식 문서를 확인한다.
6. 자동화보다 구조적 일관성을 우선한다.
7. 민감 정보와 권한 범위를 최소화한다.

---

# 환경 준비

## 설치

```bash
curl -fsSL https://ntn.dev | bash
```

## 로그인

```bash
ntn login
```

## 확인

```bash
ntn --version
ntn --help
ntn doctor
```

`ntn doctor`에서 **Token valid** 와 **Default workspace** 가 통과해야 한다.

---

# 기본 작업 흐름

```
사용자 요청
↓
Notion 조회 (ntn pages get / doctor)
↓
로컬 분석 및 처리
↓
결과 생성
↓
필요 시 Notion 업데이트
↓
변경 요약 기록
```

---

# 작업 규칙

## 조회 우선

변경 전 반드시 현재 상태를 확인한다.

```bash
ntn pages get <page-id>          # 마크다운 조회 (다운로드·분석)
ntn pages get <page-id> --json     # truncated/unknown_block_ids 확인
ntn doctor                         # 인증·워크스페이스 상태
ntn pages --help                   # pages 하위 명령 확인
```

`ntn api`는 엔드포인트 탐색용으로만 우선 사용한다. 실제 다운로드·이미지 처리에는 아래 **로컬 다운로드 규칙**을 따른다.

---

## 구조 유지

- 기존 계층 구조를 존중한다.
- 문서 스타일을 임의 변경하지 않는다.
- 데이터베이스 스키마를 임의 수정하지 않는다.

---

## 안전한 수정

- destructive action 금지
- 대량 변경 금지
- 기존 내용 삭제 최소화
- append 방식 우선

---

## Agent 행동 규칙

- 짧은 실행 단위로 작업한다.
- 실패 시 상태를 먼저 점검한다.
- 추론보다 실제 조회 결과를 우선한다.
- 장기 상태 저장은 Notion 또는 외부 DB를 사용한다.

---

# Notion의 역할

```
Notion
= 문서·협업 워크스페이스
= 페이지·wiki·프로젝트 기록
= 사람 중심 작업 공간
```

```
Local LLM
= 분석 엔진
= 자동화 엔진
= 생성 엔진
```

```
Notion CLI
= 연결 계층
```

---

# 권장 구조

```
Notion
↕
Notion CLI
↕
Local LLM Agent
↕
GitHub / Local Runtime / External Services
```

---

# 제한사항

- Notion은 고성능 트랜잭션 DB가 아니다.
- 대규모 실시간 처리에는 부적합하다.
- 장시간 stateful execution에는 한계가 있다.
- 계산 집약 작업은 외부 런타임에서 수행한다.

---

# 시행착오 및 알려진 함정 (반드시 숙지)

## 1. 페이지 ID vs URL 슬러그

| 입력 | 결과 |
|------|------|
| `305d03212bd4807d9be2f771c42a8cb6` | ✅ 정상 |
| `AI-IT-305d03212bd4807d9be2f771c42a8cb6` | ❌ 400 validation_error |

URL의 `AI-IT-` 같은 **슬러그는 page_id가 아니다**. 항상 **32자 hex ID**만 사용한다.

```bash
# URL에서 ID 추출 패턴
# https://www.notion.so/AI-IT-305d03212bd4807d9be2f771c42a8cb6
#                              └────── 32자 hex ──────┘
```

## 2. `ntn pages get` 출력 ≠ 표준 마크다운

`ntn pages get`은 **Notion 확장 마크다운**을 반환한다. GFM 미리보기에서 "깨져 보이는 것"은 손상이 아니라 **정상 export**다.

| Notion 전용 문법 | 일반 MD 렌더러 |
|------------------|----------------|
| `<columns>`, `<column>` | ❌ |
| `<span color="blue">` | ❌ |
| `<table header-row="true">` | ❌ |
| `<page url="...">` | ❌ (로컬 변환 필요) |
| `:icon-name:` | ❌ |
| `<empty-block/>`, `<details>` | ❌ |

**웹/GitHub용**으로 쓸 때만 GFM 변환을 추가한다. **Notion round-trip** 목적이면 원문 보존을 우선한다.

## 3. `ntn api` GET은 다운로드에 부적합

실무에서 확인된 패턴:

| 명령 | 결과 |
|------|------|
| `ntn pages get <id>` | ✅ 빠르고 안정 |
| `ntn api -X GET v1/blocks/.../children` | ⚠️ 장시간 hang / timeout 빈번 |
| `ntn api -X GET v1/users/me` | ⚠️ 동일 |

**로컬 다운로드·이미지 처리에는 `ntn pages get` + Notion 내부 API(`getSignedFileUrls`) 조합**을 사용한다. Public API block children으로 이미지 URL을 얻는 경로는 사용하지 않는다.

## 4. 이미지는 `file://` JSON 참조

`ntn pages get`의 이미지:

```markdown
![](file://%7B%22source%22%3A%22attachment%3A...%22%2C%22permissionRecord%22%3A%7B...%7D%7D)
```

디코딩하면:

```json
{
  "source": "attachment:<uuid>:filename.png",
  "permissionRecord": {
    "table": "block",
    "id": "<block-uuid>",
    "spaceId": "<workspace-uuid>"
  }
}
```

**함정:** `getSignedFileUrls` 요청 시 필드명은 `source`가 아니라 **`url`** 이다.

```json
{
  "urls": [{
    "url": "<source 값>",
    "permissionRecord": { ... }
  }]
}
```

**함정:** signed URL에 한글 파일명이 포함되면 `urllib` 다운로드 시 `UnicodeEncodeError` 발생 → path를 `urllib.parse.quote`로 인코딩 후 요청.

## 5. macOS 인증 토큰 위치

`ntn login` 토큰은 Keychain **`notion-cli`** 서비스에 저장된다.

```bash
security find-generic-password -s "notion-cli" -w
```

`~/.config/notion/`에는 workspace 설정만 있고 **토큰 파일은 없다**.

## 6. 내부 페이지 링크 형식

Notion 내부 페이지 링크는 표준 `[text](url)` 이 아니다:

```markdown
<page url="https://www.notion.so/305d03212bd481129756d63ee2fd25b0">
  <span color="blue">링크 텍스트</span>
</page>
```

로컬 다운로드 시 **`[링크 텍스트](상대경로.md)`** 로 변환해야 한다. URL만 있고 로컬 파일이 없으면 Notion URL fallback.

## 7. Codex 스킬 발견

| 위치 | 경로 |
|------|------|
| 프로젝트 | `.agents/skills/notion-cli/SKILL.md` |
| 개인 | `~/.agents/skills/notion-cli/SKILL.md` |

수동 호출: `/skills` 또는 `$notion-cli`. 스킬 변경이 반영되지 않으면 Codex를 재시작한다.

---

# 로컬 다운로드 작업 규칙

기존 Notion 페이지를 로컬 마크다운 + 이미지 + 내부 링크로 내려받을 때 따른다.

## 언제 이 규칙을 적용하는가

- "노션 페이지를 마크다운으로 다운로드"
- "이미지 포함해서 로컬에 저장"
- "내부 페이지 재귀 다운로드 + 상대 링크"

## 표준 디렉터리 구조

```
<project-root>/
├── <root-page>.md              # 루트 페이지 (예: ai-it-profile.md)
├── assets/                     # 루트 페이지 이미지
├── pages/
│   ├── <slug>.md               # 내부 링크된 하위 페이지
│   └── assets/<slug>/          # 하위 페이지 이미지
└── scripts/
    └── download_notion_page.py # 재귀 다운로드 스크립트
```

**규칙:**

- 루트 페이지 1개 → 프로젝트 루트 `.md`
- 내부 페이지 → `pages/<slug>.md` (frontmatter `title` 기반 slugify)
- slug 충돌 시 `-2`, `-3` suffix
- 하위 페이지 이미지 prefix: `assets/<slug>/` (해당 `.md` 기준 상대경로)

## 다운로드 워크플로우 (필수 순서)

```
1. ntn doctor                          # 인증 확인
2. normalize_page_id(root_id)          # 32자 hex 검증
3. BFS discover_pages(root_id)         # <page url> 재귀 탐색
4. assign_paths()                      # page_id → 로컬 경로 매핑
5. 각 페이지:
   a. ntn pages get <id>               # raw markdown
   b. localize_images()                # file:// → assets/...
   c. replace_page_links()             # <page> → [text](relative.md)
   d. write file
6. 결과 요약 (페이지 수, 이미지 수, 링크 변환 수)
```

**조회 우선:** 3단계 탐색으로 전체 페이지 그래프를 먼저 파악한 뒤, 5단계에서 일괄 저장·링크 치환한다.

## 재귀 내부 페이지 처리

### 탐색

- `<page url="...">` 태그만 내부 페이지로 간주
- BFS 큐로 연결된 모든 페이지 수집
- 이미 방문한 `page_id`는 skip (순환 방지)
- 외부 URL(`https://`, `mailto:`)은 변환하지 않음

### page_id 정규화

```python
# 허용: 32자 hex 단독
# 허용: notion.so / notion.site URL에서 32자 hex 추출
# 거부: 슬러그 포함 문자열 그대로 (AI-IT-305d03...)
```

### 링크 라벨 추출

`<page>` 내부 HTML에서:

1. `<span>`, `<br>` 등 태그 제거
2. `\*` 이스케이프 해제
3. `*` 마크다운 강조 제거
4. 공백 정규화 → 링크 텍스트

### 상대 링크 계산

```python
# from: ai-it-profile.md
# to:   pages/kdt-100.md
# result: pages/kdt-100.md

# from: pages/a.md
# to:   pages/b.md
# result: b.md
```

`Path.relative_to(from_path.parent)` 사용.

## 이미지 로컬화

### 처리 순서 (페이지별)

1. `file://` URL → JSON decode
2. `POST https://www.notion.so/api/v3/getSignedFileUrls`
3. signed URL 다운로드 → `assets/` (또는 `pages/assets/<slug>/`)
4. markdown 내 경로 치환: `![](assets/filename.png)`
5. 동일 `source` attachment는 dedupe (한 번만 다운로드)

### 파일명

- attachment `source`의 마지막 `:` 이후 파일명 사용
- 특수문자 → `_` 치환, 한글 유지
- 빈 이름 → `image-NN.png`

## 실행

프로젝트에 스크립트가 있으면:

```bash
python3 scripts/download_notion_page.py
```

스크립트 수정 시 `ROOT_PAGE_ID`, `ROOT`, 루트 출력 파일명만 프로젝트에 맞게 조정.

## 완료 검증 체크리스트

- [ ] `ntn doctor` 통과
- [ ] 모든 `<page url>` → `[text](relative.md)` 변환됨
- [ ] `file://` 잔여 0건
- [ ] 각 `.md`의 이미지 경로가 실제 파일과 일치
- [ ] 재귀 탐색 페이지 수 = 저장된 `.md` 수
- [ ] 변경 요약 사용자에게 보고

## 다운로드 후 사용자에게 보고할 내용

```
- 총 N페이지 (루트 1 + 내부 M)
- 이미지 K개 / assets 경로
- 변환된 내부 링크 목록 (from → to)
- Notion 확장 MD 잔존 여부 (의도적 보존)
```

---

# Notion 수정 vs 로컬 export 구분

| 작업 | 도구 | 출력 |
|------|------|------|
| 페이지 읽기/다운로드 | `ntn pages get` | Notion 확장 MD |
| 페이지 수정 | `ntn pages update` | Notion에 반영 |
| 로컬 MD → Notion | `ntn pages create/update` + stdin | Notion |
| 이미지 로컬 저장 | `getSignedFileUrls` + HTTP GET | `assets/` |
| API 탐색 | `ntn api ls`, `--spec`, `--docs` | 참고용 |

로컬 export는 **읽기 전용 작업**이다. 원본 Notion 페이지는 변경하지 않는다.

---

# 최종 시스템 프롬프트

```
너는 Notion CLI를 사용할 수 있는 로컬 개발 Agent다.

작업 원칙:
- 먼저 조회하고 이후 수정한다.
- 원본을 보존한다.
- 변경 내용을 요약한다.
- 불확실하면 문서를 확인한다.
- Notion 워크스페이스의 원본 페이지·문서를 source of truth로 사용한다.
- 실행 로직과 대규모 처리는 외부 런타임에서 수행한다.
- 최소 권한 원칙을 따른다.

로컬 다운로드 추가 원칙:
- page_id는 32자 hex만 사용한다 (URL 슬러그 금지).
- ntn pages get으로 조회하고, ntn api GET block children에는 의존하지 않는다.
- file:// 이미지는 getSignedFileUrls(url=source)로 해석한다.
- <page url> 내부 링크는 재귀 탐색 후 상대 .md 링크로 변환한다.
- Notion 확장 MD를 GFM으로 오해하지 않는다; 변환은 명시적 요청 시에만 한다.
```

---

# 참고: 핵심 API 스니펫

## 페이지 조회

```bash
ntn pages get 305d03212bd4807d9be2f771c42a8cb6
ntn pages get 305d03212bd4807d9be2f771c42a8cb6 --json
```

## signed image URL (macOS)

```python
import json, subprocess, urllib.request

token = subprocess.check_output(
    ["security", "find-generic-password", "-s", "notion-cli", "-w"],
    stderr=subprocess.DEVNULL,
).decode().strip()

payload = {
    "urls": [{
        "url": ref["source"],
        "permissionRecord": ref["permissionRecord"],
    }]
}
req = urllib.request.Request(
    "https://www.notion.so/api/v3/getSignedFileUrls",
    data=json.dumps(payload).encode(),
    headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    },
    method="POST",
)
signed_url = json.load(urllib.request.urlopen(req))["signedUrls"][0]
```

## 내부 링크 정규식

```python
PAGE_TAG_RE = re.compile(r'<page url="([^"]+)">(.*?)</page>', re.DOTALL)
IMAGE_RE = re.compile(r'!\[([^\]]*)\]\((file://[^)]+)\)')
PAGE_ID_RE = re.compile(r'(?:notion\.so/|notion\.site/)(?:[^/\s?#]+-)?([0-9a-f]{32})', re.I)
```
