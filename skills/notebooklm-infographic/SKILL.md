---
name: infographic
description: 用 NotebookLM 一键生成学习笔记配图。自动处理认证、内容上传、图片生成、下载全流程。激活方式：/infographic 或 "生成配图"、"生成信息图"
---

# notebooklm-infographic: NotebookLM 配图生成器

用 NotebookLM 一键生成学习笔记配图。自动处理认证、内容上传、图片生成、下载全流程。

## 适用场景

- 生成小红书/公众号配图
- 学习笔记可视化
- 知识点速记卡片
- 考点汇总图

## 输出规格

| 参数 | 默认值 | 可选项 |
|------|--------|--------|
| 格式 | PNG | - |
| 方向 | landscape（横版） | landscape / portrait / square |
| 风格 | sketch-note（手绘） | 11种风格可选 |
| 语言 | zh_Hans（简体中文） | 80+语言可选 |
| 详细度 | standard | concise / standard / detailed |

**默认组合**（适合小红书/学习笔记）：
- `--orientation landscape` 横版
- `--style sketch-note` 手绘风格
- `--language zh_Hans` 中文

## 使用方式

### 方式一：传入 Markdown 文件

```
/infographic /path/to/content.md --output ~/Desktop/配图/标题.png
```

### 方式二：直接传入内容

```
/infographic "创建一张关于假设检验的信息图，包含：1. 五步法 2. α/β风险 3. P值判断" --output ~/Desktop/配图/假设检验.png
```

### 方式三：批量生成

```
/infographic --batch /path/to/folder/ --output-dir ~/Desktop/配图/
```

## 参数说明

| 参数 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `--output` | 否 | ~/Downloads/infographic_{timestamp}.png | 输出路径 |
| `--orientation` | 否 | landscape | 图片方向 |
| `--detail` | 否 | standard | 内容详细程度 |
| `--style` | 否 | sketch-note | 视觉风格 |
| `--language` | 否 | zh_Hans | 输出语言 |
| `--title` | 否 | 自动提取 | 配图标题 |
| `--notebook` | 否 | 自动创建或复用 | 指定 notebook ID |

---

### 📐 图片方向 `--orientation`

| 选项 | 尺寸 | 适用场景 |
|------|------|----------|
| `landscape` | 2752 x 1536 | **推荐** 横版，适合手机阅读、小红书配图 |
| `portrait` | 1536 x 2752 | 竖版，适合公众号长图、海报 |
| `square` | 1536 x 1536 | 正方形，适合 Instagram、朋友圈 |

---

### 📊 内容详细程度 `--detail`

| 选项 | 说明 | 适用场景 |
|------|------|----------|
| `concise` | 精简版，只保留核心要点 | 概念速记、单页总结 |
| `standard` | 标准版，平衡内容与留白 | **推荐** 大多数场景 |
| `detailed` | 详细版，包含完整内容 | 复杂知识点、完整流程图 |

---

### 🎨 视觉风格 `--style`

| 选项 | 风格描述 | 适用场景 |
|------|----------|----------|
| `sketch-note` | **推荐** 手绘风格，线条自然，适合学习笔记 | 学习笔记、知识卡片、小红书 |
| `professional` | 专业商务风格，排版规整 | 工作汇报、商业文档 |
| `scientific` | 科学论文风格，严谨正式 | 学术内容、研究报告 |
| `bento-grid` | 便当盒网格布局，模块化展示 | 多知识点并列、功能对比 |
| `editorial` | 杂志编辑风格，图文并茂 | 文章配图、故事叙述 |
| `instructional` | 教学指导风格，步骤清晰 | 教程、操作指南 |
| `bricks` | 砖块积木风格，童趣可爱 | 轻松话题、入门内容 |
| `clay` | 粘土风格，柔和立体 | 创意内容、品牌调性 |
| `anime` | 动漫风格，日系插画 | 年轻受众、娱乐内容 |
| `kawaii` | 可爱卡通风，圆润萌系 | 轻松话题、女性受众 |
| `auto` | 自动选择（由 AI 判断） | 不确定时使用 |

**风格预览建议**：首次使用新风格时，先生成一张测试，确认效果后再批量使用。

---

### 🌐 输出语言 `--language`

| 代码 | 语言 |
|------|------|
| `zh_Hans` | 中文（简体）**推荐** |
| `zh_Hant` | 中文（繁体） |
| `en` | English |
| `ja` | 日本語 |
| `ko` | 한국어 |
| `es` | Español |
| `fr` | Français |
| `de` | Deutsch |

完整语言列表：`notebooklm language list`

## 执行步骤

### 步骤 1：检查认证

```bash
notebooklm auth check --test --json
```

如果认证失败：
```bash
notebooklm login
```

### 步骤 2：准备 Notebook

```bash
# 查找现有 notebook
notebooklm list --json | jq '.notebooks[] | select(.title | contains("配图"))'

# 或创建新的
notebooklm create "配图生成工作台" --json
```

### 步骤 3：添加内容作为 Source

```bash
# 从文件添加
notebooklm source add /tmp/content.md --json

# 或创建临时文件
echo "# 标题\n\n内容..." > /tmp/content.md
notebooklm source add /tmp/content.md --json
```

### 步骤 4：等待 Source 处理

```bash
notebooklm source wait <source_id> --timeout 120
```

### 步骤 5：生成 Infographic

```bash
notebooklm generate infographic "描述内容要求" \
  --style sketch-note \
  --orientation landscape \
  --language zh_Hans \
  --json
```

### 步骤 6：等待生成完成

```bash
notebooklm artifact wait <task_id> --timeout 600
```

生成时间：通常 3-10 分钟

### 步骤 7：下载图片

```bash
notebooklm download infographic /path/to/output.png -a <artifact_id>
```

## 完整示例

### 单张配图

```bash
# 1. 检查认证
notebooklm auth check --test --json

# 2. 创建内容
cat > /tmp/fmea.md << 'EOF'
# FMEA 失效模式与影响分析

## 核心公式
RPN = S × O × D

## 行动标准
- RPN ≥ 120：必须采取措施
- S ≥ 8：不管 RPN 多低都必须改
EOF

# 3. 添加 source
SOURCE_ID=$(notebooklm source add /tmp/fmea.md --json | jq -r '.source.id')

# 4. 等待处理
notebooklm source wait $SOURCE_ID --timeout 120

# 5. 生成图片
TASK_ID=$(notebooklm generate infographic "创建FMEA信息图，包含RPN公式和行动标准" \
  --style sketch-note \
  --orientation landscape \
  --language zh_Hans \
  --json | jq -r '.task_id')

# 6. 等待完成
notebooklm artifact wait $TASK_ID --timeout 600

# 7. 下载
notebooklm download infographic ~/Desktop/FMEA.png -a $TASK_ID
```

### 批量生成

```bash
# 遍历目录下的 md 文件
for file in /path/to/contents/*.md; do
  filename=$(basename "$file" .md)

  SOURCE_ID=$(notebooklm source add "$file" --json | jq -r '.source.id')
  notebooklm source wait $SOURCE_ID --timeout 120

  TASK_ID=$(notebooklm generate infographic "创建信息图" \
    --style sketch-note \
    --orientation landscape \
    --language zh_Hans \
    --json | jq -r '.task_id')

  notebooklm artifact wait $TASK_ID --timeout 600
  notebooklm download infographic ~/Desktop/"$filename".png -a $TASK_ID

  # 避免触发 rate limit
  sleep 60
done
```

## 错误处理

| 错误 | 原因 | 解决方案 |
|------|------|----------|
| 认证失败 | Session 过期 | 运行 `notebooklm login` |
| Rate limit | 请求过于频繁 | 等待 5-10 分钟后重试 |
| 生成超时 | 内容太复杂 | 简化内容或延长时间 |
| 下载失败 | Artifact 未完成 | 检查 `artifact list` 状态 |

## 限制说明

- **每日配额**：约 5-8 张（受 Google 限制）
- **生成时间**：3-10 分钟/张
- **内容长度**：建议 < 2000 字

## 最佳实践

1. **内容准备**：用 Markdown 格式，结构清晰（标题、列表、表格）
2. **描述精准**：generate 时的描述越具体，生成效果越好
3. **批量间隔**：每张之间间隔 60 秒，避免 rate limit
4. **复用 Notebook**：避免频繁创建，用一个 notebook 多次添加 source

---

## 📁 图片保存位置

生成完成后，询问用户选择保存方式：

### 选项一：添加到笔记

将图片嵌入到指定的 Obsidian 笔记中：

```markdown
![[图片名称.png]]
```

**操作**：
1. 询问用户目标笔记路径
2. 将图片移动到 vault 的附件目录（如 `attachments/`）
3. 在目标笔记中插入 wikilink 引用

### 选项二：保存到电脑文件夹

将图片保存到指定的本地目录：

**常用目录示例**：
- `~/Desktop/配图/`
- `~/Downloads/`
- 用户指定的任意路径

**操作**：
1. 询问用户目标文件夹路径
2. 将图片移动到指定位置

---

## 交互流程

### 必须使用 AskUserQuestion 工具收集参数

**不要直接使用默认值，必须先询问用户偏好。**

### 第一轮询问

使用 `AskUserQuestion` 工具，4 个问题：

```
问题1：📐 图片方向选择哪个？
- landscape 横版（2752×1536，适合小红书/手机阅读）
- portrait 竖版（1536×2752，适合公众号长图/海报）
- square 正方形（1536×1536，适合朋友圈/Instagram）

问题2：🎨 视觉风格选择哪个？
- sketch-note 手绘（手绘风格，适合学习笔记）
- professional 专业（专业商务风格，排版规整）
- scientific 科学（科学论文风格，严谨正式）
- 更多风格...（查看其他7种风格）

问题3：🌐 输出语言选择哪个？
- zh_Hans 中文简体
- zh_Hant 中文繁体
- en English

问题4：📁 图片生成后保存到哪里？
- 添加到笔记（插入到当前笔记中）
- 保存到 Downloads
- 自定义文件夹
```

### 第二轮询问（仅当用户选择"更多风格..."时触发）

```
问题1：🎨 视觉风格（第2组）
- bento-grid 网格（便当盒网格布局，模块化展示）
- editorial 杂志（杂志编辑风格，图文并茂）
- instructional 教学（教学指导风格，步骤清晰）
- bricks 砖块（砖块积木风格，童趣可爱）

问题2：🎨 视觉风格（第3组）
- clay 粘土（粘土风格，柔和立体）
- anime 动漫（动漫风格，日系插画）
- kawaii 可爱（可爱卡通风，圆润萌系）
- auto 自动（自动选择，由 AI 判断）
```

### AskUserQuestion 调用示例

**第一轮：**

```json
{
  "questions": [
    {
      "header": "方向",
      "multiSelect": false,
      "options": [
        {"label": "landscape 横版", "description": "2752×1536，适合小红书/手机阅读"},
        {"label": "portrait 竖版", "description": "1536×2752，适合公众号长图/海报"},
        {"label": "square 正方形", "description": "1536×1536，适合朋友圈/Instagram"}
      ],
      "question": "📐 图片方向选择哪个？"
    },
    {
      "header": "风格",
      "multiSelect": false,
      "options": [
        {"label": "sketch-note 手绘", "description": "手绘风格，线条自然，适合学习笔记"},
        {"label": "professional 专业", "description": "专业商务风格，排版规整"},
        {"label": "scientific 科学", "description": "科学论文风格，严谨正式"},
        {"label": "更多风格...", "description": "查看更多风格：bento-grid/editorial/bricks/clay/anime/kawaii/instructional/auto"}
      ],
      "question": "🎨 视觉风格选择哪个？"
    },
    {
      "header": "语言",
      "multiSelect": false,
      "options": [
        {"label": "zh_Hans 中文简体", "description": "简体中文"},
        {"label": "zh_Hant 中文繁体", "description": "繁体中文"},
        {"label": "en English", "description": "英语"}
      ],
      "question": "🌐 输出语言选择哪个？"
    },
    {
      "header": "保存位置",
      "multiSelect": false,
      "options": [
        {"label": "添加到笔记", "description": "插入到当前笔记中"},
        {"label": "保存到 Downloads", "description": "保存到 ~/Downloads/ 目录"},
        {"label": "自定义文件夹", "description": "指定其他文件夹路径"}
      ],
      "question": "📁 图片生成后保存到哪里？"
    }
  ]
}
```

**第二轮（当用户选择"更多风格..."时）：**

```json
{
  "questions": [
    {
      "header": "风格",
      "multiSelect": false,
      "options": [
        {"label": "bento-grid 网格", "description": "便当盒网格布局，模块化展示"},
        {"label": "editorial 杂志", "description": "杂志编辑风格，图文并茂"},
        {"label": "instructional 教学", "description": "教学指导风格，步骤清晰"},
        {"label": "bricks 砖块", "description": "砖块积木风格，童趣可爱"}
      ],
      "question": "🎨 视觉风格（第2组）"
    },
    {
      "header": "风格",
      "multiSelect": false,
      "options": [
        {"label": "clay 粘土", "description": "粘土风格，柔和立体"},
        {"label": "anime 动漫", "description": "动漫风格，日系插画"},
        {"label": "kawaii 可爱", "description": "可爱卡通风，圆润萌系"},
        {"label": "auto 自动", "description": "自动选择，由 AI 判断"}
      ],
      "question": "🎨 视觉风格（第3组）"
    }
  ]
}
```

---

## 完整交互流程图

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 用户提供内容（文件或直接输入）                              │
│                                                             │
│ 2. 第一轮 AskUserQuestion                                    │
│    - 方向：landscape / portrait / square                     │
│    - 风格：常用4种 + "更多风格..."                             │
│    - 语言：zh_Hans / zh_Hant / en                            │
│    - 保存位置：笔记 / Downloads / 自定义                       │
│                                                             │
│ 3. 如果选择"更多风格..."，第二轮 AskUserQuestion               │
│    - 风格组2：bento-grid / editorial / instructional / bricks │
│    - 风格组3：clay / anime / kawaii / auto                   │
│                                                             │
│ 4. 生成图片（3-10分钟）                                       │
│                                                             │
│ 5. 根据保存位置处理                                           │
│    - 添加到笔记 → 复制到 attachments/ → 插入 ![[图片.png]]     │
│    - 保存到 Downloads → 保持 ~/Downloads/                     │
│    - 自定义文件夹 → 移动到指定路径                              │
└─────────────────────────────────────────────────────────────┘
```
