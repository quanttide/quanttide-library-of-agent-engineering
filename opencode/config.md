# OpenCode 配置参考

官方文档：<https://github.com/opencode-ai/opencode>（已归档，项目已迁移至 Crush）

## 配置文件位置

搜索顺序（优先级从高到低）：

1. `./.opencode.json`（项目本地）
2. `$XDG_CONFIG_HOME/opencode/.opencode.json`
3. `$HOME/.opencode.json`

## 配置结构

```jsonc
{
  // 数据存储目录
  "data": { "directory": ".opencode" },

  // AI 供应商
  "providers": {
    "openai":      { "apiKey": "sk-...", "disabled": false },
    "anthropic":   { "apiKey": "sk-ant-...", "disabled": false },
    "copilot":     { "disabled": false },
    "groq":        { "apiKey": "gsk_...", "disabled": false },
    "openrouter":  { "apiKey": "sk-or-...", "disabled": false }
  },

  // 各场景使用的模型
  "agents": {
    "coder": { "model": "claude-3.7-sonnet", "maxTokens": 5000 },
    "task":  { "model": "claude-3.7-sonnet", "maxTokens": 5000 },
    "title": { "model": "claude-3.7-sonnet", "maxTokens": 80 }
  },

  // Shell 配置
  "shell": { "path": "/bin/bash", "args": ["-l"] },

  // MCP 服务器
  "mcpServers": {
    "example": { "type": "stdio", "command": "path/to/mcp-server", "args": [] }
  },

  // LSP 语言服务器
  "lsp": {
    "go": { "disabled": false, "command": "gopls" },
    "typescript": { "disabled": false, "command": "typescript-language-server", "args": ["--stdio"] }
  },

  // 自动压缩会话（接近上下文窗口时自动摘要）
  "autoCompact": true,
  "debug": false,
  "debugLSP": false
}
```

## 环境变量

| 变量 | 用途 |
|------|------|
| `ANTHROPIC_API_KEY` | Claude 模型 |
| `OPENAI_API_KEY` | OpenAI 模型 |
| `GEMINI_API_KEY` | Google Gemini |
| `GITHUB_TOKEN` | GitHub Copilot |
| `GROQ_API_KEY` | Groq |
| `LOCAL_ENDPOINT` | 自托管模型 |

## 自定义命令

存放路径：

- `$XDG_CONFIG_HOME/opencode/commands/`（用户级，前缀 `user:`）
- `<project>/.opencode/commands/`（项目级，前缀 `project:`）

每个 `.md` 文件为一个命令，支持 `$NAME` 命名参数。

## 支持的模型

- OpenAI：GPT-4o, GPT-4.1, o3/o4-mini 等
- Anthropic：Claude 3.5 Haiku, 3.7 Sonnet, 4 Sonnet/Opus
- 其他：Gemini、Groq（Llama 4）、AWS Bedrock、Azure OpenAI、GitHub Copilot

## 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+N` | 新建会话 |
| `Ctrl+K` | 命令面板 |
| `Ctrl+S` | 发送消息 |
| `Ctrl+X` | 取消生成 |
| `Ctrl+A` | 切换会话 |
