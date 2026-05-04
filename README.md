# Claude-Codex Proxy Stack

Multi-model local proxy stack for [Claude Code](https://claude.ai/code) and [Codex CLI](https://github.com/openai/codex) — connect DeepSeek, GLM, and Qwen through protocol-translating proxies with context management, reasoning passthrough, and tool-calling compatibility. Battle-tested on Windows 11 with 17 documented edge cases resolved.

## Overview

Claude Code speaks the Anthropic Messages API. Codex CLI speaks the OpenAI Responses API. Neither DeepSeek, GLM, nor Qwen implement these protocols natively. This project provides a set of local proxies that bridge the gap:

- **anthropic-proxy.mjs** (port 7861) — unified Anthropic Messages gateway routing GLM/DeepSeek/Qwen
- **deepseek_proxy.py** (port 9100) — Responses API to Chat Completions translator for Codex
- **qwen2API** (port 7860, Docker) — native Anthropic endpoint for Qwen with tool-use optimization
- **SwitchModel** — per-session model switching without touching config files
- **WebSearch script** — search fallback for non-Anthropic models
- **Hook system** — context monitoring, error recovery, memory persistence

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Claude Code CLI                       │
│               (Anthropic Messages API)                   │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│              anthropic-proxy.mjs :7861                   │
│         Unified context-management proxy                 │
│  (token inflation / error interception / preflight)      │
└───┬─────────────────┬─────────────────┬─────────────────┘
    │                 │                 │
┌───▼───────────┐ ┌───▼──────────┐ ┌───▼──────────────────┐
│ mydamoxing.cn │ │ api.         │ │ qwen2API :7860       │
│  GLM-5.1      │ │ deepseek.com │ │  Qwen 3.6 Plus       │
│  Anthropic    │ │ V4 Pro/Flash │ │  Native Anthropic    │
└───────────────┘ └──────────────┘ └──────────────────────┘


┌─────────────────────────────────────────────────────────┐
│                     Codex CLI                            │
│               (OpenAI Responses API)                     │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│               deepseek_proxy.py :9100                    │
│        Responses API ↔ Chat Completions translator       │
│  (reasoning passthrough / tool coercion / retry logic)   │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  api.deepseek.com                        │
│                  DeepSeek V4 Pro                         │
└─────────────────────────────────────────────────────────┘
```

**Critical constraint:** Clash Verge runs on `127.0.0.1:7897` and intercepts all HTTP traffic by default. All local proxies (7860/7861/9100) must be excluded via `NO_PROXY=127.0.0.1,localhost,::1` — otherwise Clash returns 502 for every local request.

## Features

| Feature | Claude Code | Codex CLI |
|---|---|---|
| DeepSeek V4 Pro (1M context, reasoning) | via anthropic-proxy | via deepseek_proxy |
| DeepSeek V4 Flash | via anthropic-proxy | via MODEL_ALIAS |
| GLM-5.1 (mydamoxing) | via anthropic-proxy | — |
| Qwen 3.6 Plus (local Docker) | direct to qwen2API :7860/anthropic | — |
| Tool calling (shell, file edit, web fetch) | triple-layer defense | argument coercion + retry |
| Reasoning/thinking passthrough | proxy passthrough | full convert_request pipeline |
| Context management | token inflation + error intercept | auto-compact |
| Model switching per session | SwitchModel (PowerShell) | config.toml |
| Web search fallback | websearch.py | websearch.py |
| Long-term memory | MEMORY.md + interactions/ | shared memory path |
| Boot-on-startup | — | startup folder .bat |

## Prerequisites

| Tool | Version / Path | Purpose |
|---|---|---|
| Python | 3.12+ (`D:/tool/py/python.exe`) | deepseek_proxy, websearch, hooks |
| Node.js | 24+ | anthropic-proxy.mjs, session-memory hook |
| Docker | — | qwen2API container |
| Git | 2.50+ | version control |
| ripgrep | 15+ | file search |
| Clash Verge | 127.0.0.1:7897 | network proxy (must NO_PROXY localhost) |
| Claude Code CLI | latest | main coding agent |
| Codex CLI | 0.128+ | secondary agent via DeepSeek |

## Quick Start

### 1. Claude Code Side

#### 1.1 Start qwen2API (Docker)

```powershell
docker run -d --name qwen2api `
  -p 7860:7860 `
  -e ADMIN_KEY=qwen2api-admin-2026 `
  -e MAX_INFLIGHT=2 `
  -e MAX_RETRIES=5 `
  yujunzhixue/qwen2api
```

Management console at `http://127.0.0.1:7860`.

#### 1.2 Start anthropic-proxy.mjs

```powershell
# Ensure NO_PROXY is set first
$env:NO_PROXY = "127.0.0.1,localhost,::1"
$env:HTTP_PROXY = ""

node ~/Desktop/qwen2api/anthropic-proxy.mjs
```

This proxy listens on port 7861 and routes requests based on `ANTHROPIC_MODEL`:
- `glm-5.1` / `glm-4.7` → mydamoxing.cn (Anthropic passthrough)
- `deepseek-v4-pro` / `deepseek-v4-flash` → api.deepseek.com (Anthropic passthrough)
- `qwen3.6-plus` → qwen2API :7860 (Anthropic→OpenAI translation)

#### 1.3 Configure Claude Code

Create `~/.claude/glm-settings.json` (or equivalent for your default model):

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:7861",
    "ANTHROPIC_MODEL": "glm-5.1",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-5.1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-5.1",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5.1",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "32768",
    "MAX_MCP_OUTPUT_TOKENS": "32768"
  },
  "modelOverrides": {
    "claude-4.5-opus": "glm-4.7",
    "claude-4.6-sonnet": "glm-4.7",
    "claude-4.5-haiku": "glm-4.7"
  },
  "maxTokens": 32768,
  "autoCompactThreshold": 0.7
}
```

Launch with:

```powershell
claude --settings ~/.claude/glm-settings.json
```

#### 1.4 SwitchModel (Per-Session Model Switching)

`SwitchModel` is a PowerShell 7 function (sourced from `~/.claude/scripts/switch.ps1`). It sets process-level environment variables only — no file writes, auto-cleanup on terminal close.

```powershell
SwitchModel glm            # GLM 5.1 (mydamoxing)
SwitchModel deepseek-pro   # DeepSeek V4 Pro
SwitchModel deepseek-flash # DeepSeek V4 Flash
SwitchModel qwen           # Qwen 3.6 Plus (local :7860/anthropic)
SwitchModel claude         # Clear override — back to settings.json default
SwitchModel status         # Show current session model
```

### 2. Codex CLI Side

#### 2.1 Start deepseek_proxy.py

```batch
# start_proxy.bat — double-click or run in terminal
@echo off
netstat -ano | findstr ":9100" | findstr LISTENING >nul 2>&1
if %ERRORLEVEL% EQU 0 exit /b 0
set HTTP_PROXY=
set HTTPS_PROXY=
start "" /B "D:\tool\py\pythonw.exe" "C:\Users\<user>\.codex\deepseek_proxy.py" 9100
```

This launches Flask + waitress on port 9100 in background (no console window). All stdout/stderr redirect to `~/.codex/proxy.log`.

#### 2.2 Configure Codex

`~/.codex/config.toml`:

```toml
model = "gpt-5.5"
model_provider = "deepseek"
model_reasoning_effort = "high"
model_reasoning_summary = "auto"
ask_for_approval = "never"
sandbox = "danger-full-access"

[model_providers.deepseek]
name = "DeepSeek"
base_url = "http://127.0.0.1:9100/v1"
env_key = "OPENAI_API_KEY"
wire_api = "responses"
```

Why `model = "gpt-5.5"`? Codex has built-in metadata for GPT-5.5 — it correctly sends `reasoning.effort` in requests and uses `shell_command` (string format). The proxy maps it back to `deepseek-v4-pro` before forwarding upstream.

#### 2.3 Launch Codex

```batch
# start_codex.bat
@echo off
chcp 65001 >nul
call "%USERPROFILE%\.codex\start_proxy.bat"
set HTTP_PROXY=
set HTTPS_PROXY=
set NO_PROXY=127.0.0.1,localhost,::1
set no_proxy=127.0.0.1,localhost,::1
codex %*
```

---

## Component Details

### anthropic-proxy.mjs (Port 7861)

**Location:** `~/Desktop/qwen2api/anthropic-proxy.mjs`

The unified context-management proxy. All three models pass through port 7861 to reach Claude Code.

**Key capabilities:**

- **Model routing** — dispatches by `ANTHROPIC_MODEL` env var to the correct upstream
- **Triple-layer context defense:**
  1. `count_tokens` inflation — when tokens exceed 80% of model window, reports 180K~200K to trigger Claude Code auto-compact
  2. Error interception — translates upstream context-limit errors to Anthropic standard format
  3. Request preflight — rejects requests exceeding model window with HTTP 400
- **Tool-calling passthrough** — preserves tool schemas for DeepSeek/GLM

**API keys configured in-code:**
- GLM (mydamoxing): <your-mydamoxing-key>
- DeepSeek: <your-deepseek-key>
- Qwen: <your-qwen2api-key>

### deepseek_proxy.py (Port 9100)

**Location:** `~/.codex/deepseek_proxy.py` (~520 lines)

A Flask + waitress server that translates OpenAI Responses API (Codex native) to DeepSeek Chat Completions API.

**Key capabilities:**

- `convert_request(data)` — full protocol translation:
  - Buffers reasoning items and attaches `reasoning_content` to the next assistant message (required by DeepSeek thinking mode)
  - Merges parallel function_call items into a single assistant message (avoids DeepSeek 400 error)
  - Maps `gpt-5.5` → `deepseek-v4-pro` via `MODEL_ALIAS`
  - Sets `thinking.enabled` for pro models, `thinking.disabled` for flash or minimal effort

- `coerce_tool_args(name, args_str)` — bidirectional argument conversion:
  - `shell_command`: array `["bash","-lc","cmd"]` → string `"cmd"` (GPT-5.5 requirement)
  - `apply_patch`: JSON → raw patch text
  - Legacy `shell`/`local_shell`: string → array

- `fetch_chunks()` — dual-mode upstream:
  - Tools present → requests library, non-streaming, 3 retries (1/2/3s backoff)
  - No tools → httpx library, true streaming, auto-fallback to non-streaming on disconnect

- **SSE state machine** — lazily creates reasoning/message/tool_call items, ensures correct event ordering (reasoning closed before message opened), all `done` events carry accumulated text

- **pythonw compatibility** — detects `sys.stdout is None` and redirects to `proxy.log`

**Debug files:**
- `~/.codex/last_inbound.json` — raw request from Codex
- `~/.codex/last_request.json` — converted request sent to DeepSeek
- `~/.codex/proxy.log` — runtime stdout/stderr

### qwen2API (Port 7860, Docker)

**Image:** `yujunzhixue/qwen2api`

Exposes three API formats on port 7860:
- `/v1/chat/completions` — OpenAI format
- `/anthropic/v1/messages` — **Native Anthropic** with tool-calling optimization (schema compression, tool name obfuscation, few-shot injection)
- `/v1beta/models/...` — Gemini format

**Important:** Claude Code must connect to `/anthropic/v1/messages` directly. Routing through anthropic-proxy.mjs (which does Anthropic→OpenAI translation) bypasses qwen2API's optimization code, resulting in 0-token responses and no tool usage.

**Docker configuration:**
```
MAX_INFLIGHT=2
MAX_RETRIES=5
ACCOUNT_MIN_INTERVAL_MS=600
REQUEST_JITTER=120~360ms
shm_size=512m
```

### SwitchModel

**Location:** `~/.claude/scripts/switch.ps1`, registered as a PowerShell 7 function.

Per-session model switching. Sets process-level `$env:ANTHROPIC_*` variables only — never writes to config files. Terminal close = auto-cleanup.

### WebSearch Script

**Location:** `~/.claude/scripts/websearch.py`

Claude Code's native WebSearch tool only works with official Anthropic models. For GLM/DeepSeek/Qwen, use this script instead:

```bash
D:/tool/py/python.exe ~/.claude/scripts/websearch.py "query"
D:/tool/py/python.exe ~/.claude/scripts/websearch.py -n 10 "query"
D:/tool/py/python.exe ~/.claude/scripts/websearch.py --news "query"
D:/tool/py/python.exe ~/.claude/scripts/websearch.py --json "query"
```

Supports `-n` (result count), `--news` (time-sensitive), `--json` (structured output), and `--backend` (search engine selection).

### Hook System (Claude Code)

| Hook | Trigger | Purpose |
|---|---|---|
| `pre-compact-hook.py` | PreCompact | Update STATE.md before context compression |
| `session-start-hook.py` | SessionStart | Inject CWD/STATE.md + memory index |
| `context-monitor-hook.py` | UserPromptSubmit | Check context water level each input |
| `error-recovery-hook.py` | Notification (error) | Trigger 6-step self-repair instructions |
| `session-memory-hook.cjs` | SessionStart (compact/clear) | Record session summary |
| `observation-logger.cjs` | PostToolUse | Log key observations after each tool call |
| `statusline-wrapper.js` | statusLine | Correct context window percentage display |

### Memory System

**Location:** `~/.claude/projects/C--Users----/memory/`

A markdown-based long-term memory system indexed by `MEMORY.md`. Categories:
- `user_*.md` — user profile
- `feedback_*.md` — lessons learned (42 files)
- `project_*.md` — project states
- `reference_*.md` — configuration references
- `interactions/` — dynamic data (quiz records, study progress)

### AGENTS.md / CLAUDE.md

Global agent instructions injected into system context:

- `~/.codex/AGENTS.md` — Codex-specific: shell priority (exe > bash > powershell), GPT-5.5 string-format commands, websearch fallback
- `~/.claude/CLAUDE.md` — Claude-specific: zero-attribution policy, coding principles, error handling, STATE.md handoff

---

## Known Issues & Fixes

### Clash Intercepts Localhost (Issue #1)
**Symptom:** `curl 127.0.0.1:9100` returns 502.
**Diagnosis:** `curl -v` shows `Trying 127.0.0.1:7897`.
**Fix:** Set `NO_PROXY=127.0.0.1,localhost,::1` in 4 places: `setx` (system-wide), bashrc, PowerShell profile, startup scripts. Also clear `HTTP_PROXY`.

### Codex 0.128 Requires wire_api=responses (Issue #2)
**Symptom:** `wire_api=chat` causes startup error.
**Fix:** Must use `wire_api=responses`. The local proxy is the only path.

### httpx Streaming + DeepSeek Multi-byte Chars (Issue #3)
**Symptom:** `peer closed connection without sending complete message body` (RemoteProtocolError).
**Root cause:** httpx 0.28 chunked tail decode fails with multi-byte characters.
**Fix:** Switch to requests non-streaming when tools are present. Auto-fallback on disconnect without tools.

### DeepSeek Stream Drops on Tool Args (Issue #4)
**Symptom:** Upstream server disconnects mid-stream when tool arguments are present.
**Fix:** Force non-streaming when tools are in the request.

### Random ChunkedEncodingError (Issue #5)
**Symptom:** Intermittent upstream errors.
**Fix:** 3 retries with 1/2/3-second backoff in proxy.

### Codex Doesn't Send reasoning.effort for Non-OpenAI Providers (Issue #6)
**Symptom:** `model_reasoning_effort=high` in config but absent from request.
**Fix:** Masquerade as `gpt-5.5` (Codex has built-in metadata). Proxy defaults thinking=enabled for pro models.

### pythonw stdout=None Crashes Flask (Issue #7)
**Symptom:** Background pythonw process crashes on first print.
**Fix:** Detect `sys.stdout is None` at proxy start and redirect to `proxy.log`.

### Codex Forces PowerShell by Default (Issue #8)
**Symptom:** Model uses PowerShell for every command — slow startup, awkward Unix commands.
**Fix:** AGENTS.md explicitly states shell priority: direct exe > bash > powershell. Empirically overrides default behavior.

### Streaming Done Events Had Empty Text (Issue #9)
**Symptom:** Codex TUI shows blank output but token count increments.
**Fix:** `response.content_part.done` and `response.output_item.done` must carry accumulated text.

### DeepSeek Thinking Mode Requires Historical reasoning_content (Issue #10)
**Symptom:** `The reasoning_content in the thinking mode must be passed back to the API`.
**Fix:** convert_request buffers reasoning item summary text and attaches it as `reasoning_content` to the next assistant message.

### Streaming State Machine Indentation Bug (Issue #11)
**Symptom:** Stream output blank because all chunk processing was nested inside `if not response_started:`.
**Fix:** Moved the condition inside the for-loop at correct indentation level.

### shell_command Argument Format Mismatch (Issue #12)
**Symptom:** Model outputs array `["bash","-lc","..."]`, GPT-5.5 expects string → `failed to parse: expected a string`.
**Fix:** coerce_tool_args handles `shell_command` array→string conversion.

### Parallel Tool Call Splitting Causes DeepSeek 400 (Issue #13)
**Symptom:** 8 parallel function_calls become 8 separate assistant messages → DeepSeek returns `insufficient tool messages` → Codex treats empty response as task complete.
**Fix:** Buffer consecutive function_call items, flush as a single merged assistant message.

### Context Window Percentage Display (Issue #14)
**Symptom:** Status bar shows inflated context percentage.
**Root cause:** CLI calculates denominator as 200K for proxy models; actual window is 1M.
**Fix:** statusline-wrapper.js rewrites `context_window_size` based on model name.

### CLAUDE_CODE_CONTEXT_WINDOW is Not a Valid Env Var (Issue #15)
**Fix:** Deleted the invalid env var. Use statusline-wrapper.js for display fix. Rely on autoCompactThreshold for actual compaction.

### mydamoxing glm-5.1 Only Has OpenAI Endpoint (Issue #16)
**Symptom:** Direct Anthropic endpoint connection fails for glm-5.1.
**Fix:** Route through 7861 proxy which does Anthropic→OpenAI translation for glm-5.1.

### qwen2API Must Use Native Anthropic Endpoint (Issue #17)
**Symptom:** Routing through 7861 translation layer causes Qwen to return 0 tokens, no tool calls.
**Root cause:** Bypasses qwen2API's built-in Anthropic optimization (schema compression, tool name obfuscation, few-shot injection).
**Fix:** Claude Code connects directly to `:7860/anthropic/v1/messages`.

---

## Configuration Reference

### Required Environment Variables

```bash
# System-wide (setx)
NO_PROXY=127.0.0.1,localhost,::1

# Claude Code (in settings.json env or SwitchModel)
ANTHROPIC_BASE_URL=http://127.0.0.1:7861
ANTHROPIC_AUTH_TOKEN=<your-key>
ANTHROPIC_MODEL=glm-5.1
CLAUDE_CODE_MAX_OUTPUT_TOKENS=32768

# Codex (in config.env)
OPENAI_API_KEY=<placeholder>
DEEPSEEK_API_KEY=<your-key>
NO_PROXY=127.0.0.1,localhost
```

### File Map

```
~/
├── .claude/
│   ├── CLAUDE.md                    # Global agent rules
│   ├── settings.json                # Claude Code config
│   ├── glm-settings.json            # GLM default model config
│   ├── scripts/
│   │   ├── switch.ps1               # SwitchModel function
│   │   ├── websearch.py             # Web search fallback
│   │   ├── pre-compact-hook.py      # STATE.md update before compact
│   │   ├── session-start-hook.py    # Memory index injection
│   │   ├── context-monitor-hook.py  # Context water level check
│   │   ├── error-recovery-hook.py   # Error-triggered repair
│   │   └── statusline-wrapper.js    # Context percentage fix
│   ├── session-memory/
│   │   ├── session-memory-hook.cjs  # Session summary logger
│   │   └── observation-logger.cjs   # Tool observation logger
│   └── projects/C--Users----/memory/
│       ├── MEMORY.md                # Memory index
│       └── ...                      # Memory files
├── .codex/
│   ├── config.toml                  # Codex model/provider config
│   ├── config.env                   # API keys + NO_PROXY
│   ├── AGENTS.md                    # Global agent rules
│   ├── deepseek_proxy.py            # Responses→Chat Completions proxy
│   ├── start_proxy.bat              # Launch proxy (pythonw)
│   ├── start_codex.bat              # Launch Codex with proxy
│   ├── last_inbound.json            # Debug: raw Codex request
│   ├── last_request.json            # Debug: converted request
│   └── proxy.log                    # Debug: proxy stdout/stderr
└── Desktop/qwen2api/
    ├── anthropic-proxy.mjs          # Unified context proxy
    └── start_proxy.bat              # Launch with NO_PROXY
```

---

## Debugging

### Codex + DeepSeek

| Check | Command |
|---|---|
| Is proxy running? | `netstat -ano | findstr ":9100"` |
| Test proxy directly | `curl --noproxy 127.0.0.1 http://127.0.0.1:9100/v1/models` |
| Test DeepSeek directly | `curl https://api.deepseek.com/v1/models -H "Authorization: Bearer <key>"` |
| View raw Codex request | `cat ~/.codex/last_inbound.json` |
| View converted request | `cat ~/.codex/last_request.json` |
| View proxy logs | `cat ~/.codex/proxy.log` |
| Restart proxy | `taskkill /F /PID <pid>` then `cmd /c ~/.codex/start_proxy.bat` |

### Claude + Multi-Model

| Check | Command |
|---|---|
| Memory index | `cat ~/.claude/projects/C--Users----/memory/MEMORY.md` |
| qwen2API console | `http://127.0.0.1:7860` |
| Current model | `SwitchModel status` |
| Global config | `cat ~/.claude/glm-settings.json` |

---

## License

MIT
