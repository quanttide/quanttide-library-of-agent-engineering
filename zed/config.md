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

## Agent Skills

文档：<https://zed.dev/docs/ai/skills>

Skill 保存路径：

| 范围 | 路径 |
|------|------|
| 全局 | `~/.agents/skills/<skill-name>/SKILL.md` |
| 项目级 | `<worktree>/.agents/skills/<skill-name>/SKILL.md` |

项目级同名 Skill 覆盖全局。

### 核心 Skill：ponytail

Ponytail 让 Agent 像 lazy senior dev 一样思考——YAGNI、优先 stdlib、原生功能、一行代码。

安装方式：从仓库复制到 `~/.agents/skills/`：

```bash
cp -r <ponytail-repo>/skills/ponytail* ~/.agents/skills/
```

包含 6 个子 Skill：

| Skill | 用途 | 触发 |
|-------|------|------|
| `ponytail` | 懒人模式下写最简代码（默认 full 级别） | 自动 / `/ponytail` |
| `ponytail-review` | 审查 diff 过度工程 | `/ponytail-review` |
| `ponytail-audit` | 全仓库过度工程审计 | `/ponytail-audit` |
| `ponytail-debt` | 提取 `ponytail:` 注释成技术债台账 | `/ponytail-debt` |
| `ponytail-gain` | 展示基准测试量化收益 | `/ponytail-gain` |
| `ponytail-help` | 命令速查卡 | `/ponytail-help` |

强度级别：`lite`（命名替代方案）/ `full`（默认，强制执行阶梯）/ `ultra`（YAGNI 极端模式）。

仓库：<https://github.com/DietrichGebert/ponytail>

## 完整参考

全部可用设置项参见官方文档：<https://zed.dev/docs/reference/all-settings>
