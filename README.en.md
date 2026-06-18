![AI Skills for Everyone](author/wildmental-bjpark.png)

# notion-cli 
> Skill for Cursor, Claude, Codex agents

**Language / 언어:** [한국어](README.md) · [English](README.en.md)

**A Skill that reduces the trial and error an Agent commonly runs into when reading, downloading, and editing Notion pages and documents with the Notion CLI (`ntn`). Works with Cursor, Claude Code, and Codex.**

Downloading a Notion page to local Markdown, resolving its images, and wiring up internal links looks simple on paper, but in practice a chain of pitfalls awaits — **page ID formats, Notion-specific Markdown, `file://` image references, and API hangs**. This skill **documents that trial and error in advance and locks it into Agent behavior rules**.

---

## What this skill solves

### 1. Lets the Agent handle Notion safely

- **Always read first** before changing anything (`ntn pages get`, `ntn doctor`)
- Preserve the original, prefer append, no destructive actions
- When unsure, check `-help` / the official docs

### 2. Provides a canonical workflow for local export

```
ntn doctor → normalize page_id → BFS to traverse internal pages
→ ntn pages get → localize images → convert <page> to relative links → verify
```

- The `ntn pages get` + `getSignedFileUrls` combination (avoids the Public API block children)
- `assets/` directory conventions, slug collision handling, circular-link prevention
- After completion, report the **page count, image count, and link conversion list**

### 3. Makes the Notion ↔ Local LLM architecture explicit

```
Notion (source of truth)
    ↕  Notion CLI (ntn)
    ↕  Local LLM Agent
    ↕  GitHub / Local Runtime / External Services
```

Notion is the human-facing document/collaboration workspace, the LLM is the analysis/generation engine, and the CLI is the connecting layer — the boundaries are defined so the roles don't get mixed up.

---

## Why this skill is needed

| Common attempt | Expectation | Reality |
|-----------|------|------|
| `ntn pages get` with a URL slug | Page lookup | `400 validation_error` |
| Previewing `ntn pages get` output on GitHub | Standard Markdown | Notion extension syntax like `<columns>`, `<page>` |
| `ntn api GET v1/blocks/.../children` | Collect blocks & images | Long hang / timeout |
| Calling the API with the `source` field of `file://` JSON as-is | signed URL | The field name must be **`url`** |
| Downloading an image with a Korean filename | Save locally | `UnicodeEncodeError` |
| Looking for the token in `~/.config/notion/` | Credentials | On macOS it's the Keychain entry **`notion-cli`** |

Rather than just "knowing to avoid" the patterns above, this skill organizes them into operational rules that span **read-first → a safe export path → a verification checklist**.

---


## Quick start

### Prerequisites

- [Notion CLI (`ntn`)](https://ntn.dev) installed and logged in
- Cursor, Claude Code, or Codex

```bash
curl -fsSL https://ntn.dev | bash
ntn login
ntn doctor   # confirm Token valid and Default workspace pass
```

### Installing the skill

Install the skill in either **personal** or **project** scope. Both scopes fetch only `SKILL.md` via `curl`. **Do not clone this entire repository into your working repo.**

| | Personal skill | Project skill |
|---|----------|--------------|
| **Scope** | Every project I open | Only the current repo |
| **Path** | `~/…/skills/notion-cli/` | `<repo-root>/.cursor/skills/notion-cli/`, etc. |
| **Git impact** | No files added to the working repo | Skill files can be committed to the repo (team sharing) |
| **When to use** | When using it solo across all projects | When pinning/sharing the skill in a team repo |

| Tool | Personal path | Project path |
|------|-----------|---------------|
| Cursor | `~/.cursor/skills/notion-cli/` | `.cursor/skills/notion-cli/` |
| Claude Code | `~/.claude/skills/notion-cli/` | `.claude/skills/notion-cli/` |
| Codex | `~/.agents/skills/notion-cli/` | `.agents/skills/notion-cli/` |

#### Personal skill (recommended — no Git repo changes)

```bash
# Cursor
mkdir -p ~/.cursor/skills/notion-cli
curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.cursor/skills/notion-cli/SKILL.md \
  -o ~/.cursor/skills/notion-cli/SKILL.md

# Claude Code
mkdir -p ~/.claude/skills/notion-cli
curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.claude/skills/notion-cli/SKILL.md \
  -o ~/.claude/skills/notion-cli/SKILL.md

# Codex
mkdir -p ~/.agents/skills/notion-cli
curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.agents/skills/notion-cli/SKILL.md \
  -o ~/.agents/skills/notion-cli/SKILL.md
```

#### Project skill (install to project paths)

Use this only when you want to include and share the AI skill configuration in the repo.

```bash
# Cursor
mkdir -p .cursor/skills/notion-cli
curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.cursor/skills/notion-cli/SKILL.md \
  -o .cursor/skills/notion-cli/SKILL.md

# Claude Code
mkdir -p .claude/skills/notion-cli
curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.claude/skills/notion-cli/SKILL.md \
  -o .claude/skills/notion-cli/SKILL.md

# Codex
mkdir -p .agents/skills/notion-cli
curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.agents/skills/notion-cli/SKILL.md \
  -o .agents/skills/notion-cli/SKILL.md
```

Just pick and run the command for the tool you need (Cursor / Claude Code / Codex).

#### After installation

- **Cursor**: do a one-time **Reload Window**
- **Claude Code**: skill edits reload live during a session; a new top-level `.claude/skills/` created after the session starts may require a restart
- **Codex**: if the skill doesn't show up, restart Codex

### How to use

When you request a Notion-related task, the Agent applies the skill automatically.

| Tool | Manual invocation |
|------|-----------|
| Cursor | `/notion-cli` |
| Claude Code | `/notion-cli` |
| Codex | `/skills` or `$notion-cli` |

The Claude Code version automatically injects the `ntn doctor` result when the skill loads. The Codex version instructs it to run `ntn doctor` at the start of a task.

**Example requests it applies to:**

- "Download this Notion page as Markdown"
- "Save it locally with images and wire up the internal links too"
- "Read and summarize the contents of this Notion page"
- "Check the page status with ntn"

---

## Key pitfalls at a glance

The skill body has the detailed rules; below are the spots people most often get stuck on.

### Page IDs are 32-character hex only

```bash
# ✅
ntn pages get 305d03212bd4807d9be2f771c42a8cb6

# ❌ includes the URL slug
ntn pages get AI-IT-305d03212bd4807d9be2f771c42a8cb6
```

### `ntn pages get` output ≠ GFM

Notion extended Markdown (`<columns>`, `<page url>`, `:icon:`) is a **valid export**. Convert to GFM only when explicitly requested for web/GitHub use.

### Images use `getSignedFileUrls` + the `url` field

The `source` value in the JSON after `file://` must be passed in the API request under the **`url`** key. URL-encode Korean filenames before requesting.

### macOS token location

```bash
security find-generic-password -s "notion-cli" -w
```

---

## Recommended local directory layout

The layout the skill follows when recursively downloading a Notion page:

```
<project-root>/
├── <root-page>.md              # root page
├── assets/                     # root images
├── pages/
│   ├── <slug>.md               # internally linked sub-pages
│   └── assets/<slug>/          # sub-page images
└── scripts/
    └── download_notion_page.py # recursive download script (optional)
```

---

## Completion verification checklist

After a local export task, the Agent confirms the following.

- [ ] `ntn doctor` passes
- [ ] Every `<page url>` converted to `[text](relative.md)`
- [ ] Zero remaining `file://`
- [ ] Image paths in each `.md` match the actual files
- [ ] Recursively traversed page count = number of saved `.md` files

---

## Skill composition

```
.cursor/skills/notion-cli/SKILL.md   # for Cursor
.claude/skills/notion-cli/SKILL.md   # for Claude Code (auto-injects ntn doctor)
.agents/skills/notion-cli/SKILL.md   # for Codex
```

| Section | Content |
|------|------|
| Basic principles | Read first, preserve the original, least privilege |
| Trial and error & known pitfalls | 7 practical pitfalls and their fixes |
| Local download rules | BFS traversal, image localization, relative links |
| API snippets | `getSignedFileUrls`, regular expressions, page_id normalization |

---

## Recommended for

- People who manage documents, wikis, and project pages in a Notion **workspace** and want to read, analyze, and update them with a Cursor, Claude Code, or Codex Agent
- People who spent time **forcing a GFM conversion** because the `ntn pages get` output looked like "broken Markdown"
- People who **gave up on local export** because of images and internal links, or who debug it from scratch every time
- People who want to hand Notion work to an Agent while avoiding **corrupting the original or using the wrong API path**

---

## References

- [Notion CLI (`ntn`)](https://ntn.dev)
- Cursor Agent Skills: [Creating Skills](https://docs.cursor.com)
- Claude Code Skills: [Extend Claude with skills](https://docs.anthropic.com/en/docs/claude-code/skills)
- Codex Agent Skills: [Agent Skills](https://developers.openai.com/codex/skills/)

---

## For LLMs

<!-- install-only; operational rules live in SKILL.md — load skill after install -->

```
skill=notion-cli
repo=https://github.com/wild-mental/notion-cli-skill
install.method=curl-only  # never clone this repo into user's project

flow:
  1. ask scope: "개인 스킬(~/, 모든 프로젝트) vs 프로젝트 스킬(./, 현재 repo만) 중 어디에 설치할까요?"
  2. ask tools: "Cursor / Claude Code / Codex 중 어떤 도구에 설치할까요?" (복수 선택 가능)
  3. run matching curl commands below; do not install both scopes unless user asks
  4. post_install steps; then load SKILL.md — do not infer rules from README

scope.user.paths:
  cursor=~/.cursor/skills/notion-cli/SKILL.md
  claude=~/.claude/skills/notion-cli/SKILL.md
  codex=~/.agents/skills/notion-cli/SKILL.md

scope.project.paths:
  cursor=.cursor/skills/notion-cli/SKILL.md
  claude=.claude/skills/notion-cli/SKILL.md
  codex=.agents/skills/notion-cli/SKILL.md

install.user.cursor=mkdir -p ~/.cursor/skills/notion-cli && curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.cursor/skills/notion-cli/SKILL.md -o ~/.cursor/skills/notion-cli/SKILL.md
install.user.claude=mkdir -p ~/.claude/skills/notion-cli && curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.claude/skills/notion-cli/SKILL.md -o ~/.claude/skills/notion-cli/SKILL.md
install.user.codex=mkdir -p ~/.agents/skills/notion-cli && curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.agents/skills/notion-cli/SKILL.md -o ~/.agents/skills/notion-cli/SKILL.md

install.project.cursor=mkdir -p .cursor/skills/notion-cli && curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.cursor/skills/notion-cli/SKILL.md -o .cursor/skills/notion-cli/SKILL.md
install.project.claude=mkdir -p .claude/skills/notion-cli && curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.claude/skills/notion-cli/SKILL.md -o .claude/skills/notion-cli/SKILL.md
install.project.codex=mkdir -p .agents/skills/notion-cli && curl -fsSL https://raw.githubusercontent.com/wild-mental/notion-cli-skill/main/.agents/skills/notion-cli/SKILL.md -o .agents/skills/notion-cli/SKILL.md

install.project.note=run from repo root; adds tracked files — confirm user chose project scope

invoke.cursor=/notion-cli
invoke.claude=/notion-cli
invoke.codex=/skills|$notion-cli

post_install.cursor=Reload Window
post_install.claude=live reload; restart if new top-level .claude/skills/ after session start
post_install.codex=restart if skill not detected
```

---

## License

[MIT License](LICENSE)
