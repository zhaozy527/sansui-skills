# sansui-skills

我的 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 自定义技能集。

## 安装

使用 [skills CLI](https://github.com/vercel-labs/skills)（基于 `npx`）一行安装：

```bash
# 安装全部技能（全局）
npx skills add sansui/sansui-skills -g --all

# 安装单个技能
npx skills add sansui/sansui-skills -g --skill pptx-to-md
npx skills add sansui/sansui-skills -g --skill roundtable
npx skills add sansui/sansui-skills -g --skill social-cast
npx skills add sansui/sansui-skills -g --skill notebooklm-infographic
npx skills add sansui/sansui-skills -g --skill depth-interview-prep
```

**参数说明：**

| 参数 | 作用 |
|------|------|
| `-g` | 全局安装到 `~/.claude/skills/`（推荐）。不加则装到当前项目 `.claude/skills/` |
| `--skill <name>` | 指定安装某个技能，可重复使用 |
| `--all` | 安装仓库内全部技能 |

## 技能

| 技能 | 说明 |
|------|------|
| **pptx-to-md** | PPT/PPTX → 结构化 Markdown — 自动提取文字、表格、图表数据，嵌入截图，输出单文件 MD。适合 Obsidian 知识库、CEO 阅读、存档等场景 |
| **roundtable** | 圆桌会议 — 多嘉宾深度辩证讨论，内置「思辨模式」与「经营模式」双引擎。支持5维人物蒸馏、综合者重构、ASCII框架图，逐轮深入直至产出可行动的结构化知识 |
| **social-cast** | 社交分享卡片 — 把内容做成带水印的图片，发朋友圈/公众号。支持长文卡片、QA卡片、短句卡片，自动生成 PNG 并可选发布 |
| **notebooklm-infographic** | NotebookLM 配图生成器 — 一键生成学习笔记配图。11种视觉风格，3种图片方向，自动处理认证、内容上传、图片生成全流程 |
| **depth-interview-prep** | 深度访谈前准备工作流 — 研究问题澄清 → 招募对象设计 → 访谈大纲生成。内置 7 类问题 ❌/✅ 话术速查、三维度招募矩阵、执行检查单，产出可直接执行的访谈大纲 |
