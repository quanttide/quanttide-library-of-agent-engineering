# Herdr

> Herdr 是终端里的智能体运行时：它把每个编程智能体关在一个真 PTY 里，客户端退出、断网、换机器都不影响这些进程继续跑。
> 开源仓库：<https://github.com/herdrdev/herdr>（Rust 为主，Apache-2.0，45,960 stars / 3,664 forks，2026-03-27 建立）
> 官网与文档：<https://herdr.dev>、<https://herdr.dev/docs/>
> 本文依据 v0.9.5（2026-10-14 发布，socket 协议 22）。

## 是什么

Herdr 自身不含模型，也不含智能体循环。它的默认配置一共 14 段（段名见下文「配置」一节），没有一处填模型名、密钥或 base_url 的字段——`herdr --default-config` 里搜 `model`、`llm`、`api_key` 零命中。用哪个模型、怎么思考，全部由塞进 pane 的那个 CLI 决定。

它做的是三件事：把真终端进程托管在后台 server 里；认出 pane 里的东西是不是智能体、现在卡在哪一步；把这两件事暴露成 CLI 与 socket API，供人和脚本使用。

它和终端复用器的分界在这里：tmux 保住终端，但不知道哪个终端里是智能体、谁在等审批。Herdr 多出来的正是这份状态与等待语义。

## 架构

运行形态是后台 server 加一到多个客户端。server 持有 pane 与进程状态，客户端只是贴上去的 TUI；`herdr server` 可以单独以无界面方式跑。分离客户端用 `ctrl+b q`，server 与智能体继续运行；要真正收摊得停 server。

层级是 workspace、tab、pane 三层。workspace 对应一个仓库、任务或一次调查；tab 是 workspace 内的布局，用来分开智能体、日志、服务器、复核等视角；pane 是货真价实的终端，可左右或向下拆分，可改名、写入、读取、关闭，客户端分离后仍然保留。

session 是 server 的持久命名空间，默认那个就叫 default。`herdr --session <name>` 起另一套完全独立的 pane、socket 与持久运行状态。多机则是 `herdr machine add <ssh-host>`：每台机器跑自己的 server 和 session，断掉一台不影响其他机器，侧边栏把它们并排列出来。连接请求 SSH 压缩以减轻屏幕刷新流量，共享连接默认 `ControlPersist 600`。

本机 v0.9.5 上实测的生命周期：在 pty 里起 `herdr`，`herdr status server` 报 running；对客户端进程 `kill -9` 之后 status 仍为 running，`herdr api snapshot` 照常返回 `w1:p1` 的 pane 与布局。也就是说活着的是 server，客户端可随时丢弃。

## 智能体状态

状态只有五个：blocked（等输入、审批或决策）、working（在跑）、done（干完了但你没看）、idle（干完或等输入且已看过）、unknown（判不准）。状态会向上汇总，一个 pane 卡住会让 tab 与 workspace 都显示卡住，这样多开时不必逐个 pane 翻。

识别走三路：pane 的前台进程、屏幕清单、集成上报。屏幕清单读的是终端当前底部画面而不是你滚动到的位置，规则来自三层——二进制内置的、herdr.dev 后台下发的、以及 `~/.config/herdr/agent-detection/<agent>.toml` 本地覆盖，本地覆盖优先。清单是规则匹配，不是推理：blocked 的判定刻意保守，只认已知的审批、提问、权限界面；没匹配上就退成 idle 并标注 `default_known_agent_idle_fallback`。判错时用 `herdr agent explain <target>` 看命中的规则与证据。

## 集成与会话恢复

Herdr 官网标注可识别 22 个 agent CLI，实际清单含 Claude Code、Codex、GitHub Copilot CLI、Cursor Agent CLI、OpenCode、Pi、Hermes Agent、Qwen Code、Grok CLI 等。不在清单里的智能体照常跑，只是退化成普通终端，没有状态、通知与等待。

集成的作用是上报会话标识。装法是 `herdr integration install <agent>`，写的是各 agent 自己的配置（例如 Hermes 写 `~/.hermes/plugins/herdr-agent-state/` 并在 `config.yaml` 的 `plugins.enabled` 里加一项）。server 重启后，Herdr 用手上的 session id 拼出 `hermes --resume <id>` 这类命令把 pane 接回原会话——恢复靠重放 CLI 自己的恢复入口，而不是序列化进程内存。少数智能体反过来自己上报状态，无需安装任何东西。

## 控制面

控制面分三层：给 pane 里的智能体读的技能文件（`herdr --skill`）、给脚本用的 CLI 包装、给自定义工具用的原始 socket API（`herdr api schema` 打印协议结构）。方法族覆盖 server、notification、client、session、workspace、worktree、tab、pane、layout、popup、agent、events、integration、plugin。

其中 `agent.wait` 是关键设计：等待由 server 拥有、事件驱动，并且会锁定解析到的 pane 占用者，换了进程不算满足等待。`agent.prompt` 可以带 `until` 与 `timeout_ms` 一次提交并开始等待，避免提交与等待之间出现竞态；如果目标已经 blocked，直接返回 `agent_blocked`，不盲敲键。这正是它区别于「往终端里打字符然后祈祷」的地方。

插件是可执行的工作流包：`plugin.toml` 里声明动作、面板、按键绑定与链接处理器，动作就是一条命令，可以调回 `herdr` CLI。官网标注插件目录现有 1,789 个社区插件。

## 配置

配置文件在 `~/.config/herdr/config.toml`，`herdr --default-config` 打印全部默认值，`herdr server reload-config` 热载。默认段为 `theme`、`terminal`、`update`、`keys`、`server`、`ui`、`ui.toast`、`ui.toast.herdr`、`ui.toast.clipboard`、`ui.sound`、`session`、`remote`、`experimental`、`advanced`。日志在 `~/.config/herdr/herdr.log`（另有 client 与 server 两份）。安装走 `curl -fsSL https://herdr.dev/install.sh | sh`，落到 `~/.local/bin/herdr`，Homebrew、mise、Nix 均有包。

## 与 qtcloud-agent 的关联

作为 qtcloud-agent 的主要参考框架，以下五条是它的可迁移部分。

运行面与控制面分离：server 持有 PTY 与进程，客户端可随时丢弃重建。云端会话的持久化因此不必绑在应用层进程上，也不必绑在某一块屏幕上。

状态来源分层：前台进程判身份、屏幕规则判状态、集成上报做精确化，三层各有退化路径。这套分层在只能看到别人家 CLI 输出的场景里是可复用的，比要求所有智能体配合改造更现实。

等待语义归服务器：等待是事件驱动的服务端能力，会锁定被等的对象，并且可以和投递指令合成一次请求。这一条直接决定编排层是否需要轮询。

恢复靠外部会话标识：集成只上报 CLI 自己的 session id，重启后拼恢复命令重入。比快照进程内存稳，也顺带兼容了各家 CLI 自己的会话存储。

多机即多 server：一台机器一套 server 与 session，多套并排汇总，断线互不影响。这是「云平台不该由单一控制面持有全部运行状态」的现成做法。

不该照抄的部分：屏幕规则匹配会随上游 CLI 改界面而失效，它靠远端热更清单兜住，云平台若采用同类方案必须自带这份更新通道；它不含模型与智能体循环，所以不能当编排内核用；它没有 diff 与复核界面，代码审查这一层仍需另配。

## 参考资料

- 概念与架构：<https://herdr.dev/docs/concepts/>
- 智能体状态与集成：<https://herdr.dev/docs/agents/>、<https://herdr.dev/docs/integrations/>
- socket API：<https://herdr.dev/docs/socket-api/>
- 远程与多机：<https://herdr.dev/docs/connecting-machines/>、<https://herdr.dev/docs/persistence-and-remote-access/>
- 插件：<https://herdr.dev/docs/plugins/>、插件目录 <https://herdr.dev/plugins/>
- 供智能体直接读取的说明文件：<https://herdr.dev/agent-guide.md>、<https://herdr.dev/llms.txt>
