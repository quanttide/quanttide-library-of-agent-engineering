# DeepSeek Harness（dsh）

> DeepSeek Harness 是 DeepSeek AI 官方开发的开源 agent harness（智能体框架）。
> 开源仓库：https://github.com/deepseek-ai/deepseek-harness（81.8k stars / 7.2k forks，MIT 许可证）
> 官网：https://deepseek.com/harness
> **当前处于开发者预览阶段，正在快速迭代，未来将出现破坏兼容性的变更。**

## 是什么

DeepSeek Harness（命令行 `dsh`）是 DeepSeek AI 的官方智能体框架。核心设计理念是**一切皆插件**（Everything is a Plugin）：模型适配器、工具注册表、会话日志，乃至 agent loop（智能体循环）本身都是插件，因此每一部分都可以从配置替换，不存在需要打补丁的特权内核。

底层由 [Cordis](https://github.com/cordiverse/cordis) 框架驱动，其设计参见论文 [_A Programming Paradigm for Spatiotemporal Composability_](https://github.com/cordiverse/paper)。

技术栈：TypeScript + pnpm monorepo（仓库含 `packages/`、`apps/`、`native/`、`python/` 等目录）。

## 快速开始

要求：Node.js。

```sh
# 一行启动 Web UI（默认 http://127.0.0.1:3080）
npx @deepseek-ai/dsh web
```

从源码运行：

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

首次使用 Web UI 的流程：

1. **配置模型**：设置 → 模型，输入 DeepSeek API 密钥并保存（无需重启，立即生效）。密钥只写不回显，存储在 `$DSH_HOME/.credentials.yaml`。
2. **选择工作区**：点击"选择工作区"，添加启动 `dsh` 时所在的项目目录。未选择工作区前会话输入框不可用。
3. **运行任务**：新建会话下达指令，agent 可读写工作区文件、执行命令、委派子 agent、维护计划；需要审批的操作会先询问用户。

## 核心概念

### Cordis 与插件树

- 插件向共享上下文贡献服务、类型化事件和可逆的副作用；各项注册是副作用，会在插件卸载时撤销。
- 运行中的 `dsh` 是一棵插件树，由启动时按序叠加的各层组合而成。
- 扩展 dsh 的方式是**把插件挂载到其他插件旁边**，不需要修改内核。

### Profile 与组合包（Bundle）

| 概念 | 说明 |
|------|------|
| **profile** | Harness home 中的具名组装：列出叠放的组合包、存放安装的树外插件、保存用户的 `cordis.patch.yml`。`web` 和 `headless` 作为模板随发行版交付 |
| **组合包** | Cordis 配置项 + 挂载代码的分发格式，插入的内容始终可被上层 patch。在 `package.json` 的 `dsh.bundle` 字段声明 |
| `dsh-base` | 每个 profile 的第一层：模型适配器、工具、持久化、沙箱与审批策略、设置、凭据、遥测 |
| `dsh-web-app` | 增加浏览器应用（Web UI） |
| `dsh-headless` | 增加一次性运行器，完全不带服务器 |

各层应用顺序：profile 列出的组合包 → profile 的 `cordis.patch.yml` → home 级 patch → `--patch` overlay。查看实际启动的配置树：

```sh
dsh --profile web --dump-config
```

### 核心包

| 包 | 职责 | ctx 键 |
|---|---|---|
| `core/session` | 仅追加的 `SessionEvent` 日志和内存存储 | `ctx.sessions` |
| `core/system-prompt` | 提示词片段与工具 schema 的组装 | `ctx.systemPrompt` |
| `core/tools` | 作用域化的工具注册表和带把关的执行流水线 | `ctx.tools` |
| `core/agent` | `Agent` 接口、活跃 agent 注册表和 `agent/*` 事件 | `ctx.agents` |
| `core/agent-loop` | 实现该接口的默认驱动器 | `ctx.agentLoop` |
| `core/scope` | 按 agent 划分作用域的注册原语 | 库，无 ctx 键 |
| `llm/llm` | 消息与流式词汇表，以及适配器 seam | `ctx.llm` |

### 事件与轮次流程

事件就是扩展点，分三个事件域：

- **会话事件**：追加到日志并通过 `session/event` 广播的持久事实（`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*`）
- **Agent 事件**（`agent/*`）：携带活跃 `Agent`，用于观察或拦截进行中的工作
- **能力事件**：向 seam（`fs/*`、`tools/*`、`telemetry/*`）附加策略和适配器

**步骤**（step）= 一次模型请求 + 它调用的工具；**轮次**（turn）= 零或多个步骤。`agent/pre-step`、`agent/request`、`llm/stream`、`tools/*` 是 waterfall 事件，监听器必须调用 `next()` 委托下去。

### 会话日志

会话日志是模型所见上下文的来源，**模型可见即已记录**：抵达模型请求的一切都必须能从日志重建。fork、恢复、transcript、遥测、持久化都派生自事件流。持久化后端：JSONL（默认，支持 Zstandard 压缩）或 SQLite（WAL）。

### 能力 seam

**seam** 是一项可替换能力，包含三种角色：声明接口的 Service Definition、实现它的 Service Provider、使用它的 Consumer（通常是面向模型的工具）。替换一个提供方即可改变整个产品——例如把文件系统与进程提供方指向远程沙箱，Bash、PTY、LSP 便一起搬过去。

### 沙箱与审批

- **沙箱模式**：默认 `read-only`（失败安全）；部署可显式选择 `workspace-write`、`danger-full-access`
- **审批策略**：`ask`（默认，委托给组合的 answerer，无则失败关闭）或 `never`（确定性拒绝，CI/无人值守场景）
- 权限预设：`workspace-write`（默认）与 `danger-full-access`

## 配置

### 目录与环境变量

| 路径/变量 | 用途 |
|---|---|
| `$DSH_HOME`（默认 `~/.dsh`） | Harness home：settings.yaml、.credentials.yaml、profile、本地插件 |
| `$DSH_AGENTS_HOME`（默认 `~/.agents`） | 共享 agent 配置根（技能等） |
| `$DEEPSEEK_API_KEY` | DeepSeek API 密钥（也可在 Web UI 设置中保存） |
| `$DEEPSEEK_BASE_URL` | 覆盖 DeepSeek API 端点 |

### settings.yaml 示例

自定义提供方（公司网关/自建服务器/OpenAI 兼容端点）写在 `$DSH_HOME/settings.yaml`：

```yaml
llm-pi-ai:
  providers:
    my-gateway:
      apiKeyEnv: GATEWAY_API_KEY
      api: openai-completions
      baseURL: https://gateway.example/v1
      models:
        - id: legacy-chat
        - id: vision-preview
          input: [text, image]   # 视觉模型需显式声明 input 模态
```

模型侧相关字段（来自 `dsh-llm-deepseek` 配置）：`thinking`（enabled/disabled）、`reasoningEffort`（off/high/max，默认 high）、`maxTokens`、`defaultContextWindow`（默认 1,000,000）。

### cordis.yml

组合配置使用 Cordis 格式，每个 `config:` 块对应一个可加载包的配置声明（完整字段见官方文档 `docs/config-catalog.md`）。常用配置面：

- `dsh-mcp-client`：stdio / Streamable HTTP 两种 MCP 传输，`serverName` 决定工具命名空间 `mcp__<serverName>__<rawName>`
- `dsh-lsp-stdio`：注册语言服务器（如 typescript、gopls）
- `dsh-bash-sandbox` / `dsh-sandbox-policy`：沙箱执行与默认模式
- `dsh-session-persistence-jsonl` / `dsh-session-persistence-sqlite`：会话持久化后端
- `dsh-compaction-basic`：上下文压缩（默认在上下文窗口 80% 时压缩，保留 16%）

## 内置工具（模型可见）

| 工具 | 用途 |
|------|------|
| `bash` / `pwsh` | 执行 shell 命令（每次调用新进程；`run_in_background` 转后台任务） |
| `read` / `write` / `edit` / `read_image` | 文件读写与编辑 |
| `glob` / `grep` | 基于 ripgrep 的文件搜索 |
| `str_replace_editor` | 字符串替换编辑器（view/create/str_replace/insert） |
| `web_search` / `web_fetch` | 联网搜索与抓取 |
| `subagent` / `subagent_fork` | 委派子 agent（可后台运行） |
| `list_agents` / `send_message` / `interrupt_agent` | 子 agent 控制 |
| `report` | 子级向父级报告结果 |
| `job_list` / `job_output` / `job_kill` | 后台任务统一控制 |
| `todo_write` | 结构化任务列表（整表替换） |
| `skill` | 加载技能（SKILL.md） |
| `lsp` | 语言服务器查询（goToDefinition/findReferences 等） |
| `ask_user_question` | 暂停并向用户提问 |
| `create_goal` / `get_goal` / `update_goal` | 持久化同会话目标（自主轮次续跑） |
| `schedule_create` / `schedule_list` / `schedule_delete` | 定时提醒（可选） |
| `workflow` | JavaScript 工作流脚本大规模编排 subagent |
| `ralph` | 每轮全新子级的 Ralph 循环（可选） |
| `exit_plan_mode` | 规划模式：提交计划供用户评审 |
| `run_code` | code 模式下执行 TypeScript 程序（工具调用 SDK） |
| `session_search` / `session_event_*` | 历史会话全文检索（可选） |
| `terminal_open` / `terminal_send` / … | 持久终端会话（可选） |
| `cordis_*` | 动态定义/运行 Cordis 插件（需显式启用） |

## 与当前项目的关联

| 场景 | 说明 |
|------|------|
| qtcloud-agent 底座评估 | dsh 是 DeepSeek 官方的完整 agent harness（Web UI + headless + SDK），可评估作为智能体云平台的应用运行时 |
| 插件化架构参考 | "一切皆插件" + Cordis 的组合包/patch 机制，与 toolkit 的可扩展性设计可互相借鉴 |
| 模型生态 | 官方模型路由（deepseek-v4-flash / deepseek-v4-pro），也支持 Anthropic/OpenAI 等目录提供方与任意 OpenAI 兼容端点 |
| 插件生态 | GitHub 话题 `dsh-plugin`；社区精选列表 awesome-dsh-plugin；社区有 Electron 桌面端、橙皮书等周边项目 |
| 子 agent / 编排 | 内置 subagent、workflow、goal 等机制，可作为多智能体编排的参照实现 |

## 注意

- **开发者预览阶段**：兼容性破坏性变更随时可能发生，配置与插件 API 不宜过早固化
- 需要 Node.js 环境；Web UI 默认监听 `127.0.0.1:3080`
- 官方中文社区活跃：GitHub Discussions + 企微群 + 微信公众号；`dsh-plugin` 话题可提升插件仓库被发现度
- 官方文档（docs/ 目录）随版本快速演进：`architecture.md`、`config-catalog.md`（配置目录）、`tool-catalog.md`（工具 schema 目录）、`subsystems/`（子系统详解）、`docs/user/guide/`（Web UI 指南）、`docs/develop/`（插件开发）
