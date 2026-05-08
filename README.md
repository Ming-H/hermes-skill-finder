# Hermes Skill Finder

从 Hermes Agent 技能市场（680+ 技能）中实时检索，根据用户自然语言需求推荐最合适的技能，并给出对应 Agent 平台的安装命令。

## 解决什么问题

[Hermes Agent 技能市场](https://hermes-agent.nousresearch.com/docs/skills) 有 680+ 个技能，涵盖 20+ 分类，用户很难快速找到自己需要的那个。这个技能就是帮你用自然语言描述需求，自动匹配并推荐最合适的 1-3 个技能。

## 支持的 Agent 平台

| 平台 | 安装命令 | 兼容性 |
|------|---------|--------|
| Claude Code | `npx skills add <id> -a claude-code -g` | 纯知识型技能完美兼容 |
| Hermes | `hermes skills install <id>` | 全部兼容 |
| 通用 | `npx skills add <id> --all` | 两个平台都装 |

## 安装

将此目录复制到你的 Agent 技能目录：

```bash
# Claude Code（项目级）
cp -r hermes-skill-finder/ <project>/.claude/skills/

# Claude Code（全局）
cp -r hermes-skill-finder/ ~/.claude/skills/

# Hermes
cp -r hermes-skill-finder/ ~/.hermes/skills/
```

## 使用示例

```
用户: 我想做前端设计，有什么技能推荐？
用户: 我需要微调一个大模型
用户: 帮我找一个做代码审查的技能
用户: 有没有什么技能可以帮我管理 GitHub PR？
```

Agent 会自动触发此技能，实时获取技能列表，匹配后返回推荐结果。

## 数据源

从 Hermes 官方网站动态获取全部 680+ 技能数据（不使用本地缓存）：

| 来源 | 数量 | 说明 |
|------|------|------|
| Built-in | 89 | Hermes 内置技能 |
| Optional | 71 | 官方可选技能 |
| Anthropic | 16 | Anthropic 官方技能 |
| LobeHub | 504 | LobeHub 社区技能 |

技能数据来自网站构建后的 JS chunk（`skills.json` → webpack chunk 2013），通过 runtime.js 动态发现 chunk hash，网站重新构建后也能自动适配。

降级方案：chunk 获取失败时回退到 GitHub 仓库数据（约 174 个技能）。

## 项目结构

```
hermes-skill-finder/
├── SKILL.md      # Agent 技能定义文件（核心）
└── README.md     # 本文件
```

## 兼容性说明

- **SKILL.md 格式** 遵循 Agent Skills 开放标准，Claude Code、Hermes、Codex CLI 等均支持
- 推荐时会根据用户的 Agent 平台标注兼容性：
  - 纯知识型/指南型技能 → 所有平台兼容
  - 依赖 Hermes 专有工具的技能 → 仅 Hermes 兼容，会标注 `⚠️ Claude Code 兼容性有限`
