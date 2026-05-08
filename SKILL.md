---
name: hermes-skill-finder
description: Use when the user wants to find, discover, or install a skill/plugin for their AI agent. Triggers on "find skill", "找技能", "推荐插件", "install skill", "need a tool for", "有什么技能可以", or any request to discover capabilities from the Hermes skill marketplace.
---

# Hermes Skill Finder

从 Hermes Agent 技能市场（680+ 技能）中实时检索，根据用户需求推荐最合适的技能并给出安装方式。

## 工作流程

### Step 1: 确认用户的 Agent 平台

先确认用户使用的是哪个 Agent，这决定了安装命令格式和兼容性提示：

| Agent | 安装命令格式 | 兼容性 |
|-------|------------|--------|
| Claude Code | `npx skills add <identifier> -a claude-code -g` | 纯知识型技能完美兼容；依赖 Hermes 专有工具的技能可能受限 |
| Hermes | `hermes skills install <identifier>` | 全部兼容 |
| 通用（两个都装） | `npx skills add <identifier> --all` | 同上，按 Agent 分别说明 |

如果用户已经在对话中提到过自己的 Agent，直接使用；否则简短询问。

### Step 2: 获取全部技能数据（680+）

网站是 Docusaurus 站点，技能数据打包在一个 JS chunk 中。用以下 Python 脚本一步完成动态发现和数据提取：

```python
import urllib.request, re

# 1. 获取页面 HTML，提取 runtime.js URL
html = urllib.request.urlopen("https://hermes-agent.nousresearch.com/docs/skills").read().decode()

# 2. 下载 runtime.js，提取 chunk 2013 的两个 hash
runtime_hash = re.search(r'runtime~main\.([a-f0-9]+)\.js', html).group(1)
runtime_js = urllib.request.urlopen(f"https://hermes-agent.nousresearch.com/docs/assets/js/runtime~main.{runtime_hash}.js").read().decode()
hashes = re.findall(r'"?2013"?:\s*"([a-f0-9]{8})"', runtime_js)
content_hash, runtime_hash_2013 = hashes[0], hashes[1]

# 3. 下载技能数据 chunk
chunk = urllib.request.urlopen(f"https://hermes-agent.nousresearch.com/docs/assets/js/{content_hash}.{runtime_hash_2013}.js").read().decode()

# 4. 提取技能列表
def extract_field(block, field):
    m = re.search(rf'"{field}":"((?:[^"\\]|\\.)*)"', block)
    return m.group(1).replace('\\"', '"') if m else ""

positions = [m.start() for m in re.finditer(r'\{"name":', chunk)]
skills = []
for i in range(len(positions)):
    start = positions[i]
    end = positions[i+1]-2 if i+1 < len(positions) else len(chunk)
    block = chunk[start:end]
    skills.append({"name": extract_field(block, "name"),
                   "description": extract_field(block, "description"),
                   "category": extract_field(block, "category"),
                   "categoryLabel": extract_field(block, "categoryLabel"),
                   "source": extract_field(block, "source")})
# skills 即为全部 680+ 技能列表
```

每个技能包含：`name`（名称）、`description`（描述）、`category`（分类 key）、`categoryLabel`（分类显示名）、`source`（来源）。

**原理说明：** 网站构建时，`src/data/skills.json` 被打包进 webpack chunk 2013。runtime.js 中有两张 hash 表：第一张映射 chunk ID → content hash，第二张映射 chunk ID → runtime hash。最终 URL 为 `contentHash.runtimeHash.js`。当网站重新构建时 hash 会变化，此脚本动态发现新 hash。

### 降级方案

如果 chunk 获取失败，使用以下数据源（覆盖约 174 个技能）：

```bash
# Built-in 技能 (~89)
curl -s https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/reference/skills-catalog.md

# Optional 技能 (~64)
curl -s https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/reference/optional-skills-catalog.md

# 社区技能索引
curl -s https://raw.githubusercontent.com/NousResearch/hermes-agent/main/skills/index-cache/anthropics_skills_skills_.json
```

### Step 3: 分类对照

技能涵盖以下分类：

| 分类 | 说明 |
|------|------|
| Apple | macOS 原生集成（Notes、Reminders、FindMy、iMessage） |
| AI Agents | 编码 Agent 委托（Claude Code、Codex、OpenCode） |
| Creative | 设计/创意（架构图、ComfyUI、Excalidraw、p5.js、设计系统） |
| MLOps | 机器学习（训练、推理、评估、向量数据库） |
| GitHub | PR review、Issues、代码审查、仓库管理 |
| Media | Spotify、YouTube、GIF、AI 音乐 |
| Productivity | Airtable、Google Workspace、Linear、Notion |
| Research | arXiv、论文写作、OSINT、生物信息 |
| Software Dev | TDD、调试、计划、代码审查 |
| Security | 1Password、取证、用户名搜索 |
| DevOps | Kanban、Docker、Webhook |
| MCP | FastMCP、MCPorter |
| 其他 | Blockchain、Health、Gaming、Smart Home、Social Media、Email |

### Step 4: 智能匹配

根据用户的自然语言需求，匹配最合适的 1-3 个技能。匹配策略：

1. **分类路由**: 根据需求所属领域锁定相关分类
2. **关键词匹配**: 需求关键词与技能名称和描述匹配
3. **语义理解**: 理解用户真正想要什么，即使表达方式与技能描述不同

匹配时考虑：
- 用户的具体使用场景
- 技能之间的互补关系（可以一起推荐）
- 用户的 Agent 平台兼容性

### Step 5: 返回推荐结果

输出格式：

```
为你推荐以下技能：

### 1. [技能名称]
**简介**: 一句话描述
**分类**: 分类标签
**来源**: Built-in / Optional / Anthropic / LobeHub
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

### Step 6: 安装辅助

如果用户确认要安装，帮助执行安装命令。identifier 规则：
- Built-in/Optional: `official/<category>/<name>` (Hermes) 或 `NousResearch/hermes-agent/skills/<category>/<name>` (npx)
- 社区技能: 按来源确定（如 `anthropics/skills/skills/<name>`）

## 注意事项

- 每次触发都重新获取数据，不使用缓存
- JS chunk 的 hash 会在网站重新构建时变化，需要动态发现
- 推荐精准（1-3 个），不让用户选择困难
- 没有匹配到时诚实告知，建议浏览 https://hermes-agent.nousresearch.com/docs/skills
