# Claude-Codex Proxy Stack

Claude Code 与 Codex CLI 的多模型本地代理方案——通过协议转换代理接入 DeepSeek、GLM、Qwen，含上下文管理、推理透传、工具调用兼容。Windows 11 实战验证，已解决 17 个边界问题。

## 概述

Claude Code 使用 Anthropic Messages API。Codex CLI 使用 OpenAI Responses API。DeepSeek、GLM、Qwen 均不原生支持这些协议。本项目提供一组本地代理桥接：

- **anthropic-proxy.mjs**（端口 7861）——统一的 Anthropic Messages 网关，路由 GLM/DeepSeek/Qwen
- **deepseek_proxy.py**（端口 9100）——Responses API 到 Chat Completions 的协议转换器，供 Codex 使用
- **qwen2API**（端口 7860，Docker）——Qwen 的原生 Anthropic 端点，含工具调用优化
- **SwitchModel**——会话级模型切换，不触碰配置文件
- **WebSearch 脚本**——非 Anthropic 模型的搜索替代方案
- **Hook 系统**——上下文监控、错误恢复、记忆持久化

## 架构

```
┌─────────────────────────────────────────────────────────┐
│                    Claude Code CLI                       │
│               (Anthropic Messages API)                   │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│              anthropic-proxy.mjs :7861                   │
│              统一上下文管理代理                            │
│    (token 虚报 / 错误拦截 / 请求预检)                      │
└───┬─────────────────┬─────────────────┬─────────────────┘
    │                 │                 │
┌───▼───────────┐ ┌───▼──────────┐ ┌───▼──────────────────┐
│ mydamoxing.cn │ │ api.         │ │ qwen2API :7860       │
│  GLM-5.1      │ │ deepseek.com │ │  Qwen 3.6 Plus       │
│  Anthropic    │ │ V4 Pro/Flash │ │  原生 Anthropic       │
└───────────────┘ └──────────────┘ └──────────────────────┘


┌─────────────────────────────────────────────────────────┐
│                     Codex CLI                            │
│               (OpenAI Responses API)                     │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│               deepseek_proxy.py :9100                    │
│         Responses API ↔ Chat Completions 转换器           │
│     (推理透传 / 工具参数兜底 / 重试逻辑)                    │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│                  api.deepseek.com                        │
│                  DeepSeek V4 Pro                         │
└─────────────────────────────────────────────────────────┘
```

**关键约束：** Clash Verge 运行在 `127.0.0.1:7897`，默认劫持所有 HTTP 流量。所有本地代理（7860/7861/9100）必须通过 `NO_PROXY=127.0.0.1,localhost,::1` 排除，否则 Clash 对每个本地请求返回 502。

## 功能矩阵

| 功能 | Claude Code | Codex CLI |
|---|---|---|
| DeepSeek V4 Pro（1M 上下文，推理） | 经 anthropic-proxy | 经 deepseek_proxy |
| DeepSeek V4 Flash | 经 anthropic-proxy | 经 MODEL_ALIAS |
| GLM-5.1（mydamoxing 中转） | 经 anthropic-proxy | 不支持 |
| Qwen 3.6 Plus（本地 Docker） | 直连 qwen2API :7860/anthropic | 不支持 |
| 工具调用（shell、文件编辑、网页抓取） | 三层防御 | 参数兜底 + 重试 |
| 推理/思考透传 | proxy 透传 | 完整 convert_request 管线 |
| 上下文管理 | token 虚报 + 错误拦截 | 自动压缩 |
| 会话级模型切换 | SwitchModel（PowerShell） | config.toml |
| 网页搜索替代 | websearch.py | websearch.py |
| 长期记忆 | MEMORY.md + interactions/ | 共享记忆路径 |
| 开机自启 | 不支持 | startup 文件夹 .bat |

## 环境依赖

| 工具 | 版本/路径 | 用途 |
|---|---|---|
| Python | 3.12+ (`D:/tool/py/python.exe`) | deepseek_proxy、websearch、hooks |
| Node.js | 24+ | anthropic-proxy.mjs、session-memory hook |
| Docker | — | qwen2API 容器 |
| Git | 2.50+ | 版本控制 |
| ripgrep | 15+ | 文件搜索 |
| Clash Verge | 127.0.0.1:7897 | 网络代理（必须 NO_PROXY 排除 localhost） |
| Claude Code CLI | 最新版 | 主力编程 Agent |
| Codex CLI | 0.128+ | DeepSeek 驱动辅助 Agent |

## 快速开始

### 一、Claude Code 侧

#### 1. 启动 qwen2API（Docker）

```powershell
docker run -d --name qwen2api `
  -p 7860:7860 `
  -e ADMIN_KEY=qwen2api-admin-2026 `
  -e MAX_INFLIGHT=2 `
  -e MAX_RETRIES=5 `
  yujunzhixue/qwen2api
```

管理台：`http://127.0.0.1:7860`

#### 2. 启动 anthropic-proxy.mjs

```powershell
# 先设 NO_PROXY
$env:NO_PROXY = "127.0.0.1,localhost,::1"
$env:HTTP_PROXY = ""

node ~/Desktop/qwen2api/anthropic-proxy.mjs
```

代理监听 7861 端口，根据 `ANTHROPIC_MODEL` 路由：
- `glm-5.1` / `glm-4.7` → mydamoxing.cn（Anthropic 直透）
- `deepseek-v4-pro` / `deepseek-v4-flash` → api.deepseek.com（Anthropic 直透）
- `qwen3.6-plus` → qwen2API :7860（Anthropic→OpenAI 转换）

#### 3. 配置 Claude Code

创建 `~/.claude/glm-settings.json`（或你默认模型的等效配置）：

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

启动：

```powershell
claude --settings ~/.claude/glm-settings.json
```

#### 4. SwitchModel（会话级模型切换）

`SwitchModel` 是 PowerShell 7 函数（来自 `~/.claude/scripts/switch.ps1`）。仅设进程级环境变量，不写文件，关终端即自动清除。

```powershell
SwitchModel glm            # GLM 5.1（mydamoxing 中转）
SwitchModel deepseek-pro   # DeepSeek V4 Pro
SwitchModel deepseek-flash # DeepSeek V4 Flash
SwitchModel qwen           # Qwen 3.6 Plus（本地 :7860/anthropic）
SwitchModel claude         # 清除覆盖，回退 settings.json 默认
SwitchModel status         # 查看当前会话模型
```

### 二、Codex CLI 侧

#### 1. 启动 deepseek_proxy.py

```batch
:: start_proxy.bat — 双击或在终端运行
@echo off
netstat -ano | findstr ":9100" | findstr LISTENING >nul 2>&1
if %ERRORLEVEL% EQU 0 exit /b 0
set HTTP_PROXY=
set HTTPS_PROXY=
start "" /B "D:\tool\py\pythonw.exe" "C:\Users\<用户名>\.codex\deepseek_proxy.py" 9100
```

Flask + waitress 在 9100 端口后台运行（无控制台窗口）。stdout/stderr 自动重定向到 `~/.codex/proxy.log`。

#### 2. 配置 Codex

`~/.codex/config.toml`：

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

为什么是 `model = "gpt-5.5"`？Codex 内置 GPT-5.5 的完整 metadata，会正确下发 `reasoning.effort`，shell 工具使用 `shell_command`（字符串格式）。proxy 在转发前映射回 `deepseek-v4-pro`。

#### 3. 启动 Codex

```batch
:: start_codex.bat
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

## 组件详解

### anthropic-proxy.mjs（端口 7861）

**位置：**`~/Desktop/qwen2api/anthropic-proxy.mjs`

统一上下文管理代理。三个模型全部经 7861 端口接入 Claude Code。

**核心能力：**

- **模型路由** — 按 `ANTHROPIC_MODEL` 环境变量分发到正确上游
- **三层上下文防御：**
  1. `count_tokens` 虚报 — token 超过模型窗口 80% 时，上报 180K~200K，触发 Claude Code 自动压缩
  2. 错误拦截 — 将上游 context-limit 错误转为 Anthropic 标准格式
  3. 请求预检 — 超过模型窗口直接返回 HTTP 400
- **工具调用透传** — 为 DeepSeek/GLM 保留完整 tool schema

**API Key 在代码内配置，需自行替换为你的 key。**

### deepseek_proxy.py（端口 9100）

**位置：**`~/.codex/deepseek_proxy.py`（约 520 行）

Flask + waitress 服务，将 OpenAI Responses API（Codex 原生）转为 DeepSeek Chat Completions API。

**核心能力：**

- `convert_request(data)` — 完整协议转换：
  - 缓冲 reasoning items，在下一个 assistant 消息挂上 `reasoning_content`（DeepSeek thinking 模式必需）
  - 合并并行 function_call items 为单个 assistant 消息（避免 DeepSeek 400 错误）
  - 通过 `MODEL_ALIAS` 映射 `gpt-5.5` → `deepseek-v4-pro`
  - Pro 模型设 `thinking.enabled`，flash 或 minimal effort 设 `thinking.disabled`

- `coerce_tool_args(name, args_str)` — 工具参数双向转换：
  - `shell_command`：数组 `["bash","-lc","cmd"]` → 字符串 `"cmd"`（GPT-5.5 要求）
  - `apply_patch`：JSON → 裸 patch 文本
  - 旧 `shell`/`local_shell`：字符串 → 数组

- `fetch_chunks()` — 双模式上游请求：
  - 有 tools → requests 库，非流式，3 次重试（1/2/3 秒退避）
  - 无 tools → httpx 库，真流式，断开时自动回退非流式

- **SSE 状态机** — 懒创建 reasoning/message/tool_call 事件，确保事件顺序正确（reasoning 先关闭再开 message），所有 `done` 事件携带累积文本

- **pythonw 兼容** — 检测 `sys.stdout is None` 自动重定向到 `proxy.log`

**调试文件：**
- `~/.codex/last_inbound.json` — Codex 发来的原始请求
- `~/.codex/last_request.json` — 转换后发往 DeepSeek 的请求
- `~/.codex/proxy.log` — 运行时日志

### qwen2API（端口 7860，Docker）

**镜像：**`yujunzhixue/qwen2api`

在 7860 端口暴露三种 API 格式：
- `/v1/chat/completions` — OpenAI 格式
- `/anthropic/v1/messages` — **原生 Anthropic**，带工具调用优化（schema 压缩、tool name 混淆、few-shot 注入）
- `/v1beta/models/...` — Gemini 格式

**重要：**Claude Code 必须直连 `/anthropic/v1/messages`。经 anthropic-proxy.mjs 做 Anthropic→OpenAI 转换会绕过 qwen2API 的优化代码，导致 0 token 输出且无工具调用。

**Docker 关键配置：**
```
MAX_INFLIGHT=2
MAX_RETRIES=5
ACCOUNT_MIN_INTERVAL_MS=600
REQUEST_JITTER=120~360ms
shm_size=512m
```

### SwitchModel

**位置：**`~/.claude/scripts/switch.ps1`，注册为 PowerShell 7 函数。

会话级模型切换。仅设进程级 `$env:ANTHROPIC_*` 变量，绝不写配置文件。关终端即自动清除。

### WebSearch 脚本

**位置：**`~/.claude/scripts/websearch.py`

Claude Code 原生 WebSearch 工具仅 Anthropic 官方模型可用。GLM/DeepSeek/Qwen 统一用此脚本：

```bash
D:/tool/py/python.exe ~/.claude/scripts/websearch.py "关键词"
D:/tool/py/python.exe ~/.claude/scripts/websearch.py -n 10 "关键词"
D:/tool/py/python.exe ~/.claude/scripts/websearch.py --news "关键词"
D:/tool/py/python.exe ~/.claude/scripts/websearch.py --json "关键词"
```

支持 `-n`（条数）、`--news`（时效新闻）、`--json`（结构化输出）、`--backend`（搜索引擎选择）。

### Hook 系统（Claude Code 侧）

| Hook | 触发时机 | 作用 |
|---|---|---|
| `pre-compact-hook.py` | 压缩前 | 更新 STATE.md 项目接力档案 |
| `session-start-hook.py` | 会话启动 | 注入当前目录/STATE.md + 记忆索引 |
| `context-monitor-hook.py` | 用户输入 | 检查上下文水位 |
| `error-recovery-hook.py` | 错误通知 | 触发六步自修复指令 |
| `session-memory-hook.cjs` | 压缩/清除 | 记录会话摘要 |
| `observation-logger.cjs` | 工具调用后 | 记录关键观察 |
| `statusline-wrapper.js` | 状态栏 | 修正上下文百分比显示 |

### 长期记忆系统

**位置：**`~/.claude/projects/C--Users----/memory/`

基于 Markdown 的长期记忆，由 `MEMORY.md` 索引。分类：
- `user_*.md` — 用户档案
- `feedback_*.md` — 踩坑经验（42 条）
- `project_*.md` — 项目状态
- `reference_*.md` — 配置参考
- `interactions/` — 动态交互数据（答题记录、学习进度）

### AGENTS.md / CLAUDE.md

注入 system context 的全局 Agent 指令：

- `~/.codex/AGENTS.md` — Codex 专用：shell 优先级（直接调 exe > bash > powershell）、GPT-5.5 字符串格式命令、websearch 替代
- `~/.claude/CLAUDE.md` — Claude 专用：零署名策略、编码原则、错误处理、STATE.md 接力

---

## 已知问题与修复

### Clash 劫持 localhost（问题 #1）
**现象：**`curl 127.0.0.1:9100` 返回 502。
**诊断：**`curl -v` 显示 `Trying 127.0.0.1:7897`。
**修复：**在四处设 `NO_PROXY=127.0.0.1,localhost,::1`：setx（系统级）、bashrc、PowerShell profile、启动脚本。同时清空 `HTTP_PROXY`。

### Codex 0.128 禁用 wire_api=chat（问题 #2）
**现象：**`wire_api=chat` 启动报错。
**修复：**必须用 `wire_api=responses`，本地 proxy 是唯一通路。

### httpx 流式 + DeepSeek 多字节字符断连（问题 #3）
**现象：**`peer closed connection without sending complete message body`
**根因：**httpx 0.28 对 chunked tail decode 失败，多字节字符触发。
**修复：**有 tools 时切换 requests 非流式；无 tools 时断开自动回退。

### DeepSeek 流式带工具参数断连（问题 #4）
**修复：**有 tools 时强制非流式。

### 上游随机 ChunkedEncodingError（问题 #5）
**修复：**proxy 内 3 次重试 + 1/2/3 秒退避。

### Codex 对非 OpenAI provider 不下发 reasoning.effort（问题 #6）
**修复：**伪装成 gpt-5.5（Codex 内置 metadata）。proxy 默认对 pro 启 thinking。

### pythonw 启动 stdout=None 崩溃（问题 #7）
**修复：**proxy 启动时检测并重定向到 proxy.log。

### Codex 默认 PowerShell 行为（问题 #8）
**修复：**AGENTS.md 明确 shell 优先级（exe > bash > powershell），实测覆盖默认行为。

### 流式 done 事件空文本（问题 #9）
**修复：**`done` 事件必须带 accumulated 文本。

### DeepSeek thinking 模式要求历史 reasoning_content（问题 #10）
**修复：**convert_request 缓冲 reasoning 文本挂到下一个 assistant 消息。

### 流式状态机缩进 bug（问题 #11）
**修复：**条件判断移到 for 循环内正确层级。

### shell_command 参数格式不匹配（问题 #12）
**修复：**coerce_tool_args 处理 array→string 转换。

### 并行工具调用拆分导致 DeepSeek 400（问题 #13）
**修复：**buffer 连续 function_call 合并为单个 assistant 消息。

### 上下文百分比虚高（问题 #14）
**修复：**statusline-wrapper.js 按模型名改写窗口大小。

### CLAUDE_CODE_CONTEXT_WINDOW 无效（问题 #15）
**修复：**删除无效变量，用 statusline-wrapper 修正显示，autoCompactThreshold 控制实际压缩。

### mydamoxing glm-5.1 仅 OpenAI 端点（问题 #16）
**修复：**经 7861 代理做 Anthropic→OpenAI 转换。

### qwen2API 必须走原生 Anthropic 端点（问题 #17）
**修复：**Claude Code 直连 `:7860/anthropic/v1/messages`。

---

## 配置文件速查

### 必需环境变量

```bash
# 系统级（setx）
NO_PROXY=127.0.0.1,localhost,::1

# Claude Code（settings.json env 或 SwitchModel 设）
ANTHROPIC_BASE_URL=http://127.0.0.1:7861
ANTHROPIC_AUTH_TOKEN=<你的key>
ANTHROPIC_MODEL=glm-5.1
CLAUDE_CODE_MAX_OUTPUT_TOKENS=32768

# Codex（config.env）
OPENAI_API_KEY=<占位>
DEEPSEEK_API_KEY=<你的key>
NO_PROXY=127.0.0.1,localhost
```

### 文件地图

```
~/
├── .claude/
│   ├── CLAUDE.md                    # 全局 Agent 规则
│   ├── settings.json                # Claude Code 配置
│   ├── glm-settings.json            # GLM 默认模型配置
│   ├── scripts/
│   │   ├── switch.ps1               # SwitchModel 函数
│   │   ├── websearch.py             # 网页搜索替代
│   │   ├── pre-compact-hook.py      # 压缩前更新 STATE.md
│   │   ├── session-start-hook.py    # 注入记忆索引
│   │   ├── context-monitor-hook.py  # 上下文水位检查
│   │   ├── error-recovery-hook.py   # 错误触发修复
│   │   └── statusline-wrapper.js    # 上下文百分比修正
│   ├── session-memory/
│   │   ├── session-memory-hook.cjs  # 会话摘要记录
│   │   └── observation-logger.cjs   # 工具观察记录
│   └── projects/C--Users----/memory/
│       ├── MEMORY.md                # 记忆索引
│       └── ...                      # 记忆文件
├── .codex/
│   ├── config.toml                  # Codex 模型/Provider 配置
│   ├── config.env                   # API key + NO_PROXY
│   ├── AGENTS.md                    # 全局 Agent 规则
│   ├── deepseek_proxy.py            # Responses→Chat Completions 代理
│   ├── start_proxy.bat              # 启动代理（pythonw）
│   ├── start_codex.bat              # 启动 Codex（含代理）
│   ├── last_inbound.json            # 调试：Codex 原始请求
│   ├── last_request.json            # 调试：转换后请求
│   └── proxy.log                    # 调试：代理日志
└── Desktop/qwen2api/
    ├── anthropic-proxy.mjs          # 统一上下文代理
    └── start_proxy.bat              # 带 NO_PROXY 启动
```

---

## 调试

### Codex + DeepSeek

| 检查项 | 命令 |
|---|---|
| 代理是否在跑？ | `netstat -ano | findstr ":9100"` |
| 直测代理 | `curl --noproxy 127.0.0.1 http://127.0.0.1:9100/v1/models` |
| 直测 DeepSeek | `curl https://api.deepseek.com/v1/models -H "Authorization: Bearer <key>"` |
| 查看 Codex 原始请求 | `cat ~/.codex/last_inbound.json` |
| 查看转换后请求 | `cat ~/.codex/last_request.json` |
| 查看代理日志 | `cat ~/.codex/proxy.log` |
| 重启代理 | `taskkill /F /PID <pid>` 然后 `cmd /c ~/.codex/start_proxy.bat` |

### Claude + 多模型

| 检查项 | 命令 |
|---|---|
| 记忆索引 | `cat ~/.claude/projects/C--Users----/memory/MEMORY.md` |
| qwen2API 管理台 | `http://127.0.0.1:7860` |
| 当前模型 | `SwitchModel status` |
| 全局配置 | `cat ~/.claude/glm-settings.json` |

---

## License

MIT
