<div align="center">

# Codex CLI 完全配置手册

**OpenAI Codex CLI 从安装到精通 —— config.toml 全字段详解、多 Provider 配置、MCP、成本控制、报错排查**

[![Codex](https://img.shields.io/badge/Codex_CLI-手册-412991?style=flat-square&logo=openai&logoColor=white)](https://github.com/leonis-ai/codex-cli-guide)
[![License](https://img.shields.io/badge/License-MIT-2ea44f?style=flat-square)](LICENSE)
[![中文](https://img.shields.io/badge/文档-简体中文-1a73e8?style=flat-square)](README.md)

</div>

---

## 目录

- [Codex CLI 是什么](#codex-cli-是什么)
- [安装](#安装)
- [第一次配置](#第一次配置)
- [config.toml 全字段详解](#configtoml-全字段详解)
- [接入第三方网关](#接入第三方网关)
- [多 Provider 并存与切换](#多-provider-并存与切换)
- [审批模式与沙箱](#审批模式与沙箱)
- [MCP 服务器](#mcp-服务器)
- [AGENTS.md 项目记忆](#agentsmd-项目记忆)
- [成本控制](#成本控制)
- [报错排查](#报错排查)
- [和 Claude Code 怎么选](#和-claude-code-怎么选)

---

## Codex CLI 是什么

OpenAI 官方的终端编程 Agent。跑在命令行里，能读写项目文件、执行命令、跑测试、提交 Git。

和 Claude Code 是同一类工具，差别主要在协议和生态：

| | Codex CLI | Claude Code |
|---|---|---|
| 厂商 | OpenAI | Anthropic |
| 协议 | OpenAI Responses / Chat | Anthropic Messages |
| 配置文件 | `~/.codex/config.toml` | 环境变量 + `settings.json` |
| 项目记忆 | `AGENTS.md` | `CLAUDE.md` |
| 沙箱 | 内置多级沙箱 | 权限模式 |

---

## 安装

### npm

```bash
npm install -g @openai/codex
```

### Homebrew

```bash
brew install codex
```

### 验证

```bash
codex --version
```

### 升级

```bash
npm update -g @openai/codex
```

---

## 第一次配置

### 方式一：官方登录

```bash
codex
# 首次运行会引导浏览器登录 ChatGPT 账号
```

需要 ChatGPT Plus / Pro / Business 订阅，或 API Key 余额。

### 方式二：API Key

```bash
export OPENAI_API_KEY="sk-your-key"
codex
```

### 方式三：第三方网关（本文重点）

见 [接入第三方网关](#接入第三方网关)。

---

## config.toml 全字段详解

配置文件位置：`~/.codex/config.toml`（Windows：`%USERPROFILE%\.codex\config.toml`）

### 顶层字段

```toml
# ── 模型与供应商 ──────────────────────────────
model = "gpt-5.6-sol"              # 默认模型
model_provider = "leonis"          # 使用哪个 provider（对应下面的 [model_providers.*]）
model_reasoning_effort = "medium"  # 推理强度: minimal | low | medium | high
model_reasoning_summary = "auto"   # 推理摘要: auto | concise | detailed | none
model_verbosity = "medium"         # 输出详细度: low | medium | high

# ── 审批与沙箱 ────────────────────────────────
approval_policy = "on-request"     # untrusted | on-failure | on-request | never
sandbox_mode = "workspace-write"   # read-only | workspace-write | danger-full-access

# ── 界面 ─────────────────────────────────────
hide_agent_reasoning = false       # 隐藏推理过程
show_raw_agent_reasoning = false   # 显示原始推理链
file_opener = "vscode"             # 点击文件路径用什么打开: vscode | cursor | windsurf | none

# ── 历史与遥测 ────────────────────────────────
disable_response_storage = false   # 关闭服务端响应存储（第三方网关建议开启此项为 true）

[history]
persistence = "save-all"           # save-all | none

# ── 网络 ─────────────────────────────────────
[tools]
web_search = false                 # 是否允许联网搜索
```

### `model_reasoning_effort` 怎么选

| 值 | 适用 | 成本 |
|---|---|---|
| `minimal` | 简单改写、格式转换 | 最低 |
| `low` | 常规编码任务 | 低 |
| `medium` | 默认，多数场景 | 中 |
| `high` | 复杂架构、疑难调试 | 高（推理 Token 显著增加） |

> 💡 推理 Token 是按 Output 价格计费的。`high` 和 `low` 的账单能差 3 倍以上，别无脑开 `high`。

### `sandbox_mode` 三档

| 值 | 权限 | 适用 |
|---|---|---|
| `read-only` | 只读，不能改文件也不能执行命令 | 让它先读懂项目 |
| `workspace-write` | 可改工作区文件、可执行命令，网络受限 | **日常推荐** |
| `danger-full-access` | 完全放开 | 只在容器 / 一次性环境里用 |

### `approval_policy` 四档

| 值 | 行为 |
|---|---|
| `untrusted` | 只有明确可信的命令自动执行，其余都问 |
| `on-failure` | 先在沙箱里跑，失败了才问要不要提权 |
| `on-request` | 模型自己判断要不要申请提权（**默认，推荐**） |
| `never` | 从不询问，全自动（危险，仅限隔离环境） |

---

## 接入第三方网关

Codex CLI 通过 `[model_providers.*]` 支持任意 OpenAI 兼容端点。

### 最小可用配置

`~/.codex/config.toml`：

```toml
model = "gpt-5.6-sol"
model_provider = "leonis"

[model_providers.leonis]
name = "Leonis AI"
base_url = "https://ai.svtun.cn/v1"
wire_api = "responses"
env_key = "LEONIS_API_KEY"
```

设置环境变量：

```bash
# macOS / Linux
echo 'export LEONIS_API_KEY="sk-your-key"' >> ~/.zshrc
source ~/.zshrc

# Windows PowerShell
[Environment]::SetEnvironmentVariable("LEONIS_API_KEY", "sk-your-key", "User")
```

启动：

```bash
codex
```

### `wire_api` 该填什么

这是**最容易配错的字段**：

| 值 | 含义 | 什么时候用 |
|---|---|---|
| `responses` | OpenAI Responses API | 网关支持 `/v1/responses`（**推荐**，功能最全） |
| `chat` | OpenAI Chat Completions | 网关只支持 `/v1/chat/completions` |

填错的表现：`404 Not Found` 或 `Unsupported endpoint`。不确定就先试 `responses`，报 404 再改 `chat`。

### 完整字段说明

```toml
[model_providers.leonis]
name = "Leonis AI"                        # 显示名，随便起
base_url = "https://ai.svtun.cn/v1"   # 网关地址，注意结尾的 /v1
wire_api = "responses"                    # responses | chat
env_key = "LEONIS_API_KEY"                # 从哪个环境变量读 Key
query_params = {}                         # 附加 URL 参数（Azure 需要）
http_headers = { "X-Custom" = "value" }   # 附加请求头
env_http_headers = { "X-Trace" = "TRACE_ID" }  # 从环境变量读的请求头
request_max_retries = 4                   # 请求失败重试次数
stream_max_retries = 5                    # 流式中断重试次数
stream_idle_timeout_ms = 300000           # 流式空闲超时(毫秒)
```

### 用 Claude 模型跑 Codex CLI

网关支持协议转换的话，可以在 Codex CLI 里直接用 Claude：

```toml
[model_providers.leonis]
name = "Leonis AI"
base_url = "https://ai.svtun.cn/v1"
wire_api = "chat"
env_key = "LEONIS_API_KEY"
```

```bash
codex --model claude-sonnet-5
```

> ⚠️ 注意：协议转换会损失部分 Anthropic 原生特性（thinking 块、缓存控制标记）。
> 深度使用 Claude 建议直接用 [Claude Code](https://github.com/leonis-ai/claude-code-guide)。

---

## 多 Provider 并存与切换

`config.toml` 里可以同时定义多个 provider：

```toml
model = "gpt-5.6-sol"
model_provider = "leonis"

# 第三方网关
[model_providers.leonis]
name = "Leonis AI"
base_url = "https://ai.svtun.cn/v1"
wire_api = "responses"
env_key = "LEONIS_API_KEY"

# 官方
[model_providers.openai]
name = "OpenAI Official"
base_url = "https://api.openai.com/v1"
wire_api = "responses"
env_key = "OPENAI_API_KEY"

# 本地 Ollama
[model_providers.ollama]
name = "Ollama Local"
base_url = "http://localhost:11434/v1"
wire_api = "chat"
```

### 临时切换

```bash
codex --config model_provider=openai
codex --config model_provider=ollama --model qwen3:32b
```

### 用 Profile 打包一组配置

```toml
[profiles.work]
model = "gpt-5.6-sol"
model_provider = "leonis"
model_reasoning_effort = "medium"
approval_policy = "on-request"

[profiles.deep]
model = "gpt-5.6"
model_provider = "leonis"
model_reasoning_effort = "high"
approval_policy = "on-failure"

[profiles.cheap]
model = "gpt-5.4-mini"
model_provider = "leonis"
model_reasoning_effort = "minimal"
```

```bash
codex --profile work
codex --profile deep
codex --profile cheap
```

> 🔀 **想要图形界面一键切换？** 见 [cc-switch-guide](https://github.com/leonis-ai/cc-switch-guide) ——
> 用 cc-switch 桌面工具管理 Claude Code 和 Codex 的多套配置，点一下就切换，不用手改 toml。

---

## 审批模式与沙箱

### 常用组合

**日常开发（推荐）：**

```toml
approval_policy = "on-request"
sandbox_mode = "workspace-write"
```

**只读分析（让它先读懂项目）：**

```bash
codex --config sandbox_mode=read-only
```

**全自动（仅限容器）：**

```bash
codex --dangerously-bypass-approvals-and-sandbox
```

### 命令行快捷开关

```bash
codex --full-auto           # 等价于 on-failure + workspace-write
codex --ask-for-approval never
codex --sandbox read-only
```

---

## MCP 服务器

Codex CLI 支持 MCP（Model Context Protocol）扩展工具能力。

```toml
[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed"]

[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { GITHUB_PERSONAL_ACCESS_TOKEN = "ghp_xxx" }

[mcp_servers.postgres]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
```

配置后重启 Codex，工具会自动注册。用 `/mcp` 查看已加载的服务器。

---

## AGENTS.md 项目记忆

在项目根目录建 `AGENTS.md`，Codex 每次启动都会读。作用等同于 Claude Code 的 `CLAUDE.md`。

```markdown
# 项目说明

## 技术栈
Next.js 15 (App Router) + TypeScript + Tailwind + Prisma + PostgreSQL

## 目录约定
- `src/app/` 路由
- `src/components/ui/` shadcn 基础组件，不要手改
- `src/lib/` 工具函数

## 编码规范
- 组件用函数式 + 具名导出
- 数据获取一律 Server Component
- 提交前必须跑 `pnpm lint && pnpm typecheck`

## 命令
- 开发：`pnpm dev`
- 测试：`pnpm test`
- 构建：`pnpm build`

## 禁止
- 不要改 `src/components/ui/`
- 不要引入新的状态管理库
- 不要在没测试的情况下改 `src/lib/billing/`
```

### 层级查找

Codex 会从当前目录向上查找并合并：

```
~/.codex/AGENTS.md          全局
../AGENTS.md                上级目录
./AGENTS.md                 项目根
./packages/api/AGENTS.md    子包（monorepo）
```

---

## 成本控制

### 推理强度是最大的变量

```bash
codex --config model_reasoning_effort=low      # 日常
codex --config model_reasoning_effort=high     # 只在真需要时
```

推理 Token 按 Output 价格计费。同一个任务，`high` 相比 `low` 账单差 3 倍以上是常态。

### 关掉不用的功能

```toml
tools = { web_search = false }      # 联网搜索会显著增加上下文
hide_agent_reasoning = true         # 不显示不代表不计费，但能减少干扰
disable_response_storage = true     # 第三方网关必开，否则可能报错
```

### 及时开新会话

```bash
/new       # 清空上下文，开新会话
/compact   # 压缩历史
```

上下文是累积计费的。任务切换时开新会话，成本立刻回到起点。

### 用 `.gitignore` 之外的排除

Codex 默认尊重 `.gitignore`。把大文件、构建产物、依赖目录都排除掉：

```gitignore
node_modules/
dist/
.next/
coverage/
*.min.js
*.map
public/assets/
```

### 查用量

```bash
/status      # 当前会话的 Token 用量
```

📖 更细的成本分析见 [ai-api-pricing](https://github.com/leonis-ai/ai-api-pricing)。

---

## 报错排查

### `404 Not Found`

**最常见原因：`wire_api` 填错。**

```toml
wire_api = "responses"   # 先试这个
wire_api = "chat"        # 报 404 就改这个
```

**第二常见：`base_url` 结尾少了或多了 `/v1`。**

```toml
base_url = "https://ai.svtun.cn/v1"   # ✅
base_url = "https://ai.svtun.cn"      # ❌ 少了 /v1
base_url = "https://ai.svtun.cn/v1/"  # ❌ 多了尾斜杠
```

**验证网关端点：**

```bash
curl -s https://ai.svtun.cn/v1/chat/completions \
  -H "Authorization: Bearer $LEONIS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-5.6-sol","messages":[{"role":"user","content":"hi"}],"max_tokens":16}'
```

### `401 Unauthorized`

1. `env_key` 指定的环境变量真的设了吗？

   ```bash
   echo $LEONIS_API_KEY
   ```

2. 改完 `~/.zshrc` 后 `source` 了吗？
3. Key 有没有多余的空格 / 引号 / 换行？
4. Windows 上用 `setx` 设置后**必须重开终端**。

### `Unsupported parameter: 'store'` / 响应存储报错

第三方网关通常不支持 OpenAI 的响应存储功能：

```toml
disable_response_storage = true
```

### 流式输出中断

```toml
[model_providers.leonis]
stream_max_retries = 8
stream_idle_timeout_ms = 600000
request_max_retries = 6
```

如果频繁中断，多半是网关侧的 SSE 透传有缓冲，或网络不稳。

### 配置改了不生效

```bash
codex --config model_provider=leonis   # 命令行覆盖，验证配置本身对不对
```

如果命令行覆盖能用而配置文件不行，检查：

- 文件路径对不对：`~/.codex/config.toml`
- TOML 语法有没有错（表头 `[model_providers.leonis]` 的点号别写错）
- 有没有多个 `model_provider` 行互相覆盖

```bash
# 验证 TOML 语法
python -c "import tomllib,sys; tomllib.load(open(sys.argv[1],'rb')); print('OK')" ~/.codex/config.toml
```

### Windows 上路径 / 编码异常

推荐用 WSL2：

```powershell
wsl --install
```

原生 Windows 下如果遇到中文乱码：

```powershell
chcp 65001
```

### 沙箱阻止了合法操作

```bash
codex --sandbox danger-full-access   # 临时放开
```

或改成 `on-failure` 让它先试再申请：

```toml
approval_policy = "on-failure"
```

---

## 和 Claude Code 怎么选

| 场景 | 推荐 |
|---|---|
| 大规模跨文件重构 | Claude Code（上下文管理更强） |
| 精确的单点修改 | Codex CLI（沙箱更细，审批更可控） |
| 需要严格权限隔离 | Codex CLI（三级沙箱） |
| 长会话、大项目 | Claude Code（Prompt 缓存效率高） |
| 需要 MCP 工具生态 | 两者都支持 |

**实际做法是两个都装。** 用 [cc-switch](https://github.com/leonis-ai/cc-switch-guide) 统一管理配置，按任务切换。

---

## 相关项目

| 仓库 | 说明 |
|---|---|
| [claude-code-guide](https://github.com/leonis-ai/claude-code-guide) | Claude Code 中文完全指南 |
| [gemini-api-guide](https://github.com/leonis-ai/gemini-api-guide) | Gemini API 配置手册 |
| [cc-switch-guide](https://github.com/leonis-ai/cc-switch-guide) | 多配置一键切换 |
| [ai-client-configs](https://github.com/leonis-ai/ai-client-configs) | 20+ 客户端配置模板 |
| [ai-api-pricing](https://github.com/leonis-ai/ai-api-pricing) | 成本计算与缓存经济学 |

需要一个同时支持 `/v1/responses` 和 `/v1/chat/completions`、覆盖 GPT / Claude / Gemini / Grok 全系模型的网关，可以看 [**Leonis AI**](https://ai.svtun.cn)。

## 贡献

发现错误或有补充，欢迎 Issue 和 PR。

## License

[MIT](LICENSE)

---

<sub>

**关键词** · Codex CLI · Codex 配置 · Codex 教程 · config.toml · OpenAI Codex · model_providers ·
wire_api · Codex 中转 · Codex 第三方 API · AGENTS.md · MCP · AI 编程助手 · AI 中转 · API 中转

</sub>
