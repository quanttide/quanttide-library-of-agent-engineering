# Vicoa

> Vicoa 是开源的 AI 编排器（AI orchestrator）：在任意设备上统一调度一支编程智能体团队，在同一仓库里并行工作。
> 开源仓库：<https://github.com/vicoa-ai/vicoa>（AGPLv3 许可证）
> 官网：<https://vicoa.ai>　文档：<https://vicoa.ai/docs>

## 是什么

Vicoa 不替代单个编程智能体，而是把若干个智能体组成一支团队：同一工作区并行开工，每个智能体独占一个 git worktree 与分支，因此同一个仓库里互不干扰——这是它「并行而不打架」的根基。可接入的智能体有 Claude Code、Codex、OpenCode、Gemini、Cursor、GitHub Copilot、Kimi、Hermes 等。

它解决的是「一支 AI 团队怎么组织、怎么并行、怎么盯」：接下任务、分派给智能体、把过程与结论集中呈现给人。团队之外还有配套：内置 git 差异审查、文件浏览器、终端、任务看板、自动化调度（cron）与技能管理。

## 四类能力

| 能力 | 做法 |
| :-- | :-- |
| 多智能体并行编排 | 同一仓库多智能体并行，git worktree 隔离互不干扰 |
| 统一指挥台 | 所有会话、任务与自动化集中可见可操作 |
| 多端协同与通知 | 桌面、CLI 与手机同步同一会话；决策与完成时推送 |
| BYO-key | 用户自带模型订阅与 API key；平台不锁死模型供应商 |

## 架构

仓库约 28 万行，核心在 `backend/`（Python）：云端 API 与实时服务器、CLI 与本地 daemon、各智能体的无头运行适配器。`apps/web` 是 Next.js 仪表盘，`apps/desktop` 是 Electron 桌面壳（内置 daemon），`apps/mobile` 是 Flutter 移动端。

运行链路：客户端 → 云端 FastAPI 与 WebSocket → 本地 `vicoa daemon` → 拉起各智能体 CLI。智能体接入平台有两种方式：

| 接入方式 | 适用智能体 | 实现 |
| :-- | :-- | :-- |
| 原生 SDK | Claude Code、Codex | 直连各自的 Agent SDK 与 app-server |
| ACP 协议 | OpenCode、Cursor、Gemini、Copilot、Kimi | 统一封装 Agent Client Protocol，会话、权限、中断等通用逻辑复用 |

daemon 以硬件标识与 API key 确定性派生机器 ID，注册后经 REST 轮询与 WebSocket 双通道接收任务；实时协议里连接先发 hello 帧声明作用域，按机器、会话、用户、终端四类通道隔离数据。智能体侧通过 MCP 工具（`log_step`、`ask_question`、`end_session`）与平台交互，自动化调度依赖 cron 任务。

## 与本库的关联

| 场景 | 说明 |
| :-- | :-- |
| 并行隔离 | worktree + 分支的做法可借鉴：多智能体分工同一仓库的前提是隔离 |
| 长任务与人在环 | 决策与完成推送到手机，人不在电脑前也能批准——异步审批的形态参考 |
| 模型无关 | BYO-key 与 ACP 这类统一适配层，是「不锁死供应商」的做法参考 |

## 参考资料

- [README（中文）](https://github.com/vicoa-ai/vicoa/blob/main/README.zh.md)、[README（英文）](https://github.com/vicoa-ai/vicoa/blob/main/README.md)
- [Vicoa 文档](https://vicoa.ai/docs)、[自托管指南](https://github.com/vicoa-ai/vicoa/blob/main/SELF_HOSTING.md)
- 基本资料与源码架构的逐项分析另见 `quanttide-code` 域的 `data/library/vicoa/index.md`（跨领域关联）
