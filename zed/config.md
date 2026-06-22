# Zed 配置参考

官方文档：<https://zed.dev/docs/configuring-zed>

## 配置文件位置

| 平台 | 路径 |
|------|------|
| Linux | `~/.config/zed/settings.json` |
| macOS | `~/Library/Application Support/Zed/settings.json` |

## 关键配置分类

### AI / 模型（language_models）

配置各模型供应商的 API 接入：

```jsonc
{
  "language_models": {
    "openai_compatible": {
      "Z.AI": {
        "api_url": "https://open.bigmodel.cn/api/paas/v4",
        "available_models": [
          { "name": "glm-5.2", "display_name": "GLM-5.2", "max_tokens": 1000000 }
        ]
      }
    },
    "openai": { "api_url": "https://api.openai.com/v1" },
    "anthropic": { "api_url": "https://api.anthropic.com" },
    "ollama": { "api_url": "http://localhost:11434" }
  }
}
```

- provider ID 如 `Z.AI` 对应环境变量 `Z_AI_API_KEY`（转大写蛇形 + `_API_KEY`）
- `api_url` 为 base URL，Zed 自动追加 `/chat/completions`
- 每个模型需声明 `max_tokens`（上下文）、`max_output_tokens`、`max_completion_tokens`、`capabilities`

### Agent 配置（agent）

```jsonc
{
  "agent": {
    "default_model": {
      "provider": "deepseek",
      "model": "deepseek-v4-flash",
      "effort": "high",
      "enable_thinking": true
    },
    "dock": "right",
    "tool_permissions": {
      "tools": {
        "write_file": { "always_allow": [{ "pattern": "^data/" }] },
        "terminal": { "default": "allow" },
        "fetch": { "default": "allow" }
      }
    }
  }
}
```

### 编辑器（通用）

| 设置 | 说明 | 默认 |
|------|------|------|
| `buffer_font_size` | 编辑器字号 | 15 |
| `ui_font_size` | UI 字号 | 16 |
| `theme` | 主题（支持 mode/dark/light 对象） | One Dark |
| `tab_size` | 缩进空格数 | 4 |
| `soft_wrap` | 自动换行 | none |
| `format_on_save` | 保存时格式化 | on |
| `autosave` | 自动保存 | off |
| `vim_mode` | Vim 模式 | false |

### 面板布局

```jsonc
{
  "project_panel": { "dock": "left", "default_width": 240 },
  "outline_panel": { "dock": "left" },
  "collaboration_panel": { "dock": "left" },
  "git_panel": { "dock": "left" },
  "terminal": { "dock": "bottom" }
}
```

### 语言特定覆盖（languages）

```jsonc
{
  "languages": {
    "C": { "format_on_save": "off", "preferred_line_length": 64, "soft_wrap": "bounded" },
    "JSON": { "tab_size": 4 }
  }
}
```

## 完整参考

全部可用设置项参见官方文档：<https://zed.dev/docs/reference/all-settings>
