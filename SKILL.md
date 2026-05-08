---
name: hermes-skill-finder
description: Use when the user wants to find, discover, or install a skill/plugin for their AI agent. Triggers on "find skill", "找技能", "推荐插件", "install skill", "need a tool for", "有什么技能可以", or any request to discover capabilities from the Hermes skill marketplace.
version: 2.0.0
author: Ming-H
license: MIT
metadata:
  hermes:
    tags: [skill-discovery, skill-search, hermes, marketplace, plugin-finder]
    requires_toolsets: [web]
---

# Hermes Skill Finder

从 Hermes Agent 技能市场（2000+ 技能）中实时检索，根据用户需求推荐最合适的技能并给出安装方式。

## 工作流程

### Step 1: 确认用户的 Agent 平台

先确认用户使用的是哪个 Agent，这决定了安装命令格式和兼容性提示：

| Agent | 安装命令格式 | 兼容性 |
|-------|------------|--------|
| Claude Code | `npx skills add <identifier> -a claude-code -g` | 纯知识型技能完美兼容；依赖 Hermes 专有工具的技能可能受限 |
| Hermes | `hermes skills install <identifier>` | 全部兼容 |
| 通用（两个都装） | `npx skills add <identifier> --all` | 同上，按 Agent 分别说明 |

如果用户已经在对话中提到过自己的 Agent，直接使用；否则简短询问。

### Step 2: 获取全部技能数据（2000+）

通过官方 JSON API 一次请求获取全部技能：

```python
import urllib.request, json
data = json.loads(urllib.request.urlopen(
    "https://hermes-agent.nousresearch.com/docs/api/skills-index.json").read())
skills = data["skills"]  # 2000+ skills
```

API 地址：`https://hermes-agent.nousresearch.com/docs/api/skills-index.json`

公开免费，无需认证，CORS 无限制（`access-control-allow-origin: *`）。

每个技能包含以下字段：

| 字段 | 说明 | 用途 |
|------|------|------|
| `name` | 技能名称 | 展示 |
| `description` | 功能描述 | 搜索匹配 |
| `source` | 来源（official/skills.sh/lobehub/clawhub/github/claude-marketplace） | 信任度判断 |
| `trust_level` | 信任级别（builtin/trusted/community） | 推荐排序 |
| `identifier` | 安装标识符 | 直接用于安装命令 |
| `tags[]` | 标签列表 | 精准搜索匹配 |
| `repo` | GitHub 仓库 | 源码链接 |

来源分布：
- skills.sh: 1234 | lobehub: 500 | clawhub: 199 | official: 71 | github: 62 | claude-marketplace: 1

### 降级方案

如果官方 API 不可用，使用 GitHub 仓库数据（约 174 个技能）：

```bash
curl -s https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/reference/skills-catalog.md
curl -s https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/reference/optional-skills-catalog.md
```

### Step 3: 智能匹配

根据用户的自然语言需求，匹配最合适的 1-3 个技能。匹配策略：

1. **标签匹配（优先）**: 用 `tags[]` 字段精准匹配用户需求关键词
2. **分类路由**: 根据 `identifier` 中的分类路径锁定领域
3. **描述语义匹配**: 在 `description` 中搜索关键词
4. **名称匹配**: 直接匹配技能名称

匹配时考虑：
- **信任度排序**: builtin > trusted > community，优先推荐高质量技能
- 用户的具体使用场景
- 技能之间的互补关系（可以一起推荐）
- 用户的 Agent 平台兼容性

### Step 4: 返回推荐结果

输出格式：

```
为你推荐以下技能：

### 1. [技能名称]
**简介**: 一句话描述
**来源**: official / skills.sh / lobehub / clawhub / github
**信任级别**: builtin / trusted / community
**标签**: tag1, tag2, tag3
**安装**: `安装命令`
```

### 兼容性标注

根据用户的 Agent 平台标注：

**Claude Code 用户注意：**
- 纯知识型/指南型技能 → 完美兼容
- 依赖 Hermes 专有工具的技能可能受限：
  - 涉及 `terminal()` 调用、PTY/tmux 编排的技能
  - 需要 Hermes 内部 toolset（browser、vision 等）的技能
- 如有兼容性问题，附上 `⚠️ Claude Code 兼容性有限: <原因>` 说明

**Hermes 用户：** 全部兼容，无需额外标注。

### Step 5: 安装辅助

如果用户确认要安装，帮助执行安装命令。`identifier` 字段直接用于安装：

- Claude Code: `npx skills add <identifier> -a claude-code -g`
- Hermes: `hermes skills install <identifier>`
- 通用: `npx skills add <identifier> --all`

identifier 示例：
- 官方技能: `official/security/1password`
- GitHub 技能: `anthropics/skills/skills/frontend-design`
- skills.sh 技能: `skills-sh/<org>/<skill-name>`

## 注意事项

- 每次触发都重新获取数据，不使用缓存
- API 响应包含 `generated_at` 时间戳，可用于判断数据新鲜度
- 推荐精准（1-3 个），不让用户选择困难
- 优先推荐 trust_level 高的技能（builtin > trusted > community）
- 没有匹配到时诚实告知，建议浏览 https://hermes-agent.nousresearch.com/docs/skills
