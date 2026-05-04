# Codex + DeepSeek / Claude + 多模型接入方案

## 总体架构

```
                        ┌─────────────────────────────┐
                        │       Claude Code CLI        │
                        │  (Anthropic Messages API)    │
                        └──────────────┬──────────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │  anthropic-proxy.mjs :7861   │
                        │  统一上下文管理代理            │
                        │  三层防御 (虚报tokens/错误拦截/   │
                        │  请求预检)                    │
                        └──┬─────────┬─────────┬──────┘
                           │         │         │
              ┌────────────▼──┐ ┌────▼──────┐ ┌▼──────────────┐
              │ mydamoxing.cn │ │DeepSeek   │ │ qwen2API :7860 │
              │  GLM-5.1      │ │V4 Pro/    │ │ Qwen 3.6 Plus  │
              │  Anthropic    │ │Flash      │ │ 原生Anthropic  │
              └───────────────┘ └───────────┘ └───────────────┘


                        ┌─────────────────────────────┐
                        │       Codex CLI              │
                        │  (Responses API)             │
                        └──────────────┬──────────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │  deepseek_proxy.py :9100     │
                        │  Responses→Chat Completions  │
                        └──────────────┬──────────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │  api.deepseek.com            │
                        │  DeepSeek V4 Pro             │
                        └─────────────────────────────┘
```

环境整体约束：Clash Verge 运行在 `127.0.0.1:7897`，默认劫持所有 HTTP 流量。所有本地代理（7860/7861/9100）必须通过 `NO_PROXY=127.0.0.1,localhost,::1` 排除，否则 Clash 返回 502。

---

## 一、Claude Code 侧：多模型接入

### 1.1 本地代理 anthropic-proxy.mjs（端口 7861）

位置：`~/Desktop/qwen2api/anthropic-proxy.mjs`

统一上下文管理代理，三个模型全部通过 127.0.0.1:7861 接入 Claude Code。核心功能：

- **模型路由**：根据 ANTHROPIC_MODEL 分发到不同上游
  - `glm-5.1` → mydamoxing.cn `/v1/messages`（Anthropic 直透）
  - `glm-4.7` → mydamoxing.cn `/v1/messages`（Anthropic 直透）
  - `deepseek-v4-pro/flash` → api.deepseek.com `/anthropic/v1/messages`
  - `qwen3.6-plus` → qwen2API `:7860/v1/chat/completions`（Anthropic→OpenAI 转换）

- **三层防御上下文管理**：
  1. `count_tokens` 虚报：超过模型窗口 80% 时虚报到 180K~200K，触发 Claude Code 自动 compact
  2. 错误拦截：上游 context limit 错误转为 Anthropic 标准格式
  3. 请求预检：超过模型实际窗口直接 400 拒绝

### 1.2 qwen2API Docker 容器（端口 7860）

- 镜像：YuJunZhiXue/qwen2API
- 管理台：`http://127.0.0.1:7860`，`ADMIN_KEY=qwen2api-admin-2026`
- 暴露三套端点：
  - `/v1/chat/completions` — OpenAI 格式
  - `/anthropic/v1/messages` — Anthropic 原生（工具调用优化：schema 压缩、tool name 混淆、few-shot 注入）
  - `/v1beta/models/...` — Gemini 格式
- Docker 关键配置：MAX_INFLIGHT=2, MAX_RETRIES=5, ACCOUNT_MIN_INTERVAL_MS=600, REQUEST_JITTER 120~360ms, shm_size=512m

### 1.3 mydamoxing.cn 中转站

- URL：https://mydamoxing.cn
- 可用模型（Anthropic 端点）：glm-4.7（带 thinking）、glm-5
- glm-5.1 仅提供 OpenAI 端点，需经 7861 做 Anthropic→OpenAI 转换
- 模型名为中转站自创，非智谱官方命名

### 1.4 模型切换：SwitchModel

脚本：`~/.claude/scripts/switch.ps1`，注册为 PowerShell 7 函数。仅影响当前终端会话（设置进程级 `$env:ANTHROPIC_*`），不写 settings.json。关闭终端自动恢复默认。

```powershell
SwitchModel glm            # GLM 5.1（mydamoxing 中转）
SwitchModel deepseek-pro   # DeepSeek V4 Pro
SwitchModel deepseek-flash # DeepSeek V4 Flash
SwitchModel qwen           # Qwen 3.6 Plus（本地 qwen2API:7860/anthropic 原生端点）
SwitchModel claude         # 清除覆盖，回退 settings.json 默认
SwitchModel status         # 查看当前会话模型
```

### 1.5 Claude Code settings.json 关键配置

```json
{
  "permissions": { "defaultMode": "bypassPermissions" },
  "model": "opus[1m]",
  "provider": "anthropic",
  "defaultAgent": "reasoner",
  "autoCompact": true,
  "autoCompactThreshold": 0.7,
  "maxTokens": 32768,
  "hooks": {
    "PreCompact": [{ "matcher": "auto", "hooks": [{ "type": "command", "command": "python ...pre-compact-hook.py" }] }],
    "SessionStart": [...],
    "UserPromptSubmit": [{ "hooks": [{ "type": "command", "command": "python ...context-monitor-hook.py" }] }],
    "PostToolUse": [{ "hooks": [{ "type": "command", "command": "node ...observation-logger.cjs" }] }],
    "Notification": [{ "matcher": "error", "hooks": [{ "type": "command", "command": "python ...error-recovery-hook.py" }] }]
  },
  "statusLine": { "type": "command", "command": "...statusline-wrapper.js" }
}
```

glm-settings.json（全局默认，走 7861 统一代理）：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:7861",
    "ANTHROPIC_MODEL": "glm-5.1",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-5.1",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-5.1",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5.1",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "32768",
    "MAX_MCP_OUTPUT_TOKENS": "32768",
    "ANTHROPIC_MAX_OUTPUT_TOKENS": "32768"
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

### 1.6 上下文百分比修正（statusline-wrapper.js）

Claude Code CLI 对非官方模型一律按 200K 算分母，导致百分比虚高。`~/.claude/scripts/statusline-wrapper.js` 在 stdin JSON 进 claude-hud 前根据 `model.display_name` 改写 `context_window.context_window_size`：DeepSeek/GLM/Qwen → 1M，Kimi → 256K。

---

## 二、Codex CLI 侧：DeepSeek 接入

### 2.1 总体原理

Codex CLI 0.128 禁用了 `wire_api=chat`，只能走 Responses API。DeepSeek 没有 Responses 端点，所以起本地代理做协议转换。模型伪装成 `gpt-5.5`（Codex 内置其完整 metadata，reasoning.effort 正确透传，shell 工具变 `shell_command`），proxy 映射回 `deepseek-v4-pro` 发给上游。

### 2.2 核心文件

#### deepseek_proxy.py（~520 行）

位置：`~/.codex/deepseek_proxy.py`

技术栈：Flask + waitress（pythonw 后台无窗口运行），httpx 流式 + requests 非流式双栈。

关键功能：

- **convert_request(data)**：Responses API → Chat Completions
  - 跳过 reasoning input items，但 buffer 其 summary text
  - 下一个 assistant message 时挂上 `reasoning_content`（DeepSeek thinking 模式硬要求）
  - 并行 function_call items buffer 后合并为一个 assistant 消息（避免 DeepSeek 400 错误）
  - thinking 字段：flash 永远 disabled；pro 默认 enabled，仅 effort=minimal 时关

- **coerce_tool_args(name, args_str)**：工具参数双向转换
  - `shell_command`：array → string（GPT-5.5 要求 command 是字符串）
  - `apply_patch`：JSON → 裸 patch 文本
  - `shell`/`local_shell`：string → array

- **fetch_chunks()**：两种请求路径
  - tools 在场 → requests 非流式 + 3 次重试 → 合成单个 fake chunk
  - 无 tools → httpx 真流式，连接断开时回退到非流式

- **主 generate() 状态机**：懒创建 reasoning/message/tool_call SSE 事件，reasoning 先 close 再开 message（顺序合法），所有 done 事件带 accumulated 文本

- **pythonw 兼容**：启动时判 `sys.stdout is None` 自动重定向到 `proxy.log`

#### config.toml

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

[projects.'C:\Users\曾金昌']
trust_level = "trusted"
```

### 2.3 启动脚本

#### start_proxy.bat

```batch
@echo off
netstat -ano | findstr ":9100" | findstr LISTENING >nul 2>&1
if %ERRORLEVEL% EQU 0 exit /b 0
set HTTP_PROXY=
set HTTPS_PROXY=
set http_proxy=
set https_proxy=
start "" /B "D:\tool\py\pythonw.exe" "C:\Users\曾金昌\.codex\deepseek_proxy.py" 9100
exit /b 0
```

#### start_codex.bat

```batch
@echo off
chcp 65001 >nul
call "C:\Users\曾金昌\.codex\start_proxy.bat"
set HTTP_PROXY=
set HTTPS_PROXY=
set http_proxy=
set https_proxy=
set NO_PROXY=127.0.0.1,localhost,::1
set no_proxy=127.0.0.1,localhost,::1
codex %*
exit /b %ERRORLEVEL%
```

#### 开机自启

`%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\codex-deepseek-proxy.bat` 调用 start_proxy.bat。

### 2.4 AGENTS.md（全局 Agent 守则）

位置：`~/.codex/AGENTS.md`

Codex 自动注入到 system context。核心规约：

- 零 AI 痕迹（交付物禁止任何模型名/人名/署名/套路措辞）
- Shell 选择优先级：直接调 exe > bash -lc > powershell（覆盖 Codex 工具描述里的 powershell 默认）
- GPT-5.5 的 `shell_command` 用字符串格式（如 `"git status"`），非数组
- WebSearch 替代：`D:/tool/py/python.exe C:/Users/曾金昌/.claude/scripts/websearch.py -n 5 "关键词"`
- 长期记忆指针：`~/.claude/projects/C--Users----/memory/`
- 错误处理：六步循环（读错误 → 定位 → 根因 → 修改 → 验证 → 终止），遇错切换策略

---

## 三、共用基础设施

### 3.1 Clash 防劫持：NO_PROXY

Clash 运行在 `127.0.0.1:7897`，默认劫持所有 HTTP。本地代理（7860/7861/9100）必须排除。四处都需要设：

1. **全局环境变量（setx）**：`NO_PROXY=127.0.0.1,localhost,::1`
2. **PowerShell Profile**：`$env:NO_PROXY = "127.0.0.1,localhost,::1"`
3. **bashrc**：`export NO_PROXY="127.0.0.1,localhost,::1"`
4. **启动脚本**：`set NO_PROXY=...` + `set HTTP_PROXY=`（清空代理）

### 3.2 WebSearch 替代脚本

位置：`~/.claude/scripts/websearch.py`

Claude Code 原生 WebSearch 工具仅 Anthropic 官方模型可用。GLM/DeepSeek/Qwen 统一走此脚本：

```bash
D:/tool/py/python.exe C:/Users/曾金昌/.claude/scripts/websearch.py "关键词"
D:/tool/py/python.exe C:/Users/曾金昌/.claude/scripts/websearch.py -n 10 "关键词"
D:/tool/py/python.exe C:/Users/曾金昌/.claude/scripts/websearch.py --news "关键词"
D:/tool/py/python.exe C:/Users/曾金昌/.claude/scripts/websearch.py --json "关键词"
```

### 3.3 长期记忆系统

位置：`~/.claude/projects/C--Users----/memory/`

- `MEMORY.md`：索引文件，按类别组织（User/Feedback/Project/Reference）
- `user_*.md`：用户档案
- `feedback_*.md`：踩坑经验（42 个）
- `project_*.md`：项目状态
- `reference_*.md`：配置参考
- `interactions/`：动态交互数据（答题记录、学习进度）

### 3.4 跨工具状态接力（STATE.md）

约定：项目根 STATE.md 是跨会话接力档案。SessionStart hook 注入当前 STATE.md，PreCompact hook 指令先更新再压缩。适用场景：长项目、里程碑、技术决策、`/compact` 前。

### 3.5 Hook 体系（Claude Code 侧）

- `pre-compact-hook.py`：压缩前更新 STATE.md
- `session-start-hook.py`：会话启动注入 CWD/STATE.md + 记忆索引
- `context-monitor-hook.py`：每次用户输入检查上下文水位
- `error-recovery-hook.py`：错误通知触发六步自修复指令
- `session-memory-hook.cjs`：压缩/清除时记录会话摘要
- `observation-logger.cjs`：每次工具调用后记录关键观察
- `statusline-wrapper.js`：修正上下文百分比显示

---

## 四、已知坑及修复（按发现顺序）

### 4.1 Clash 劫持 localhost

- 现象：curl 127.0.0.1 返回 502
- 诊断：`curl -v` 看 `Trying 127.0.0.1:7897` 即命中
- 修复：NO_PROXY 四处置（setx + bashrc + pwsh profile + 启动脚本），HTTP_PROXY 清空

### 4.2 Codex 0.128 禁用 wire_api=chat

- 现象：配置 `wire_api=chat` 启动报错
- 修复：必须 responses，本地 proxy 是唯一通路

### 4.3 httpx 流式 + DeepSeek 中文断连

- 现象：`peer closed connection without sending complete message body`（RemoteProtocolError）
- 根因：httpx 0.28 对 chunked tail decode 失败，多字节字符触发
- 修复：tools 在场时切 requests 非流式；无 tools 时流式断连自动回退非流式

### 4.4 DeepSeek 流式带 tool args 断连

- 现象：上游 server 自己断（curl 也复现）
- 修复：tools 在场强制非流式

### 4.5 DeepSeek 上游随机 ChunkedEncodingError

- 现象：偶发
- 修复：proxy 内 3 次重试 + 1/2/3 秒退避

### 4.6 Codex 对非 OpenAI provider 不下发 reasoning.effort

- 现象：config.toml 设了 `model_reasoning_effort=high` 但 request 里没有
- 修复方案一：伪装成 gpt-5.5，Codex 识别内置 metadata 后正确透传
- 修复方案二：proxy 默认对 pro 启用 thinking，仅 effort=minimal 时关

### 4.7 pythonw 启动 stdout=None

- 现象：waitress/Flask 内部 print 崩溃
- 修复：proxy 顶部判 `sys.stdout is None` 自动重定向到 proxy.log

### 4.8 Codex shell 工具默认行为是 powershell

- 现象：模型按工具描述走 powershell，启动慢、unix 命令笨重
- 修复：AGENTS.md 写明 shell 优先级（直接 exe > bash > powershell）

### 4.9 流式 done 事件文本写死空串

- 现象：Codex TUI 显示空白但 token 计数正常
- 修复：`response.content_part.done` / `response.output_item.done` 必须带 accumulated 文本

### 4.10 DeepSeek thinking 模式要求历史 reasoning_content 回传

- 现象：`The reasoning_content in the thinking mode must be passed back to the API`
- 修复：convert_request 把 Codex 回灌的 reasoning item summary 文本挂到下一个 assistant message 的 `reasoning_content` 字段

### 4.11 流式状态机缩进 bug

- 现象：`if not response_started:` 在 for 循环外，chunk 全部嵌套在 if 内，只有最后一个生效 → 流输出空白
- 修复：移到 for 循环内正确层级

### 4.12 shell_command 参数格式不匹配

- 现象：模型输出数组 `["bash","-lc","..."]`，GPT-5.5 的 shell_command 要求字符串 → Codex 报 `failed to parse: expected a string`
- 修复：coerce_tool_args 加 shell_command 分支，array → string

### 4.13 并行工具调用拆分导致 DeepSeek 400

- 现象：convert_request 把每个 function_call 转成独立 assistant 消息，8 个并行调用变 8 个连续 assistant → DeepSeek 报 `insufficient tool messages` → Codex 收到空响应当作任务完成
- 修复：buffer 连续 function_call items，flush 时合并为一个 assistant 消息；非流式错误路径返回 `{"error": ...}` JSON

### 4.14 Claude Code 上下文百分比虚高

- 现象：状态栏百分比涨太快
- 根因：CLI 对代理模型按 200K 算分母，实际窗口 1M
- 修复：statusline-wrapper.js 按模型名改写 context_window_size

### 4.15 CLAUDE_CODE_CONTEXT_WINDOW 无效

- 现象：设了该 env 无效
- 根因：这不是 Claude Code 官方支持的变量，CLI 按内置规则判窗口
- 修复：删掉无效 env，由 statusline-wrapper.js 修正显示，autoCompactThreshold 控制实际压缩

### 4.16 mydamoxing glm-5.1 仅 OpenAI 端点

- 现象：Claude Code 直接配 glm-5.1 + mydamoxing Anthropic 端点不工作
- 修复：走 7861 统一代理做 Anthropic→OpenAI 转换

### 4.17 qwen2API 必须走原生 Anthropic 端点

- 现象：走 7861 做 Anthropic→OpenAI 转换时 Qwen 返回 0 token，没有工具调用
- 根因：绕过 qwen2API 自带的 Anthropic 优化代码（schema 压缩、tool name 混淆、few-shot 注入）
- 修复：Claude Code 直连 7860/anthropic 原生端点

---

## 五、调试接口

### Codex + DeepSeek

| 入口 | 说明 |
|---|---|
| `~/.codex/last_inbound.json` | Codex 发到 proxy 的原始请求 |
| `~/.codex/last_request.json` | proxy 转换后发给 DeepSeek 的请求 |
| `~/.codex/proxy.log` | pythonw 模式下的 stdout/stderr |
| `curl --noproxy 127.0.0.1 http://127.0.0.1:9100/v1/models` | 直测 proxy |
| `curl https://api.deepseek.com/v1/models -H "Authorization: Bearer <key>"` | 直测 DeepSeek |
| 重启 proxy | `taskkill /F /PID <9100占用>` 然后 `cmd /c ~/.codex/start_proxy.bat` |

### Claude + 多模型

| 入口 | 说明 |
|---|---|
| `~/.claude/projects/C--Users----/memory/MEMORY.md` | 记忆索引 |
| `http://127.0.0.1:7860` | qwen2API 管理台 |
| `~/.claude/glm-settings.json` | Claude Code 全局模型默认配置 |
| `SwitchModel status` | 查看当前会话模型 |

---

## 六、环境依赖

| 工具 | 版本/路径 | 用途 |
|---|---|---|
| Python | 3.12.10 (`D:/tool/py/python.exe`) | websearch、hook 脚本、deepseek_proxy |
| Node.js | 24.11.1 | anthropic-proxy.mjs、session-memory hook |
| Git | 2.53.0 | 版本控制 |
| ripgrep | 15.1.0 | 文件搜索 |
| Docker | - | qwen2API 容器 |
| Clash Verge | 127.0.0.1:7897 | 网络代理（需 NO_PROXY 排除本地） |
