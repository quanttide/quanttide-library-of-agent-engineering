# Hermes Session 存储与上下文机制

> 基于 `~/.hermes/hermes-agent/` 源码 v0.9.x 分析。
> 最后更新：2026-07-11

## 目录

- [存储架构](#存储架构)
- [会话生命周期](#会话生命周期)
- [会话恢复（Resume）流程](#会话恢复resume流程)
- [上下文压缩机制](#上下文压缩机制)
- [记忆系统](#记忆系统)
- [关键文件路径](#关键文件路径)
- [数据脱敏注意事项](#数据脱敏注意事项)

---

## 存储架构

双层存储：SQLite DB 为主存储，JSON 文件为辅。

### SQLite DB

**路径**: `~/.hermes/state.db`
**模式**: WAL（Write-Ahead Logging），支持一写多读并发

#### 表: `sessions`

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | TEXT PK | 格式 `YYYYMMDD_HHMMSS_6位hex` |
| `source` | TEXT | 来源标记：`cli`, `weixin`, `telegram`, `discord` 等 |
| `user_id` | TEXT | 平台用户 ID |
| `model` | TEXT | 模型名 |
| `model_config` | TEXT | JSON 序列化的模型配置 |
| `system_prompt` | TEXT | 系统提示词（含注入的记忆） |
| `parent_session_id` | TEXT | 父会话 ID（上下文压缩分裂时产生） |
| `started_at` | REAL | Unix 时间戳 |
| `ended_at` | REAL | 结束时间戳 |
| `end_reason` | TEXT | 结束原因 |
| `message_count` | INTEGER | 消息数 |
| `tool_call_count` | INTEGER | 工具调用数 |
| `input_tokens` | INTEGER | 输入 tokens |
| `output_tokens` | INTEGER | 输出 tokens |
| `cache_read_tokens` | INTEGER | 缓存读取 tokens |
| `cache_write_tokens` | INTEGER | 缓存写入 tokens |
| `reasoning_tokens` | INTEGER | 推理 tokens |
| `title` | TEXT | 会话标题 |

#### 表: `messages`

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | INTEGER PK AUTO | 自增 ID |
| `session_id` | TEXT FK | 归属会话 |
| `role` | TEXT | `system`, `user`, `assistant`, `tool` |
| `content` | TEXT | 消息内容 |
| `tool_call_id` | TEXT | 工具调用 ID |
| `tool_calls` | TEXT | JSON 序列化的工具调用列表 |
| `tool_name` | TEXT | 工具名 |
| `timestamp` | REAL | Unix 时间戳 |
| `token_count` | INTEGER | Token 数 |
| `finish_reason` | TEXT | 结束原因 |
| `reasoning` | TEXT | 推理内容（思考链） |
| `reasoning_details` | TEXT | JSON 序列化的推理细节 |
| `codex_reasoning_items` | TEXT | Codex 格式推理项 |

#### 索引

- `idx_sessions_source` — 按来源查询
- `idx_sessions_parent` — 按父会话查询
- `idx_sessions_started` — 按时间排序
- `idx_messages_session` — 按会话查询消息

#### FTS5 全文搜索

虚拟表 `messages_fts`，通过触发器自动同步 `messages.content` 的增删改。

### JSON 文件

**路径**: `~/.hermes/sessions/session_{timestamp}_{uuid}.json`

每会话一个 JSON 文件，结构：

```json
{
  "session_id": "20260711_145813_946390",
  "model": "deepseek-chat",
  "base_url": "https://api.deepseek.com/v1/",
  "platform": "weixin",
  "session_start": "2026-07-11T14:58:13.523034",
  "last_updated": "2026-07-11T14:58:20.118827",
  "system_prompt": "...",
  "tools": [...],
  "message_count": 3,
  "messages": [...]
}
```

**索引文件**: `~/.hermes/sessions/sessions.json`

```json
{
  "agent:main:weixin:dm:{user_id}": {
    "session_key": "...",
    "session_id": "20260711_145813_946390",
    "created_at": "...",
    "updated_at": "...",
    "display_name": "...",
    "platform": "weixin",
    "chat_type": "dm",
    "input_tokens": 0,
    "output_tokens": 0,
    "last_prompt_tokens": 0,
    "memory_flushed": false,
    "suspended": false
  }
}
```

---

## 会话生命周期

### 创建

```python
# cli.py
timestamp_str = session_start.strftime("%Y%m%d_%H%M%S")
short_uuid = uuid.uuid4().hex[:6]
session_id = f"{timestamp_str}_{short_uuid}"
```

### 消息写入

每轮对话通过 `SessionDB.append_message()` 写入 SQLite。

### 结束

`SessionDB.end_session(session_id, end_reason)` — 更新 `ended_at` 和 `end_reason`。

### 分裂

当上下文压缩触发时，系统创建子会话并通过 `parent_session_id` 链接到原会话。参见 [上下文压缩机制](#上下文压缩机制)。

---

## 会话恢复（Resume）流程

```python
# cli.py:_preload_resumed_session()
1. SessionDB.get_session(session_id)     # 查询元数据
2. SessionDB.get_messages_as_conversation()  # 拉取全部消息
3. 过滤掉 role=session_meta 的行
4. 赋值到 conversation_history
5. 传递给 AIAgent 作为初始上下文
```

**`get_messages_as_conversation()` 返回格式**:

```python
[
  {"role": "system", "content": "你是一个..."},
  {"role": "user", "content": "帮我..."},
  {"role": "assistant", "content": "好的", "reasoning": "...", "tool_calls": [...]},
  {"role": "tool", "content": "结果", "tool_call_id": "call_xxx"},
]
```

> 注意：消息从 SQLite 原样还原，**不经过上下文压缩器**。压缩只发生在运行时，持久化存储的是未压缩的原始消息。

---

## 上下文压缩机制

这是导致「越用越笨」的最可能原因。

### 配置

```yaml
# config.yaml
context:
  engine: compressor
compression:
  enabled: true
  threshold: 0.5          # 50% 上下文阈值
  target_ratio: 0.2       # 压缩目标比例
  protect_last_n: 20      # 保护尾部 N 条消息
```

### 算法（`agent/context_compressor.py:ContextCompressor.compress()`）

```
输入: messages 列表

1. 裁剪旧工具结果
   - 遍历消息（尾部保护区域除外）
   - tool role 且内容 >200 字符 → 替换为 "[tool result was pruned]"
   - 目的：无 LLM 调用的低成本清理

2. 确定压缩边界
   - 保护头部: system prompt + protect_first_n 条消息
   - 保护尾部: 按 token 预算 ~20K tokens 计算尾部边界
   - 头部和尾部之间的中间轮次进入摘要

3. LLM 生成摘要
   - 序列化中间轮次 → 调用 LLM 生成结构化摘要
   - 摘要包含：目标、已完成任务、关键发现、未完成任务、当前状态

4. 组装压缩后消息列表
   - 头部（保留原样）
   - 摘要（以 system 角色插入，标注为压缩摘要）
   - 尾部（保留原样）

5. 清理孤儿 tool_call/tool_result 对
```

### 逐层衰减问题

```
第1次压缩: [头部] + 摘要A(轮次5-20) + [尾部轮次21-30]
第2次压缩: [头部] + 摘要B(摘要A + 轮次21-35) + [尾部轮次36-45]
第3次压缩: [头部] + 摘要C(摘要B + 轮次36-50) + [尾部轮次51-60]
```

- 摘要A 包含轮次 5-20 的意图细节
- 摘要B 是对摘要A 的再摘要 → 细节被进一步压缩
- 摘要C 是摘要的摘要的摘要 → 原始意图几乎不可恢复

**这是核心退化路径。** 用户的意图约束、中途修正、未完成的需求，都在逐层摘要中丢失。

### 会话分裂

压缩也可能触发 session 分裂：

```python
# 新 session 通过 parent_session_id 指向原 session
new_session.parent_session_id = original_session.id
```

但当 `parent_session_id` 链过长时，系统可能无法有效追溯早期上下文。

---

## 记忆系统

### 配置

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200      # 记忆上限 2200 字符
  user_char_limit: 1375        # 用户画像上限 1375 字符
  flush_min_turns: 6           # 至少 6 轮才刷新记忆
```

### 注入方式

记忆内容被**注入到每轮对话的 system prompt 末尾**：

```
╔══ system prompt ══╗
║  你是一个...      ║
║                    ║
║  ─ MEMORY ─       ║
║  项目A 的配置...   ║
║  用户偏好...       ║
║  v0.6.9 发版...   ║
║  ...              ║
╚════════════════════╝
```

### 已知问题

- 记忆占用过高（实测 87%，1,930/2,200 字符）会稀释对当前意图的注意力
- 记忆内容包含大量项目细节但未区分优先级
- 记忆只通过 `flush_min_turns` 控制刷新频率，缺乏过期/相关性机制

---

## 关键文件路径

| 路径 | 用途 |
|---|---|
| `~/.hermes/config.yaml` | 运行时配置 |
| `~/.hermes/state.db` | SQLite 会话存储 |
| `~/.hermes/sessions/session_*.json` | 会话 JSON 文件 |
| `~/.hermes/sessions/sessions.json` | 会话索引 |
| `~/.hermes/memories/` | 记忆存储 |
| `~/.hermes/checkpoints/` | 检查点 |
| `~/.hermes/hermes-agent/` | Python 源码根目录 |
| `~/.hermes/hermes-agent/hermes_state.py` | `SessionDB` 实现 |
| `~/.hermes/hermes-agent/agent/context_compressor.py` | 上下文压缩器 |
| `~/.hermes/hermes-agent/agent/context_engine.py` | 上下文引擎基类 |
| `~/.hermes/hermes-agent/cli.py` | CLI 入口（会话生命周期管理） |
| `~/.hermes/hermes-agent/trajectory_compressor.py` | 轨迹压缩器（离线压缩工具） |
| `~/.hermes/logs/` | 日志 |

---

## 数据脱敏注意事项

分析 Hermes 会话数据时需要脱敏的字段：

| 字段 | 脱敏方式 |
|---|---|
| `user_id` | 替换为占位符 |
| `content` 中的姓名/项目名 | 替换为 `[NAME]`/`[PROJECT]` |
| 代码片段中的 API key、token | 替换为 `[REDACTED]` |
| 内部链接/URL | 替换为 `[INTERNAL_URL]` |
| `system_prompt` 中的记忆内容 | 检查是否含敏感信息 |
| `tool_calls` 参数中的敏感数据 | 逐字段检查 |
