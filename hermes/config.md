# Hermes 配置参考

> 基于 `~/.hermes/hermes-agent` 源码与仓库内 `cli-config.yaml.example` 整理，对应 Hermes Agent v0.19.0（upstream `cff97285`），本机配置版本 33。
> 最后更新：2026-09-22

## 配置文件位置

主配置是 `$HERMES_HOME/config.yaml`，API 密钥与少量环境变量在 `$HERMES_HOME/.env`。`HERMES_HOME` 未设置时取平台默认路径：Linux/macOS 为 `~/.hermes`，Windows 为 `%LOCALAPPDATA%\hermes`。解析顺序是 context-local 覆盖、`HERMES_HOME` 环境变量、平台默认路径，统一由 `hermes_constants.get_hermes_home()` 提供，其余模块都从这一处导入。

以下命令可查当前状态：

- `hermes config path` 打印配置路径；
- `hermes config env-path` 打印 `.env` 路径；
- `hermes config check` 比对配置版本并列出已配与缺失的环境变量；
- `hermes config show` 按段列出解析后的生效值。

多套环境用 profile 隔离，由 `hermes profile` 管理（`list`、`use`、`create`、`export`、`install` 等）。命名 profile 存在 `~/.hermes/profiles/<名字>/`，各自是一份独立的 `HERMES_HOME`，因此配置、密钥、状态库、网关互不共享。

## 配置结构

`config.yaml` 只记录与默认值不同的项，默认值全表在 `hermes_cli/config.py` 的 `DEFAULT_CONFIG`（79 个顶层键），带注释的样板是仓库根目录的 `cli-config.yaml.example`。常用段：

- `model` 模型与 provider：`default`、`provider`、`base_url`、`api_key`、`context_length`、`max_tokens`、`default_headers`；
- `providers` 按 provider 覆写请求超时与按模型例外；
- `fallback_model` 主 provider 不可用时的兜底模型（单个或链条）；
- `agent` 行为：`max_turns`、`reasoning_effort`、`verbose`、`personalities`、`tool_use_enforcement`；
- `terminal` 终端工具 backend（`local`、`ssh`、`docker`、`singularity`、`modal`、`daytona`）及其资源与超时；
- `compression` 上下文自动压缩：`enabled`、`threshold`、`target_ratio`、`protect_last_n`；
- `memory` 持久记忆与用户画像：开关、字数上限、nudge 与 flush 时机；
- `session_reset` 消息平台会话重置：`none`、`idle`、`daily`、`both`；
- `display` 显示：`personality`、`skin`、`streaming`、`show_reasoning`、`tool_progress`；
- `stt` 语音转写：`local`、`groq`、`openai`、`mistral`；
- `skills` 技能目录与外部技能路径；
- `delegation` 子智能体派发的模型、provider 与迭代上限；
- `platform_toolsets` 各平台可用的工具箱；
- `approvals` 审批策略：`mode`、`timeout`、`destructive_slash_confirm`；
- `command_allowlist` 免确认的命令模式；
- `hooks` 注册 shell 脚本为 hook 回调；
- `mcp` 外部 MCP server；
- `privacy` PII 脱敏；
- `model_aliases` `/model` 命令的短名映射；
- `telemetry` 遥测，默认关闭。

### 模型段

```yaml
model:
  default: deepseek-flash   # 可用 --model 单次覆盖
  provider: deepseek        # 内置 provider id，或 auto 表示按凭证自动探测
  # base_url: 仅自定义 OpenAI 兼容端点需要，内置 provider 有默认值
  # api_key:  默认从 .env 读，不写进 config.yaml
  # context_length: 总上下文（输入加输出），留空自动探测
  # max_tokens: 单次输出上限，留空用模型原生上限
```

两个长度字段容易混。`context_length` 是总上下文窗口，决定何时压缩历史、请求是否超限；`max_tokens` 只是单次回复的输出上限，与对话历史长度无关。两者都建议留空，由 Hermes 从 provider 自动探测。

### 环境变量与优先级

`.env` 中的环境变量优先级高于 `config.yaml` 的同名设置（见样板文件顶部注释）。各 provider 的密钥变量与内置 base URL：

| provider | 名称 | 内置 base URL | 密钥环境变量 |
|:--|:--|:--|:--|
| `deepseek` | DeepSeek | `https://api.deepseek.com/v1` | `DEEPSEEK_API_KEY` |
| `zai` | Z.AI / GLM | `https://api.z.ai/api/paas/v4` | `GLM_API_KEY`、`ZAI_API_KEY`、`Z_AI_API_KEY` |
| `xiaomi` | Xiaomi MiMo | `https://api.xiaomimimo.com/v1` | `XIAOMI_API_KEY` |
| `kimi-coding` | Kimi / Moonshot | `https://api.moonshot.ai/v1` | `KIMI_API_KEY` |
| `anthropic` | Anthropic | `https://api.anthropic.com` | `ANTHROPIC_API_KEY` |
| `openai-api` | OpenAI API | `https://api.openai.com/v1` | `OPENAI_API_KEY` |
| `gemini` | Google AI Studio | `https://generativelanguage.googleapis.com/v1beta` | `GOOGLE_API_KEY`、`GEMINI_API_KEY` |
| `copilot` | GitHub Copilot | GitHub Models 端点 | `GITHUB_TOKEN`、`GH_TOKEN` |
| `ollama-cloud` | Ollama Cloud | Ollama Cloud 端点 | `OLLAMA_API_KEY` |

每个 provider 都有对应的 `*_BASE_URL` 覆盖变量（如 `DEEPSEEK_BASE_URL`、`XIAOMI_BASE_URL`）。完整清单见 `hermes_cli/config.py` 的 `OPTIONAL_ENV_VARS`，`hermes config check` 会标出哪些已配。

模型清单不写进配置：选择器运行时拉 provider 的 `/v1/models`，结果缓存在 `provider_models_cache.json` 一小时；live 拉取失败时回退到 `hermes_cli/models.py` 的静态兜底表，那部分要等 Hermes 发版。provider id 支持别名归一化，如 `glm`、`z.ai` 归一为 `zai`，`mimo` 归一为 `xiaomi`，`kimi`、`moonshot` 归一为 `kimi-coding`。

### 规则文件

会话会加载 `AGENTS.md`、`SOUL.md`、`.cursorrules` 与持久记忆；`--ignore-rules`（或 `HERMES_IGNORE_RULES=1`）可整轮跳过。

## 常用命令

- `hermes config get <key>` 打印解析后的值，如 `hermes config get model`；
- `hermes config set <key> <value>` 写入配置，如 `hermes config set model.default mimo-v2.6-flash`；
- `hermes config unset <key>` 删除键；
- `hermes config edit` 用编辑器打开配置；
- `hermes config check` 检查配置版本与环境变量；
- `hermes config migrate` 按新版本补齐配置项；
- `hermes model [--refresh]` 交互选择模型，`--refresh` 清缓存并重拉各 provider 的 `/v1/models`；
- `hermes -m <model> --provider <provider>` 单次运行覆盖模型；
- `hermes doctor` 体检。

## 版本与迁移

配置带 `_config_version` 字段，当前为 33。Hermes 升级后 `hermes config check` 会比对默认值并报出版本差异，`hermes config migrate` 负责补齐新键。因为只写差异项，迁移后文件依然很短——本机配置只有 10 个顶层键。

## 关键文件路径

- `~/.hermes/config.yaml` 主配置；
- `~/.hermes/.env` API 密钥；
- `~/.hermes/SOUL.md` 默认系统提示词；
- `~/.hermes/skills/` 技能；
- `~/.hermes/auth.json` provider 凭证池与端点探测结果；
- `~/.hermes/provider_models_cache.json` 模型清单缓存，一小时 TTL；
- `~/.hermes/models_dev_cache.json` models.dev 注册表缓存；
- `~/.hermes/state.db` 会话与状态主库（SQLite）；
- `~/.hermes/sessions/` 会话导出；
- `~/.hermes/logs/`、`~/.hermes/cron/`、`~/.hermes/memories/` 日志、定时任务、记忆。

## 相关文档

配置维护方法（provider、密钥、切换模型）见本仓 profile 的 [hermes/model-setup.md](../../profile/default/agents/hermes/model-setup.md)。[session.md](./session.md) 描述会话存储与上下文压缩机制，依据 v0.9.x 源码（最后更新 2026-07-11），与 v0.19.0 可能存在出入。
