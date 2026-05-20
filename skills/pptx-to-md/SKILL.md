---
name: pptx-to-md
description: |
  将 PPT/PPTX 文件转换为结构化 Markdown，用于 Obsidian 知识库、CEO 阅读、存档等场景。
  自动提取文字、表格、图表数据，嵌入截图，输出单文件 MD。

  **触发场景：** 用户有 PPTX 文件，想转成 Markdown 笔记、知识库文档、述职记录等。

  **触发词：** "转成 MD"、"PPT 转 Markdown"、"提取 PPT 内容"、"PPTX 转文字"、"存到 Obsidian"、"pptx to md"。
---

# PPTX → Markdown 转换 Skill

将 PPTX 演示稿转换为结构完整、信息无损的单文件 Markdown，适合 CEO 阅读/Obsidian 存档。

---

## Phase 0：确认输入和规格

### Step 0.1：收集必要信息

询问用户（一次性问完）：

1. PPTX 文件路径在哪里？
2. 输出 MD 的用途？（Obsidian / CEO 阅读 / 存档 / 其他）
3. 有没有特殊结构要求？（如按项目分节、特定 frontmatter 字段）

### Step 0.2：创建工作目录

```bash
# 在 PPTX 同级目录创建 assets 文件夹
PPTX_DIR=$(dirname "/path/to/file.pptx")
ASSETS_DIR="${PPTX_DIR}/$(basename file .pptx)_assets"
mkdir -p "$ASSETS_DIR"
```

---

## Phase 1：导出幻灯片截图

> **核心原则**：截图是保底兜底层，无论文字提取多完整，都必须保留每页截图（base64嵌入）。

### Step 1.1：AppleScript + PowerPoint 导出 PDF

```applescript
tell application "Microsoft PowerPoint"
    set pptFile to POSIX file "/absolute/path/to/file.pptx"
    open pptFile
    delay 3
    set pdfPath to "/absolute/path/to/assets/slides.pdf"
    save active presentation in pdfPath as save as PDF
    delay 2
    close active presentation saving no
end tell
```

**执行方式**：
```bash
osascript << 'EOF'
# 上面的 AppleScript 内容
EOF
```

> **常见坑**：路径必须用绝对路径，不能用 `~`；中文路径需要正确编码；`delay` 必须足够（大文件延长至5秒）。

### Step 1.2：PyMuPDF 拆分 PDF 为 PNG

```python
import fitz  # pip install pymupdf

doc = fitz.open('/path/to/assets/slides.pdf')
for i, page in enumerate(doc):
    mat = fitz.Matrix(2, 2)  # 2x 分辨率，输出 1440×810
    pix = page.get_pixmap(matrix=mat)
    pix.save(f'/path/to/assets/slide_{i+1:02d}.png')
print(f"共导出 {len(doc)} 页")
```

**验证**：
```bash
ls -la /path/to/assets/slide_*.png | wc -l
# 应等于 PPT 总页数
```

---

## Phase 2：提取 PPTX 文字内容

> **核心原则**：必须递归提取，否则嵌套在 GroupShape 内的文字会全部丢失。

### Step 2.1：递归提取函数（必用模板）

```python
from pptx import Presentation
from pptx.enum.shapes import MSO_SHAPE_TYPE

def extract_texts(shape, depth=0, parent_top=0, parent_left=0):
    """递归提取形状中的所有文字，包括嵌套 GroupShape"""
    results = []
    try:
        abs_top = parent_top + (shape.top if shape.top else 0)
        abs_left = parent_left + (shape.left if shape.left else 0)

        if shape.shape_type == MSO_SHAPE_TYPE.GROUP:
            # GroupShape：记录组的绝对位置，递归进入子形状
            g_top = shape.top if shape.top else 0
            g_left = shape.left if shape.left else 0
            for s in shape.shapes:
                results.extend(extract_texts(s, depth + 1, g_top, g_left))
        elif shape.has_text_frame:
            texts = [p.text.strip() for p in shape.text_frame.paragraphs
                     if p.text.strip()]
            if texts:
                results.append((abs_top, abs_left, depth, texts))
    except Exception:
        pass
    return results


def get_slide_texts(slide):
    """获取一页幻灯片的所有文字，按位置排序"""
    all_results = []
    for shape in slide.shapes:
        all_results.extend(extract_texts(shape))
    all_results.sort(key=lambda x: (x[0], x[1]))  # 按 top, left 排序
    return all_results


# 使用示例
prs = Presentation('/path/to/file.pptx')
for i, slide in enumerate(prs.slides):
    texts = get_slide_texts(slide)
    print(f"\n=== 第{i+1}页 ===")
    for top, left, depth, lines in texts:
        for line in lines:
            print(f"{'  '*depth}[{top//914:.0f},{left//914:.0f}] {line}")
```

> **坐标说明**：python-pptx 的 top/left 单位是 EMU（914400 EMU = 1英寸）。`top//914` 近似为毫英寸，用于判断元素在页面上的位置关系。

### Step 2.2：提取表格数据

```python
def extract_tables(slide):
    tables = []
    for shape in slide.shapes:
        if shape.has_table:
            rows = []
            for row in shape.table.rows:
                cells = [cell.text.strip() for cell in row.cells]
                rows.append(cells)
            tables.append({'name': shape.name, 'rows': rows})
    return tables
```

---

## Phase 3：提取图表数据

> **核心原则**：图表数据在 XML 中，用 `chart._chartSpace`（不是 `chart.chart_part`）访问。

### Step 3.1：图表数据提取函数

```python
from datetime import date, timedelta

def excel_to_date(n):
    """Excel 序列数转日期（如 45931 → 2025年10月）"""
    try:
        n = int(float(n))
        if 40000 <= n <= 60000:  # 合理的日期范围
            d = date(1899, 12, 30) + timedelta(days=n)
            return f"{d.year}年{d.month}月"
    except (ValueError, TypeError):
        pass
    return str(n)


def get_chart_data(chart_shape):
    """从图表形状提取系列和分类数据"""
    ns = "http://schemas.openxmlformats.org/drawingml/2006/chart"
    chart = chart_shape.chart
    chart_elem = chart._chartSpace  # 关键：用 _chartSpace 而非 chart_part

    # 提取分类（X轴）
    cat_names = []
    for v in chart_elem.findall(f'.//{{{ns}}}cat//{{{ns}}}v'):
        val = v.text or ''
        cat_names.append(excel_to_date(val) if val.isdigit() else val)

    # 提取系列
    series_data = []
    for ser in chart_elem.findall(f'.//{{{ns}}}ser'):
        # 系列名称
        tx = ser.find(f'.//{{{ns}}}tx//{{{ns}}}v')
        ser_name = tx.text if tx is not None else '数据'

        # 系列数值（yVal 用于散点图/折线图，val 用于柱状图）
        values = []
        for tag in ['yVal', 'val']:
            pts = ser.findall(f'.//{{{ns}}}{tag}//{{{ns}}}v')
            if pts:
                values = [p.text or '' for p in pts]
                break

        series_data.append({'name': ser_name, 'values': values})

    return {'categories': cat_names, 'series': series_data}


def chart_to_markdown(chart_shape):
    """将图表转换为 Markdown 表格"""
    try:
        data = get_chart_data(chart_shape)
        cats = data['categories']
        series = data['series']

        if not series:
            return None

        # 构建表头
        header = ['分类'] + [s['name'] for s in series]
        separator = ['---'] * len(header)

        # 构建数据行
        rows = []
        for i, cat in enumerate(cats):
            row = [cat]
            for s in series:
                val = s['values'][i] if i < len(s['values']) else ''
                row.append(val)
            rows.append(row)

        # 格式化为 Markdown 表格
        lines = ['| ' + ' | '.join(header) + ' |',
                 '| ' + ' | '.join(separator) + ' |']
        for row in rows:
            lines.append('| ' + ' | '.join(str(v) for v in row) + ' |')

        return '\n'.join(lines)
    except Exception as e:
        return None  # 无法提取时返回 None，改用截图
```

### Step 3.2：判断图表类型

```python
from pptx.enum.chart import XL_CHART_TYPE

VISUAL_ONLY_CHARTS = {
    # 瀑布图/桥梁图（无法用表格表达）
    XL_CHART_TYPE.WATERFALL,
}

def is_visual_only(chart_shape):
    """判断是否为纯视觉图表（无法转为表格）"""
    try:
        return chart_shape.chart.chart_type in VISUAL_ONLY_CHARTS
    except Exception:
        return False
```

---

## Phase 4：截图嵌入

```python
import base64

def slide_img_embed(slide_num, assets_dir, alt_text=""):
    """读取 PNG 截图，返回 base64 嵌入的 Markdown 图片"""
    img_path = f"{assets_dir}/slide_{slide_num:02d}.png"
    try:
        with open(img_path, 'rb') as f:
            b64 = base64.b64encode(f.read()).decode()
        return f"![{alt_text}](data:image/png;base64,{b64})"
    except FileNotFoundError:
        return f"<!-- 截图未找到: {img_path} -->"
```

---

## Phase 5：内容转换规则（10条）

每一页按以下规则处理，**规则之间有优先级**：

| 规则 | 页面类型 | 处理方式 |
|------|----------|----------|
| R1 | 所有页 | YAML frontmatter 放在文件开头 |
| R2 | 封面/目录页 | 仅提取文字，不附截图（无信息量） |
| R3 | 所有内容页 | 提取所有文字，不得遗漏 |
| R4 | 数据表格 | 转为标准 Markdown 表格 |
| R5 | 数据标签清晰的图表 | 转为 Markdown 表格 |
| R6 | 分布/对比类图表 | Markdown 表格 + 截图 |
| R7a | 单图页 | 直接嵌入截图 |
| R7b | 多图复杂页 | 先结构化文字，再附全页截图 |
| R8a | 流程图/逻辑图 | 文字描述 + 必须保留截图 |
| R8b | 多小图页 | 文字先行，全页截图兜底 |
| R9 | 所有页 | 关键数字/结论/口径一字不漏 |
| R10 | 输出 | 单文件 MD，图片 base64 内联，不打包 |

### 视觉页处理决策树

```
页面有图片？
├─ 是：图片是核心内容？
│   ├─ 是单张图（产品图/照片）→ 直接嵌入截图
│   └─ 多图/流程图/截图展示 → 文字结构化 + 全页截图
└─ 否：图表（Chart对象）？
    ├─ 可提取数据 → Markdown 表格（+ 截图可选）
    └─ 视觉专用（瀑布图等）→ 说明 + 截图
```

---

## Phase 6：MD 文件结构

### YAML Frontmatter 模板

```yaml
---
title: [报告标题]
date: YYYY-MM-DD
city: [城市]
report_type: [类型，如 述职汇报]
period: [汇报周期]
doc_source: [源文件名.pptx]
tags:
  - [标签1]
  - [标签2]
---
```

### 每页节段结构

```markdown
## [P{N}] {页面标题}

> **关键结论**：...（如果有）

### 小节标题

内容...

| 表格 | 数据 |
|------|------|
| ... | ... |

📸 **{截图说明}：**

![{alt}](data:image/png;base64,...)

---
```

### 内容层级约定

- `##` → 每页顶级标题（含 [P{N}] 页码标记）
- `###` → 页内分节
- `####` → 节内子项
- `>` → 核心结论/关键数字（引用块）
- **加粗** → 重要指标、关键动作
- `→` → 流程、转化、趋势
- 表格 → 所有结构化数据

---

## Phase 7：生成脚本结构

```python
# gen_md.py 核心结构
import base64
from pptx import Presentation
from pptx.enum.shapes import MSO_SHAPE_TYPE
from datetime import date, timedelta
import fitz  # PyMuPDF

PPTX_PATH = '/path/to/file.pptx'
ASSETS_DIR = '/path/to/assets'
OUTPUT_PATH = '/path/to/output.md'

prs = Presentation(PPTX_PATH)
sections = []

# YAML frontmatter
sections.append("""---
title: ...
date: ...
---
""")

# 逐页处理
for i, slide in enumerate(prs.slides):
    page_num = i + 1
    texts = get_slide_texts(slide)  # Phase 2
    tables = extract_tables(slide)  # Phase 2
    
    page_md = f"\n## [P{page_num}] {get_title(texts)}\n\n"
    
    # 提取文字内容
    page_md += format_texts(texts)
    
    # 提取表格
    for table in tables:
        page_md += format_table(table)
    
    # 提取图表
    for shape in slide.shapes:
        if shape.has_chart:
            chart_md = chart_to_markdown(shape)
            if chart_md:
                page_md += f"\n{chart_md}\n"
            else:
                # 视觉专用图表：只加截图
                page_md += "\n<!-- 视觉图表，见截图 -->\n"
    
    # 截图（按规则决定是否附加）
    if should_embed_screenshot(slide, page_num):
        page_md += f"\n{slide_img_embed(page_num, ASSETS_DIR)}\n"
    
    page_md += "\n---\n"
    sections.append(page_md)

with open(OUTPUT_PATH, 'w', encoding='utf-8') as f:
    f.write('\n'.join(sections))
```

---

## Phase 8：内容补全与修复

### 发现缺失内容时的修复流程

1. **用 PPTX 文字提取验证**（比对截图和文字提取结果）：
   ```python
   prs = Presentation(PPTX_PATH)
   slide = prs.slides[N-1]  # 第N页
   texts = get_slide_texts(slide)
   for top, left, depth, lines in texts:
       for t in lines:
           print(f"{'  '*depth}[{top//914:.0f},{left//914:.0f}] {t}")
   ```

2. **精准替换 MD 中的某页内容**（保留截图）：
   ```python
   with open(OUTPUT_PATH, 'r', encoding='utf-8') as f:
       content = f.read()

   # 定位当前页和下一页
   p_start = content.find(f'## [P{N}]')
   p_next = content.find(f'## [P{N+1}]')
   p_section = content[p_start:p_next]

   # 保留截图部分
   ss_marker = '📸'  # 或 '![' 
   ss_pos = p_section.find(ss_marker)
   ss_part = p_section[ss_pos:] if ss_pos != -1 else ''

   # 构建新内容（保留截图）
   new_section = f"""## [P{N}] {title}

{new_content}

""" + ss_part

   new_content_full = content[:p_start] + new_section + '\n' + content[p_next:]

   with open(OUTPUT_PATH, 'w', encoding='utf-8') as f:
       f.write(new_content_full)
   ```

### 各类内容的结构化建议

**流程/逻辑图页**：
```markdown
### 流程名称

**流程节点**：步骤A → 步骤B → 步骤C → 步骤D

📸 **流程图（必须保留截图）：**
![流程图](data:...)
```

**问题-解法-结果类页**（如述职中的项目页）：
```markdown
### 一、问题（客户痛点/现状）
| 痛点 | 描述 |
|------|------|

### 二、分析（根因/帕累托）
...

### 三、解法
| 问题 | 解法 |
|------|------|

### 四、落地策略
- **举措1**：...

### 五、结果变化
- **核心指标**：X → Y（+Z%）
```

**矩阵/战略框架页**：
```markdown
> **核心逻辑**：[一句话概括框架]

### 维度1（目标：...）
- 举措A / 举措B / 举措C

### 维度2（目标：...）
...

### 管理机制
| 抓手 | 具体内容 |
|------|----------|
```

---

## Phase 9：常见坑与解法

| 坑 | 原因 | 解法 |
|----|------|------|
| 文字提取不全 | 形状嵌套在 GroupShape 里 | 用递归提取函数（Phase 2） |
| `chart.chart_part` 报错 | API 变了 | 改用 `chart._chartSpace` |
| AppleScript 导出失败 | 路径含中文/空格，或 PowerPoint 未响应 | 用绝对 POSIX 路径，加 `delay` |
| Excel 序列日期变成数字 | 图表日期存为序列数（如 45931） | 用 `date(1899,12,30)+timedelta(days=n)` 转换 |
| 瀑布图/桥梁图无数据 | 图表类型不支持数据提取 | 标注 `<!-- 视觉图表 -->`，嵌入截图，手动补文字 |
| 脚本中文语法错误 | Bash heredoc 不支持全角符号 | 用 Write 工具写脚本文件，不要 heredoc |
| 修复某页破坏截图 | 替换了包含 base64 的段落 | 替换时先定位 `📸` 标记，单独保留截图部分 |
| 坐标顺序错乱 | GroupShape 内坐标是相对坐标 | 累加 parent_top/parent_left（见递归函数） |

---

## 快速启动检查清单

- [ ] 确认 PPTX 路径（绝对路径）
- [ ] 安装依赖：`pip install python-pptx pymupdf`
- [ ] PowerPoint 已安装（AppleScript 导出需要）
- [ ] 创建 assets 目录
- [ ] 运行 AppleScript 导出 PDF
- [ ] 运行 PyMuPDF 拆分 PNG（验证页数一致）
- [ ] 运行文字提取脚本（验证无遗漏）
- [ ] 运行 MD 生成脚本
- [ ] 按规则逐页检查，对照截图补全
- [ ] 最终验证：每页标题 `[P{N}]`，截图齐全，数字准确
