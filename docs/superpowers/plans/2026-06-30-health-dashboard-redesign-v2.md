# 健康管理态势感知看板 Skill 改进实现计划（v2）

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 修好现有 health-dashboard skill 的 4 个问题（图表截断、分析不准、数值不一致、大数据丢失），重写为全中文 skill 插件。

**架构：** 单 SKILL.md（全流程指令）+ references/（3 个纯文本规范）。AI 用 opencode 原生工具直接操作，不生成临时脚本程序。借鉴 sourceCPT 命令执行灵活度分级——Excel 读取/聚合半开（AI 按环境自适应），写盘/校验死命令，语义分析 AI 自主。

**技术栈：** ECharts 5（CDN）、PowerShell/Python/Node（AI 按环境自适应选一个）、opencode 原生工具（Glob/Grep/Read/Write/Edit/Bash）

**设计规格：** `docs/superpowers/specs/2026-06-30-health-dashboard-redesign-v2.md`

**测试数据：** `/root/health_data/` 下 4 个 Excel 文件

---

## 文件结构

| 文件 | 职责 | 操作 |
|------|------|------|
| `skills/health-dashboard/SKILL.md` | Skill 主文件，全流程指令，全中文 | 重写 |
| `skills/health-dashboard/references/echarts-config.md` | ECharts 防截断配置规范 | 创建 |
| `skills/health-dashboard/references/aggregate-json-schema.md` | 聚合数据结构定义 | 创建 |
| `skills/health-dashboard/references/result-excel-structure.md` | 结果 Excel sheet 结构定义 | 保留现有（已是 .md） |
| `skills/health-dashboard/reference/` | 旧代码骨架目录 | 删除整个目录 |

所有文件位于 `/root/.config/opencode/skills/health-dashboard/` 下。

---

## 任务 1：清理旧文件

**文件：**
- 删除：`/root/.config/opencode/skills/health-dashboard/reference/`（整个目录）

- [ ] **步骤 1：删除旧 reference 目录**

```bash
rm -rf /root/.config/opencode/skills/health-dashboard/reference/
```

- [ ] **步骤 2：创建新 references 目录**

```bash
mkdir -p /root/.config/opencode/skills/health-dashboard/references/
```

- [ ] **步骤 3：把现有 result-excel-structure.md 移到新目录**

```bash
# 旧文件已删除，需重新创建到新目录
# 如果旧文件还在 reference/ 下则移动，否则在任务3中重新创建
```

- [ ] **步骤 4：Commit**

```bash
cd /root && git add -A .config/opencode/skills/health-dashboard/
git commit -m "refactor(health-dashboard): 删除旧代码骨架目录，创建references目录"
```

---

## 任务 2：创建 references/echarts-config.md

**文件：**
- 创建：`/root/.config/opencode/skills/health-dashboard/references/echarts-config.md`

- [ ] **步骤 1：编写 ECharts 配置规范**

完整文件内容：

```markdown
# ECharts 防截断配置规范

> 本文件定义 health-dashboard 看板中所有 ECharts 图表必须遵循的配置规范。
> AI 生成 HTML 看板时必须 Read 本文件并严格遵循。

## ECharts 引入

```html
<script src="https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js"></script>
```

如用户要求离线，AI 下载 echarts.min.js 内嵌到 HTML 的 `<script>` 标签中。

## 通用防截断配置（所有图表必须应用）

| 配置项 | 值 | 作用 |
|--------|-----|------|
| `grid.containLabel` | `true` | 网格自动包含轴标签，不被裁剪 |
| `grid.left` | `"3%"` 或 `"8%"` | 左侧留白 |
| `grid.right` | `"4%"` 或 `"18%"`（横向条形图留更多） | 右侧留白 |
| `grid.top` | `"15%"` | 顶部留白给柱顶标签 |
| `grid.bottom` | `"8%"` 或 `"12%"` | 底部留白 |
| `yAxis.boundaryGap` | `["0", "20%"]` | Y轴顶部留20%空间给柱顶标签 |
| `series.label.show` | `true` | 显示数据标签 |
| `xAxis.axisLabel.interval` | `0` | 强制显示所有标签 |
| `xAxis.axisLabel.rotate` | `30`（柱状图）或 `0`（横向条形图） | X轴标签倾斜防重叠 |
| `xAxis.axisLabel.overflow` | `"truncate"` | 超长标签截断 |
| `xAxis.axisLabel.width` | `80` | 限制标签宽度 |
| `tooltip.confine` | `true` | 提示框不溢出容器 |
| 图表容器 CSS | `min-height: 400px` | 每图最小高度，防矮图挤压 |

## 图表类型 → 配置映射

### 柱状图（医生投入排行等对比场景）

```javascript
{
  grid: { containLabel: true, left: '3%', right: '4%', top: '15%', bottom: '8%' },
  xAxis: {
    type: 'category',
    data: [...],
    axisLabel: { interval: 0, rotate: 30, overflow: 'truncate', width: 80 }
  },
  yAxis: { type: 'value', boundaryGap: ['0', '20%'] },
  series: [{
    type: 'bar',
    label: { show: true, position: 'top', fontSize: 11 },
    itemStyle: { color: '#2E86C1' }
  }]
}
```

### 横向条形图（地区分布等长标签场景）

```javascript
{
  grid: { containLabel: true, left: '3%', right: '18%', top: '10%', bottom: '10%' },
  xAxis: { type: 'value' },
  yAxis: {
    type: 'category',
    data: [...],
    axisLabel: { width: 100, overflow: 'truncate' }
  },
  series: [{
    type: 'bar',
    label: { show: true, position: 'right', fontSize: 11 },
    itemStyle: { color: '#2E86C1' }
  }]
}
```

### 饼图（风险等级构成等占比场景）

```javascript
{
  series: [{
    type: 'pie',
    radius: ['35%', '65%'],
    data: [...],
    label: {
      show: true,
      formatter: '{b}: {c} ({d}%)',
      overflow: 'truncate'
    },
    itemStyle: { borderColor: '#fff', borderWidth: 2 }
  }]
}
```

### 折线图（月度趋势等时间序列场景）

```javascript
{
  grid: { containLabel: true, left: '3%', right: '4%', top: '15%', bottom: '12%' },
  xAxis: {
    type: 'category',
    data: [...],
    axisLabel: { interval: 0, rotate: 30 }
  },
  yAxis: { type: 'value', boundaryGap: ['0', '15%'] },
  series: [{
    type: 'line',
    symbolSize: 8,
    label: { show: true, position: 'top' },
    itemStyle: { color: '#2E86C1' },
    markPoint: { data: [{ type: 'max' }, { type: 'min' }] }
  }]
}
```

### 堆叠柱状图（风险等级月度趋势等分层场景）

同柱状图配置，series 中每个系列加 `stack: 'total'`。

### 热力图（医生×地区矩阵等交叉分析场景）

```javascript
{
  grid: { containLabel: true, left: '10%', right: '12%', top: '8%', bottom: '15%' },
  xAxis: { type: 'category', data: [...], axisLabel: { interval: 0, rotate: 30 } },
  yAxis: { type: 'category', data: [...], axisLabel: { width: 60, overflow: 'truncate' } },
  visualMap: { min: 0, max: N, calculable: true, orient: 'horizontal', left: 'center', bottom: '2%' },
  series: [{
    type: 'heatmap',
    data: [...],
    label: { show: true }
  }]
}
```

## 配色规范

| 元素 | 颜色 |
|------|------|
| 主色 | `#2E86C1`（医蓝） |
| 深色 | `#1A5276` |
| 标题栏背景 | `linear-gradient(135deg, #2E86C1, #1A5276)` |
| 卡片背景 | `#fff` |
| 卡片阴影 | `0 2px 12px rgba(0,0,0,0.08)` |
| 文字解读区背景 | `#F4F6F7` |
| 文字解读区左边框 | `4px solid #2E86C1` |
| 正文文字 | `#333` |
| 次要文字 | `#666` |

## 响应式规范

```css
@media (max-width: 768px) {
  .chart-grid { grid-template-columns: 1fr; }
  .kpi-grid { grid-template-columns: 1fr; }
  .chart-container { min-height: 400px; }
}
```

```javascript
window.addEventListener('resize', function() {
  Object.keys(chartInstances).forEach(function(id) {
    if (chartInstances[id]) chartInstances[id].resize();
  });
});
```
```

- [ ] **步骤 2：Commit**

```bash
cd /root && git add .config/opencode/skills/health-dashboard/references/echarts-config.md
git commit -m "feat(health-dashboard): 添加ECharts防截断配置规范"
```

---

## 任务 3：创建 references/aggregate-json-schema.md

**文件：**
- 创建：`/root/.config/opencode/skills/health-dashboard/references/aggregate-json-schema.md`

- [ ] **步骤 1：编写聚合数据结构定义**

完整文件内容：

```markdown
# 聚合数据结构定义

> 本文件定义 health-dashboard 数据聚合后的结构。
> AI 生成聚合命令时必须 Read 本文件，输出必须符合此结构。

## 顶层结构

聚合数据包含 4 个顶层字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `meta` | object | 元信息：文件名、总记录数、时间范围、区域/医生列表 |
| `quality` | object | 数据质量：输入行数、过滤行数、checksum 校验结果 |
| `aggregations` | object | 各维度聚合数据 |
| `summary` | object | 机械统计摘要，AI 做语义分析时的数据源 |

## meta 字段

| 子字段 | 类型 | 说明 |
|--------|------|------|
| `files` | string[] | 文件名列表 |
| `total_records` | integer | 总记录数（过滤后） |
| `date_range` | {start: string, end: string} | 时间范围 |
| `regions` | string[] | 区域列表 |
| `doctors` | string[] | 医生列表 |
| `columns_identified` | object | 各文件列识别结果 |

## quality 字段

| 子字段 | 类型 | 说明 |
|--------|------|------|
| `total_in` | integer | 输入总行数 |
| `total_after_filter` | integer | 过滤后行数 |
| `filtered_rows` | integer | 被过滤行数 |
| `filter_reasons` | string[] | 过滤原因列表 |
| `nulls_per_column` | object | 每列空值数 |
| `checksum` | {sum_groups: integer, match: boolean} | 校验结果 |

## aggregations 字段

| 子字段 | 类型 | 说明 |
|--------|------|------|
| `doctor_workload` | array | 医生投入 `[{name, records, patients, regions}]`，按 records 降序 |
| `region_distribution` | array | 地区分布 `[{region, records, patients, doctors}]` |
| `monthly_trend` | array | 月度趋势 `[{month, count, patients}]`，按月份升序 |
| `risk_level_dist` | array | 风险等级 `[{level, count, pct}]` |
| `diagnosis_dist` | array | 诊断分布 `[{diagnosis, count, pct}]`，按 count 降序 |
| `numeric_stats` | object | 数值统计 `{指标名: {mean, median, std, abnormal_rate, abnormal_count}}` |
| `top3_doctors` | {names: string[], values: integer[]} | 医生 Top3 |
| `top3_diagnoses` | {names: string[], values: integer[]} | 诊断 Top3 |
| `doctor_region_matrix` | {doctors, regions, matrix} | 医生×地区矩阵 |
| `risk_trend` | array | 风险趋势 `[{month, 各风险等级人数}]` |

## summary 字段（机械统计，AI 语义分析的数据源）

summary 是基础机械统计结果，**不是预计算洞察**。交叉洞察、规律发现、异常识别由 AI 语义分析完成。

| 子字段 | 类型 | 说明 |
|--------|------|------|
| `total_records` | integer | 总记录数 |
| `total_patients` | integer | 总患者数 |
| `total_doctors` | integer | 总医生数 |
| `total_regions` | integer | 总区域数 |
| `top5_doctors` | {names: string[], values: integer[]} | 医生 Top5（含具体值） |
| `top5_diagnoses` | {names: string[], values: integer[]} | 诊断 Top5（含具体值） |
| `top5_regions` | {names: string[], values: integer[]} | 地区 Top5（含具体值） |
| `risk_distribution` | {level: count} | 各风险等级人数 |
| `abnormal_rate_top5` | {names: string[], values: number[]} | 异常率前5指标（含异常率百分比） |
| `trend_direction` | string | 趋势方向："上升"/"下降"/"持平" |
| `trend_pct_change` | number | 变化幅度百分比 |
| `trend_peak_month` | {month: string, count: integer} | 峰值月份 |
| `trend_valley_month` | {month: string, count: integer} | 低谷月份 |

## 完整示例

```json
{
  "meta": {
    "files": ["门诊记录.xlsx", "健康咨询.xlsx"],
    "total_records": 570,
    "date_range": {"start": "2025-01-01", "end": "2025-06-28"},
    "regions": ["北京朝阳区", "上海浦东新区", "广州天河区"],
    "doctors": ["张伟明", "李芳华", "王建国"],
    "columns_identified": {}
  },
  "quality": {
    "total_in": 570,
    "total_after_filter": 568,
    "filtered_rows": 2,
    "filter_reasons": ["BMI>100 过滤1行", "血压负值过滤1行"],
    "checksum": {"sum_groups": 568, "match": true}
  },
  "aggregations": {
    "doctor_workload": [
      {"name": "张伟明", "records": 45, "patients": 38, "regions": 3}
    ],
    "top3_doctors": {"names": ["张伟明", "李芳华", "王建国"], "values": [45, 38, 35]}
  },
  "summary": {
    "total_records": 570,
    "total_patients": 420,
    "total_doctors": 7,
    "total_regions": 6,
    "top5_doctors": {"names": ["张伟明", "李芳华", "王建国", "陈秀英", "刘志强"], "values": [45, 38, 35, 30, 28]},
    "trend_direction": "上升",
    "trend_pct_change": 12.5,
    "trend_peak_month": {"month": "2025-06", "count": 120},
    "trend_valley_month": {"month": "2025-02", "count": 40}
  }
}
```
```

- [ ] **步骤 2：Commit**

```bash
cd /root && git add .config/opencode/skills/health-dashboard/references/aggregate-json-schema.md
git commit -m "feat(health-dashboard): 添加聚合数据结构定义"
```

---

## 任务 4：创建 references/result-excel-structure.md

**文件：**
- 创建：`/root/.config/opencode/skills/health-dashboard/references/result-excel-structure.md`

- [ ] **步骤 1：编写结果 Excel 结构定义**

完整文件内容：

```markdown
# 结果 Excel 文件结构

> 本文件定义 health-dashboard 结果 Excel 的 sheet 结构。
> AI 生成结果 Excel 时必须 Read 本文件并遵循。

## Sheet 列表

| Sheet 名 | 内容 | 数据源字段 | 列名 |
|----------|------|-----------|------|
| 概览 | 总体统计 | meta + summary | 指标, 数值 |
| 医生投入 | 医生工作量 | aggregations.doctor_workload | 医生姓名, 记录数, 患者数, 覆盖地区数 |
| 地区分布 | 地区统计 | aggregations.region_distribution | 地区, 记录数, 患者数, 医生数 |
| 月度趋势 | 时间趋势 | aggregations.monthly_trend | 月份, 记录数, 患者数 |
| 风险等级 | 风险构成 | aggregations.risk_level_dist | 风险等级, 人数, 占比(%) |
| 诊断分布 | 疾病排名 | aggregations.diagnosis_dist | 诊断, 人数, 占比(%) |
| 指标统计 | 数值统计 | aggregations.numeric_stats | 指标名, 均值, 中位数, 标准差, 异常率(%), 异常人数 |
| 医生地区矩阵 | 交叉分析 | aggregations.doctor_region_matrix | 首行=地区名, 首列=医生名, 值=记录数 |
| 风险趋势 | 堆叠趋势 | aggregations.risk_trend | 月份, 低风险, 中风险, 中高风险, 高风险 |
| 数据质量 | 校验信息 | quality | 项目, 值 |

## 生成规则

1. Sheet 数量随用户选的视图动态增减
2. 每个 sheet 首行为列名，后续为数据行
3. 数值与 HTML 中嵌入的数据、MD 报告完全同源
4. Excel 写出由 AI 用 opencode 原生工具完成，按实际环境选可用方式

## 各环境写出方式（参考，AI 自适应）

- **Python（如有）**: `pd.DataFrame(data).to_excel(writer, sheet_name='xxx')`
- **Node.js（如有）**: `xlsx.utils.book_append_sheet(wb, ws, 'xxx')` 然后 `xlsx.writeFile(wb, path)`
- **PowerShell（Windows 自带）**: 构建 CSV 再转 xlsx，或用 COM 对象 `New-Object -ComObject Excel.Application`
- **其他方式**: AI 按实际环境选可用方式，不限于以上三种
```

- [ ] **步骤 2：Commit**

```bash
cd /root && git add .config/opencode/skills/health-dashboard/references/result-excel-structure.md
git commit -m "feat(health-dashboard): 添加结果Excel结构定义"
```

---

## 任务 5：重写 SKILL.md

**文件：**
- 重写：`/root/.config/opencode/skills/health-dashboard/SKILL.md`

- [ ] **步骤 1：编写完整 SKILL.md**

完整文件内容（全中文，~350 行）：

```markdown
---
name: health-dashboard
description: >
  健康管理态势感知看板技能（opencode SKILL）。对员工健康管理 Excel 数据做态势感知分析，
  生成自包含交互式 HTML 看板 + Markdown 分析报告 + 结果 Excel 文件。
  AI 用 opencode 原生工具直接操作，不生成临时脚本程序。
  AI 做语义分析——发现规律、识别异常、交叉洞察、趋势判断、生成干预建议。
  使用场景：对健康管理 Excel 数据做态势感知分析，生成交互式看板。
  不使用场景：数据在数据库中而非 Excel 文件、非健康/医疗领域的数据分析。
---

# 健康管理态势感知看板

## 概述

本 skill 指导 AI 对员工健康管理 Excel 数据做态势感知分析，生成自包含交互式 HTML 看板 + Markdown 报告 + 结果 Excel 文件。

**核心原则：**
1. **聚合数据是唯一真相源** — 图表、表格、文字、Excel 四者都从它派生
2. **AI 做语义分析** — 发现规律、识别异常、交叉洞察、趋势判断、写态势感知是 AI 的核心能力，工具做不到
3. **数据落地文件** — 聚合数据写磁盘，不靠 AI 上下文维持，抗会话压缩
4. **校验断言** — 聚合后必须 checksum 校验，通过后才继续，校验失败不生成错误看板
5. **ECharts 替代 Plotly** — 自动标签避让，防截断（详见 references/echarts-config.md）
6. **启动时用户确认视图** — 先问后做，不全自动
7. **命令执行灵活度分级** — Excel 读取/聚合半开（AI 按环境自适应），写盘/校验死命令，语义分析 AI 自主

**流程图：**

```
用户提供 Excel 路径
    │
    ▼ AI 检测平台环境（bash / PowerShell / 其他），按实际选可用方式
    ▼ AI 扫描文件 + 语义理解列含义（自动）
    ▼ AI 列出候选视图清单 → 用户多选确认
    ▼ AI 用 opencode 原生工具把 Excel 转为可读格式 + 基础计数聚合
    ▼ 聚合数据写盘 + checksum 校验
    ▼ AI 读聚合数据，做语义分析（发现规律、识别异常、交叉洞察）
    ▼ AI 用 Write 工具写 HTML 看板（内嵌 ECharts + 聚合数据 + 态势分析文字）
    ▼ AI 用 Write 工具写 Markdown 报告
    ▼ AI 按实际环境选可用方式写结果 Excel
    ▼ 输出结果路径给用户
```

---

## 何时使用

**触发场景：**
- 用户要求分析健康/医疗 Excel 数据
- 用户要求生成健康管理看板、态势感知报告
- 关键词：健康数据、医疗看板、态势感知、医生投入、疾病对比、趋势分析、Excel 分析、数据看板、可视化报表、健康管理、门诊记录、健康咨询、风险人群、随访记录

**不适用场景：**
- 数据在数据库中而非 Excel 文件
- 非健康/医疗领域的数据分析
- 只需要简单的文件格式转换不涉及分析

---

## 命令执行灵活度分级（借鉴 sourceCPT）

本 SKILL 中的命令示例按分级灵活度控制——不写死操作方式，AI 按实际环境自适应：

| 命令场景 | 灵活度 | 允许 AI 调整 | 禁红线 |
|---------|-------|-------------|-------|
| 平台检测 | **LLM 自适应** | 先检测 bash/PowerShell/其他，选对应风格 | 不许硬死单一方式 |
| Excel→可读格式 | **半开** | 有 Python 用 pandas，有 Node 用 xlsx，有 PowerShell 用 ZIP 解压，有别的方式也行 | 不许跳过格式转换直接读 .xlsx 二进制 |
| 基础计数聚合 | **半开** | 用什么工具聚合都行 | 不许 AI 自己手工算（必须用工具算保证准确） |
| 聚合数据写盘 | **死命令** | 必须写盘 | 不许靠 AI 上下文维持，必须落盘 |
| checksum 校验 | **死命令** | 必须校验 | 不许跳过校验直接生成看板 |
| AI 语义分析 | **LLM 决策为主** | AI 按实际数据自适应发现洞察 | 不许自创数字，必须引用聚合数据中的精确值 |
| HTML 生成 | **LLM 决策为主** | AI 按视图清单自适应生成 | 必须遵循 references/echarts-config.md 防截断规范 |

**设计哲学**：纯工程操作（写盘/校验）给死命令；环境适配型（Excel 读取/聚合）给半开；分析决策（语义分析/HTML 生成）LLM 自主。

---

## Step 1：Excel 扫描与语义理解

### 1.1 平台检测

AI 必须先检测运行环境，选可用方式：
- 检测 bash：`uname` 或 `echo "$OS"` 输出含 Linux/Darwin/MSYS/MINGW
- 检测 PowerShell：`echo $PSVersionTable` 不报错
- 检测 Python：`python --version` 或 `py --version` 或 `python3 --version`
- 检测 Node.js：`node --version`

按实际可用的工具选操作方式，不许硬死单一方式。

### 1.2 文件扫描

- 扫描用户指定路径下所有 `.xlsx` 和 `.xls` 文件
- 全部自动纳入分析
- 简短告知扫描结果：文件数和名称

### 1.3 逐文件结构探测

对每个文件，读取前 100 行，提取：
- 列名列表
- 每列的前 5 个唯一值示例
- 每列的数据类型

**禁止直接打印全部原始数据**——只需列名 + 数据样例 + 类型信息。

### 1.4 语义理解每列含义

对于每一列，AI 必须通过以下两步判断其含义：

(a) **列名字面分析**：根据列名（中英文）推断可能的业务含义
(b) **数据内容验证**：检查该列的实际数据值是否与 (a) 的推断一致
  - 分类维度特征：有限个唯一值（如"高/中/低"、"男/女"、国家名）
  - 数值度量特征：连续数值范围（如 BMI 指数、血糖值、血压值）
  - 人员维度特征：中文姓名或 ID 编号
  - 时间维度特征：日期/时间格式

AI 对每列完成语义识别后，直接在报告中附上简要列分类说明，无需等待用户确认。

### 1.5 参考语义词典（辅助判断，不写死）

以下为常见健康管理表格中可能出现的列类型，供 AI 在语义分析时作参考。
**硬约束：任何情况下都必须以实际文件中存在的列名+数据值为准，以下仅为提示。**

| 语义类别 | 常见列名模式 | 判断依据 |
|---------|------------|---------|
| 人员-医生 | 接诊医生、咨询医生、随访医生、医生姓名 | 列名含"医生/医师"，值为中文姓名 |
| 人员-患者 | 患者姓名、姓名、病人 | 列名含"患者/姓名/病人"，值为中文姓名 |
| 地理 | 所属区域、所属国家、机构名称、医务室名称、项目名称 | 列名含"区域/国家/机构"，值为地名 |
| 时间 | 接诊时间、咨询时间、随访日期、监测时间 | 列名含"时间/日期"，值为日期格式 |
| 分级 | 风险等级、接诊类型、咨询类型、人员类别、标签 | 值通常<10个唯一值 |
| 分类 | 初步诊断、咨询内容、风险因素、主诉、治疗建议、随访结果 | 值为文本描述 |
| 数值-体征 | BMI指数、体温、血压、脉搏、心率、血氧 | 值在生理范围内 |
| 数值-生化 | 血糖、胆固醇、甘油三酯、尿酸、肌酐、转氨酶 | 值在临床范围内 |
| 数值-血液学 | 红细胞、白细胞、血小板、血红蛋白 | 值在临床范围内 |
| 数值-尿检 | 尿红细胞、尿糖、尿白细胞、尿蛋白 | 值在临床范围内 |
| 其他 | 病历号、手机号码、性别、年龄、是否为补录 | 按值特征判断 |

### 1.6 常见完整表结构参考（仅供参考）

> 以下为该系统历史上出现过的 4 类表格完整列名。仅作 AI 理解上下文之用。
> **硬约束：必须以文件实际列名为准，不得因为参考表中没有而忽略实际存在的列。**

**A. 门诊记录：** 接诊时间、接诊医生、病历号、患者姓名、性别、年龄、手机号码、接诊类型、既往病史、身高、体重、BMI指数、体温、收缩压、舒张压、脉搏、呼吸频次、血氧饱和度、尿酸、空腹血糖、餐后2小时血糖、总胆固醇、低密度脂蛋白胆固醇、高密度脂蛋白胆固醇、甘油三酯、红细胞计数、白细胞计数、血小板计数、血红蛋白、尿红细胞计数、尿糖、尿白细胞计数、尿蛋白、心电诊断结果、主诉、初步诊断、治疗建议、是否为补录、所属区域、所属国家

**B. 健康咨询：** 咨询时间、咨询医生、病历号、患者姓名、性别、年龄、手机号、咨询内容、咨询类型、咨询方式、咨询建议、所属区域、所属国家

**C. 风险人群：** 机构名称、项目名称、医务室名称、医生姓名、病历号、患者姓名、性别、年龄、手机号、人员类别、风险等级、风险因素、设为风险人员日期、标签、体温、心率、收缩压、舒张压、血氧、呼吸频率、空腹血糖、餐后两小时血糖、糖化血红蛋白、总胆固醇、低密度脂蛋白胆固醇、高密度脂蛋白胆固醇、甘油三酯、丙氨酸氨基转移酶、天门冬氨酸氨基转移酶、碱性磷酸酶、谷氨酰胺转移酶、血尿素氮、血肌酐、血尿酸、最近一次监测时间

**D. 随访记录：** 病历号、患者姓名、性别、年龄、手机号码、人员类别、最近就诊日期、风险人群、随访方式、随访结果、随访日期、随访医生、医务室名称、所属区域、所属国家

---

## Step 2：数据提取与聚合

### 2.1 Excel 转可读格式（半开，AI 自适应）

AI 按实际环境选可用方式把 Excel 转成可读格式（CSV/JSON/文本）：
- 有 Python：用 pandas 读 Excel
- 有 Node.js：用 xlsx 库读 Excel
- 有 PowerShell：解压 .xlsx ZIP + 解析 sheet XML
- 其他方式：AI 按实际环境选

**禁红线**：不许跳过格式转换直接读 .xlsx 二进制。

### 2.2 基础计数聚合（半开，AI 自适应）

AI 按实际环境选可用方式做基础计数聚合：
- 各维度 groupby 计数（医生工作量、地区分布、月度趋势、风险等级、诊断分布）
- 数值统计（均值、中位数、标准差、异常率）
- Top5 排名
- 交叉矩阵

**禁红线**：不许 AI 自己手工算，必须用工具算保证准确。

### 2.3 聚合数据结构

聚合数据必须符合 `references/aggregate-json-schema.md` 定义的结构，包含：
- `meta`：文件名、总记录数、时间范围、区域/医生列表
- `quality`：输入行数、过滤行数、checksum 校验结果
- `aggregations`：各维度聚合数据
- `summary`：机械统计摘要（Top5、分布、趋势方向等），AI 做语义分析时的数据源

### 2.4 聚合数据写盘（死命令）

聚合数据必须写入磁盘文件（如 `aggregate.json`），不许靠 AI 上下文维持。

### 2.5 校验断言（死命令）

聚合数据写盘后，AI 生成看板前必须验证以下断言：

| 编号 | 断言 | 说明 |
|------|------|------|
| V1 | `quality.checksum.match === true` | 分组求和 = 总记录 - 过滤行 |
| V2 | `sum(risk_level_dist.pct) ≈ 100`（±0.5 容差） | 百分比闭合 |
| V3 | `sum(monthly_trend.count) === total_records - filtered_rows` | 记录数闭合 |
| V4 | 每个 `numeric_stats.*.abnormal_rate` 在 [0, 100] 区间 | 异常率合理 |
| V5 | `top3_doctors.values[0] === doctor_workload[0].records` | Top1 一致 |

**校验失败 → AI 报错并重新聚合，不强行生成错误看板。这是数据准确性红线。**

---

## Step 3：动态看板决策（用户确认）

### 3.1 候选视图清单

AI 基于列识别结果，列出候选视图清单向用户确认：

```
识别到 4 个文件、15 个维度列
建议生成以下视图（可多选/取消）：
☑ 1. 医生投入排行（柱状图）
☑ 2. 地区分布对比（横向条形图）
☑ 3. 月度工作量趋势（折线图）
☑ 4. 风险等级构成（饼图）
☑ 5. 关键指标异常率（柱状图+参考线）
☐ 6. 疾病诊断交叉分析（热力图）
☐ 7. 医生×地区投入矩阵（热力图）

全选推荐 / 自定义选择？
```

### 3.2 视图规则

- 至少 3 个视图，不超过 8 个
- 用户确认后才执行后续步骤
- 如果数据不支持某类分析，说明原因并跳过

### 3.3 看板决策维度

- **人员维度**（如有医生/人员姓名列）：医生投入、排名对比
- **地理维度**（如有区域/国家/机构列）：地区分布、地区指标对比
- **分类维度**（如有疾病/诊断/风险等级列）：构成和趋势、交叉分析
- **时间维度**（如有时间列）：月度趋势、增长率、异常波动
- **数值度量**（如有体征/生化指标列）：统计描述、异常率、时间趋势

---

## Step 4：AI 语义分析与生成 HTML 看板

### 4.1 AI 语义分析（LLM 决策为主）

AI 读聚合数据后，用自己的语义理解能力做分析：

- **发现跨维度规律**：如"北京朝阳区高风险占比异常高，可能与该地区饮食结构有关"
- **识别需优先干预的异常指标**：如"收缩压异常率20%居首，血压管理需优先关注"
- **判断趋势是否逆转**：如"前3个月趋势上升，后3个月趋于平缓"
- **生成干预建议**：分 P0（紧急）/P1（重要）/P2（建议）三级
- **根据实际数据自适应**：不限于固定分析模板，按数据实际情况发现值得关注的模式

**禁红线**：不许自创数字，必须引用聚合数据中的精确值。

### 4.2 生成 HTML 看板（LLM 决策为主）

AI 用 Write 工具直接写 HTML 文件，包含：

- ECharts CDN 引入
- 防截断配置（**必须 Read references/echarts-config.md 并严格遵循**）
- KPI 卡片行
- 筛选器（区域/医生下拉 + 重置按钮）
- 图表卡片（每个含 ECharts 图表 + AI 写的数据洞察文字）
- 态势感知分析区块（AI 语义分析的产物：关键发现 + 问题分析 + 干预建议）
- 数据质量说明区块
- 嵌入完整聚合数据为 `<script type="application/json" id="dashboardData">{...}</script>`

### 4.3 数据绑定

- HTML 中嵌入完整聚合数据
- ECharts 图表 JS 从嵌入的数据读取
- HTML 表格通过 JS 从同一数据填充
- **禁止 AI 在 HTML 中硬编码任何数字**——所有数字必须来自嵌入的聚合数据

### 4.4 筛选器

- 筛选器 onChange 时 JS 从嵌入的数据中取子集，调用 `chart.setOption(newOption)` 刷新
- 纯前端，离线可用

### 4.5 HTML 看板布局

```
┌─────────────────────────────────────────────┐
│              标题栏（深色渐变背景）             │
├─────────────────────────────────────────────┤
│  筛选器栏: [区域▼] [医生▼] [重置]              │
├─────────────────────────────────────────────┤
│  [总接诊] [风险人数] [咨询量] [随访完成率]      │  KPI 卡片
├──────────────────┬──────────────────────────┤
│  图表1 + 数据洞察  │  图表2 + 数据洞察          │
├──────────────────┼──────────────────────────┤
│  图表3 + 数据洞察  │  图表4 + 数据洞察          │
├──────────────────┼──────────────────────────┤
│  图表5 + 数据洞察                             │
├─────────────────────────────────────────────┤
│  态势感知分析                                 │
│  ├─ 关键发现 (3-5条，含精确数据)              │
│  ├─ 问题分析 (异常指标/趋势逆转/区域差异)     │
│  └─ 干预建议 (P0/P1/P2 分级)                 │
├─────────────────────────────────────────────┤
│  数据质量说明 (过滤行数/原因/校验结果)         │
└─────────────────────────────────────────────┘
```

---

## Step 5：生成结果 Excel

AI 按实际环境选可用方式写结果 Excel（与 HTML 同目录、同名 `.xlsx`）。

**Sheet 结构详见 references/result-excel-structure.md。**

生成规则：
- 每个聚合表一个 sheet
- Sheet 数量随用户选的视图动态增减
- 数值与 HTML 中嵌入的数据、MD 报告完全同源
- **禁止硬编码数字**

---

## Step 6：生成 Markdown 报告

AI 用 Write 工具直接写 `.md` 文件，与 HTML 同目录，内容包括：

- **数据概览**：总记录数、时间范围、区域数、医生数（引用聚合数据中的精确值）
- **看板分析**：按视图逐一输出关键数据表格 + 文字解读（与 HTML 中文字一致）
- **态势总结**：3-5 条关键发现和建议（与 HTML 中态势分析一致，AI 语义分析的产物）
- **数据说明**：数据来源文件、分析时间、过滤行数、校验结果

---

## 数据准确性红线

### 四道防线

**防线 1：聚合层零误差**
- 所有统计量用工具算（pandas groupby / PowerShell Measure-Object / Node reduce / 其他）
- AI 不做任何算术运算，只读取引用精确值

**防线 2：HTML/Excel 数据绑定**
- HTML 文件内嵌完整聚合数据
- ECharts 图表 JS 从嵌入的数据读取
- HTML 表格通过 JS 从同一数据填充
- 结果 Excel 每个 sheet 对应一个聚合表，数据从同一聚合数据写入
- **禁止 AI 在 HTML 或 Excel 中硬编码任何数字**

**防线 3：交叉校验断言**
- 见 Step 2.5 节，5 条断言
- 校验失败中止生成

**防线 4：文字解读规范**
- AI 写文字解读时必须引用聚合数据中的精确值
- 禁止模糊表述（如"张医生是投入最多的"）
- 禁止与图表矛盾的表述

---

## 抗会话压缩

### 数据落地文件

聚合数据写磁盘，AI 上下文中不持有全量原始数据。

### AI 读聚合数据做语义分析

AI 从磁盘读取聚合数据，用自己的语义理解能力做分析。不靠会话记忆维持数据。

### 三层兜底（借鉴 sourceCPT SRC_ACCESS §5）

| 层 | 机制 | 实现 |
|---|------|------|
| 1 | 聚合数据立即写盘 | 聚合完成后立即落盘，不靠 AI 上下文维持 |
| 2 | AI 分析时读磁盘 | AI 读聚合数据做语义分析时从磁盘读取，不依赖会话记忆 |
| 3 | checksum 最终校验 | sum(各分组) === total_records，不通过则报错重跑 |

---

## 大数据量处理（借鉴 sourceCPT 智能派发）

### 设计哲学

借鉴 sourceCPT 的两个核心哲学：
- **"内存不可信，磁盘可信"** — 所有中间结果写盘，不靠 AI 上下文维持
- **智能派发** — 按数据规模自动选策略，不搞一刀切

### 分批处理策略

AI 聚合前先统计总行数，按规模自动选策略：

| 总行数 | 策略 | 说明 |
|--------|------|------|
| ≤10万行 | `single-batch` | 全量一次聚合 |
| 10万-100万行 | `file-batched` | 按文件分批，每批聚合后写 partial_N.json，最后合并 |
| >100万行 | `chunk-batched` | 每个文件内部也分块读取，流式聚合 |

### 防会话压缩三层兜底

| 层 | 机制 | 实现 |
|---|------|------|
| 1 | 每批 partial 聚合立即写盘 | partial_N.json 每批完成即落盘 |
| 2 | 合并阶段读磁盘 | 合并所有 partial 时从磁盘读取，不依赖 AI 记忆 |
| 3 | checksum 最终校验 | sum(各 partial 的 total_records) === meta.total_records |

### LLM 不碰原始数据

- AI 永远不接触原始 Excel 行数据
- AI 只读聚合结果做语义分析
- 即使几百个文件，AI 上下文负担不变（只读聚合结果）

### 大文件预警

| 单文件大小 | 处理 |
|-----------|------|
| <20MB | 正常处理 |
| 20-100MB | 提示"数据量较大，处理需数秒" |
| >100MB | 用 chunk-batched 策略，分块读取 |

---

## 异常处理

| 异常情况 | 处理方式 |
|---------|---------|
| Excel 文件无法打开 | 尝试不同引擎和编码（utf-8、gbk、utf-8-sig、latin1） |
| 列名无法理解 | 列出该列的前 5 个唯一值，向用户询问含义 |
| 全部分类列（无数值度量） | 以"频次"（记录数）作为度量值 |
| 无时间列 | 跳过趋势分析，仅输出静态分布和对比 |
| 全部时间列为同一时间 | 降级为静态分析，说明无法计算趋势 |
| 只有单一维度可分析 | 输出统计汇总视图（频次分布表+饼图） |
| 数值列含无效值 | 过滤 NaN、Inf、明显异常值，并在报告中说明过滤了多少行 |
| 校验断言失败 | 报错并重新聚合，不生成错误看板 |
| 工具全不可用 | 提示用户安装 Python 或 Node.js，或确认 PowerShell 可用 |
| 文件编码导致中文乱码 | 依次尝试 utf-8-sig → gbk → gb2312 → latin1 |

---

## 验证检查清单

看板生成后，AI 自检：

- [ ] 是否确实从实际文件读取了列名和数据样例？（不能凭空猜测）
- [ ] 校验断言 V1-V5 是否全部通过？
- [ ] HTML 中图表数值 = 表格数值 = 文字解读数值 = Excel数值？（四者同源）
- [ ] 柱状图标签是否完整可见（不被截断）？
- [ ] 态势感知分析 TopN 是否与图表 TopN 一致？
- [ ] 态势感知分析是否由 AI 语义分析生成（非预计算），覆盖医生对比/地区差异/指标预警/趋势变化/交叉发现/干预建议？
- [ ] HTML 是否能独立打开（不依赖服务器）？
- [ ] 结果 Excel 是否生成且 sheet 结构完整？
- [ ] Markdown 报告是否包含数据概览、态势总结、数据质量说明？
- [ ] 异常数据（过滤的行）是否在报告和 HTML 底部说明？
- [ ] 全部内容是否为中文？
```

- [ ] **步骤 2：Commit**

```bash
cd /root && git add .config/opencode/skills/health-dashboard/SKILL.md
git commit -m "feat(health-dashboard): 重写SKILL.md v2——全中文+AI语义分析+灵活度分级+无脚本程序"
```

---

## 自检结果

### 规格覆盖度

| 规格章节 | 实现任务 | 状态 |
|---------|---------|------|
| 2. 目录结构 | 任务 1（清理+创建目录） | ✓ |
| 3.1 SKILL.md | 任务 5 | ✓ |
| 3.2 echarts-config.md | 任务 2 | ✓ |
| 3.3 aggregate-json-schema.md | 任务 3 | ✓ |
| 3.4 result-excel-structure.md | 任务 4 | ✓ |
| 2. 分工（AI 语义分析） | 任务 5 Step 4.1 | ✓ |
| 2. 命令执行灵活度分级 | 任务 5 命令执行灵活度分级章节 | ✓ |
| 4. 验收标准 AC1-AC10 | 任务 5 验证检查清单 | ✓ |
| 6. 大数据处理 | 任务 5 大数据量处理章节 | ✓ |
| 6.3 防会话压缩三层兜底 | 任务 5 抗会话压缩章节 | ✓ |

无遗漏。

### 占位符扫描

无 TODO/待定。所有步骤包含完整文件内容。

### 类型一致性

- 聚合数据结构在任务 3（schema 定义）和任务 5（SKILL.md 引用）中一致
- references 文件名在任务 1（目录结构）和任务 5（SKILL.md 引用）中一致
- ECharts 配置在任务 2（echarts-config.md）和任务 5（SKILL.md 引用）中一致
