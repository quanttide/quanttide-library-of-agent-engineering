# Hermes 配置参考

> Hermes 是一个终端 AI 编码 agent，由 generativebootstrap 开发。
> 开源仓库已归档。以下内容基于公开文档总结。

## 配置文件位置

- `$HOME/.hermes/config.json` 或 `$HOME/.config/hermes/config.json`

## 配置结构

```jsonc
{
  // AI 供应商
  "provider": "openai",
  "model": "gpt-4o",
  "api_key": "sk-...",     // 也支持环境变量 HERMES_API_KEY
  "max_tokens": 4096,

  // 工具权限
  "tools": {
    "bash": { "enabled": true },
    "file_read": { "enabled": true },
    "file_write": { "enabled": true },
    "glob": { "enabled": true },
    "grep": { "enabled": true },
    "fetch": { "enabled": true }
  },

  // 会话管理
  "session": {
    "auto_save": true,
    "max_history": 100
  },

  // shell
  "shell": "/bin/bash"
}
```

## 环境变量

| 变量 | 用途 |
|------|------|
| `HERMES_API_KEY` | API 密钥 |
| `OPENAI_API_KEY` | OpenAI 兼容接口密钥 |

## 特点

- 终端 TUI 交互，支持多轮对话
- 内置 bash、文件读写、grep 等工具
- 支持自定义系统提示词
