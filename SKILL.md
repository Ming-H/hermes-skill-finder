---
name: agent-skill-finder
description: Use when the user wants to find, discover, or install a skill/plugin for their AI agent. Triggers on "find skill", "找技能", "推荐插件", "install skill", "need a tool for", "有什么技能可以", or any request to discover capabilities from agent skill marketplaces.
version: 2.1.0
author: Ming-H
license: MIT
metadata:
  hermes:
    tags: [skill-discovery, skill-search, agent, marketplace, plugin-finder]
    requires_toolsets: [web]
---

# Agent Skill Finder

从 Agent 技能市场（2000+ 技能）中实时检索，根据用户需求推荐最合适的技能并给出安装方式。

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

使用双源混合策略，确保覆盖完整且搜索质量高：

**数据源 A: 官方 JSON API（2000+ 社区技能）**

```python
import urllib.request, json
api_data = json.loads(urllib.request.urlopen(
    "https://hermes-agent.nousresearch.com/docs/api/skills-index.json").read())
community_skills = api_data["skills"]  # 2067 skills
```

API 地址：`https://hermes-agent.nousresearch.com/docs/api/skills-index.json`

公开免费，无需认证，CORS 无限制。

覆盖来源：skills.sh (1234)、lobehub (500)、clawhub (199)、github (62)、claude-marketplace (1)。

**数据源 B: GitHub 仓库目录（153 个高质量内置技能）**

```bash
# Built-in 技能 (89 个，有完整描述)
curl -s https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/reference/skills-catalog.md

# Optional 技能 (64 个，有完整描述)
curl -s https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/reference/optional-skills-catalog.md
```

这是数据源 A 的**必要补充**。API 缺失了 Hermes 全部 89 个 Built-in 技能（如 `github-code-review`、`spotify`、`unsloth`、`claude-design` 等），这些是质量最高、最常用的技能，必须从 GitHub 目录获取。目录中的 Markdown 表格格式为：

```
| [`skill-name`](link) | 描述文字 | `category/skill-name` |
```

**合并策略：**

1. 先从数据源 B 获取内置技能（完整描述，trust_level=builtin）
2. 再从数据源 A 获取社区技能
3. 以数据源 B 的内置技能为准（覆盖 API 中可能存在的同名旧版本）
4. 推荐排序：数据源 B 的 builtin > 数据源 A 的 trusted > community

**数据源 A 已知问题（推荐时注意）：**

| 问题 | 影响 | 处理方式 |
|------|------|---------|
| skills.sh 1234 个条目无实际描述（仅 `"Indexed by skills.sh from <repo>"`） | 搜索匹配无效 | 跳过无描述的 skills.sh 条目，或在描述中提取 repo 名称作为线索 |
| 91 个技能名重复（如 `frontend-design` 出现 11 次） | 推荐时混淆 | 同名技能只推荐 trust_level 最高的那个 |
| clawhub identifier 使用裸名称格式 | 安装命令可能不兼容 | 推荐 clawhub 技能时提示用户确认安装方式 |

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

## 已知限制

### API 缺失 Hermes Built-in 技能
官方 JSON API 不包含 Hermes 的 89 个 Built-in 技能（如 `github-code-review`、`spotify`、`claude-design`、`unsloth` 等）。这些是最常用、质量最高的技能，必须从 GitHub 仓库目录获取。已在 Step 2 中通过双源混合策略解决。

### skills.sh 技能无实际描述
API 中 1234 个 skills.sh 条目的 description 仅为 `"Indexed by skills.sh from <repo>"` 或 `"Featured on skills.sh from <repo>"`，无法用于搜索匹配。推荐时跳过这些条目，或从 repo 名称提取线索。

### 同名技能重复
91 个技能名在 API 中重复（如 `frontend-design` 出现 11 次）。推荐时只取 trust_level 最高的版本，不重复推荐。

### clawhub identifier 格式
clawhub 来源（199 个）的 identifier 使用裸名称格式（如 `1password-browser-login`），缺少标准路径前缀。推荐时提示用户确认安装方式。
