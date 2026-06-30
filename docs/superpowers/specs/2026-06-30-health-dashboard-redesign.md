# 健康管理态势感知看板 Skill 改进设计

> **日期**: 2026-06-30
> **状态**: 已批准，待编写实现计划
> **目标文件**: `/root/.config/opencode/skills/health-dashboard/SKILL.md` + 新增 `reference/` 目录

---

## 1. 问题陈述

现有 `health-dashboard` skill 存在四类问题：

| 编号 | 问题 | 根因 |
|------|------|------|
| P1 | 柱状图数据标签被截断，柱子过长时标签只显示一半 | 旧版用 Plotly，未规定 margin/boundaryGap 等布局约束 |
| P2 | 态势感知分析文字与图表数据不一致（如文字说 A 是 Top1，图表显示 B 是 Top1） | AI 独立于 Python 脚本写文字解读，两条计算路径不同步 |
| P3 | HTML 表格数值与图表数值不一致 | 表格和图表分别生成，无统一数据源 |
| P4 | 大数据量（万~十万级）日积月累后数据丢失或分析不准 | 仅有"全量聚合"泛泛描述，无校验机制；且 LLM 会话压缩导致聚合 JSON 截断 |

用户环境约束：
- Windows，无 Python，非技术用户
- 不希望看到程序代码文件
- 数据准确性是红线，数据不丢失是红线，质量第一

---

## 2. 解决方案概述

**方案 A（已批准）**：多引擎自动探测 → 临时脚本聚合 → JSON 落地文件 → 临时脚本直接生成 HTML 骨架 → AI 只补充文字解读。

核心设计原则：
1. **聚合 JSON 是唯一真相源** — 图表、表格、文字三者都从它派生
2. **AI 不做算术** — 所有统计量由提取引擎计算，AI 只读取引用
3. **数据落地文件** — 聚合 JSON 写磁盘，AI 上下文中只保留 <2KB 摘要，抗会话压缩
4. **脚本生成 HTML 骨架** — 图表 ECharts 配置由脚本从 JSON 自动生成，不经 AI 上下文
5. **校验断言** — 提取层输出 checksum，AI 验证通过后才继续
6. **ECharts 替代 Plotly** — 自动标签避让，防截断
7. **启动时用户确认视图** — 非全自动，先问后做

---

## 3. 详细设计

### 3.1 整体架构

```
用户提供 Excel 路径
    │
    ▼ AI 自动探测环境（Python → Node.js → PowerShell）
    │
    ▼ AI 扫描文件 + 语义理解列含义（自动）
    │
    ▼ AI 列出候选视图清单 → 用户多选确认
    │
    ▼ AI 生成临时脚本（引擎模板），脚本职责：
    │   ├─ 读 Excel 全量
    │   ├─ 聚合 → 写 aggregate.json 到磁盘
    │   ├─ 校验 checksum → 写校验结果到 json
    │   ├─ 读 aggregate.json → 生成完整 HTML 骨架
    │   │   （含 ECharts 图表 + 表格 + KPI + 筛选器 + DATA 嵌入）
    │   └─ 读 aggregate.json → 生成结果 Excel 文件
    │       （每个聚合表一个 sheet，数据与 JSON 同源）
    │
    ▼ AI 读 aggregate.json 的 meta + top3 摘要（<2KB）
    │
    ▼ AI 用 Edit 工具向 HTML 骨架注入文字解读和态势总结
    │
    ▼ AI 生成 Markdown 报告（引用同一份摘要）
    │
    ▼ AI 删除临时脚本，保留 HTML + JSON + MD + Excel
    │
    ▼ 输出结果路径给用户
```

### 3.2 数据提取层

#### 3.2.1 多引擎自动探测

AI 按优先级依次探测可用引擎：

| 优先级 | 引擎 | 探测命令 | Excel 读取方式 |
|--------|------|---------|---------------|
| 1 | Python | `python --version` 或 `py --version` | pandas + openpyxl |
| 2 | Node.js | `node --version` | xlsx 库（npx 自动安装） |
| 3 | PowerShell | Windows 自带 | 解压 .xlsx ZIP + 解析 sheet XML |

探测到第一个可用引擎即停止，使用该引擎生成临时脚本。

#### 3.2.2 临时脚本执行方式

- AI 将临时脚本写入系统临时目录（Windows `%TEMP%` / Linux `/tmp`）
- 文件名带随机后缀避免冲突（如 `hd_extract_abc123.py`）
- 脚本执行成功且 HTML 骨架生成完毕后，AI 自动删除临时脚本文件
- 如脚本执行失败，保留临时脚本供排查，AI 向用户报告错误路径
- 用户全程不可见代码文件（成功场景下）

#### 3.2.3 聚合 JSON 结构

提取层输出的唯一数据源，写入磁盘 `aggregate.json`：

```json
{
  "meta": {
    "files": ["门诊记录.xlsx", "健康咨询.xlsx"],
    "file_types": {"门诊记录.xlsx": "outpatient", "健康咨询.xlsx": "consultation"},
    "total_records": 570,
    "date_range": {"start": "2025-01-01", "end": "2025-06-28"},
    "regions": ["北京朝阳区", "上海浦东新区"],
    "doctors": ["张伟明", "李芳华"],
    "columns_identified": {
      "门诊记录.xlsx": {"接诊时间": "time", "接诊医生": "doctor", "BMI指数": "numeric_vital"}
    }
  },
  "quality": {
    "total_in": 570,
    "total_after_filter": 568,
    "filtered_rows": 2,
    "filter_reasons": ["BMI>100 过滤1行", "血压负值过滤1行"],
    "nulls_per_column": {"BMI指数": 3, "血压": 0},
    "checksum": {
      "sum_groups": 568,
      "match": true
    }
  },
  "aggregations": {
    "doctor_workload": [
      {"name": "张伟明", "records": 45, "patients": 38, "regions": 3},
      {"name": "李芳华", "records": 38, "patients": 32, "regions": 2}
    ],
    "region_distribution": [
      {"region": "北京朝阳区", "records": 95, "patients": 80, "doctors": 5}
    ],
    "monthly_trend": [
      {"month": "2025-01", "count": 50, "patients": 45}
    ],
    "risk_level_dist": [
      {"level": "低风险", "count": 30, "pct": 25.0}
    ],
    "diagnosis_dist": [
      {"diagnosis": "高血压", "count": 30, "pct": 15.0}
    ],
    "numeric_stats": {
      "BMI指数": {"mean": 24.5, "median": 24.0, "std": 3.2, "abnormal_rate": 15.0, "abnormal_count": 30},
      "收缩压": {"mean": 125.0, "median": 124.0, "std": 15.0, "abnormal_rate": 20.0, "abnormal_count": 40}
    },
    "top3_doctors": {"names": ["张伟明","李芳华","王建国"], "values": [45, 38, 35]},
    "top3_diagnoses": {"names": ["高血压","糖尿病","高脂血症"], "values": [30, 25, 20]},
    "doctor_region_matrix": {
      "doctors": ["张伟明", "李芳华"],
      "regions": ["北京朝阳区", "上海浦东新区"],
      "matrix": [[20, 15], [10, 20]]
    },
    "risk_trend": [
      {"month": "2025-01", "低风险": 10, "中风险": 5, "高风险": 3}
    ]
  },
  "summary": {
    "total_records": 570,
    "total_patients": 420,
    "total_doctors": 7,
    "total_regions": 6,
    "avg_bmi": 24.5,
    "abnormal_rate_bmi": 15.0,
    "abnormal_rate_bp": 20.0,
    "top_doctor": "张伟明",
    "top_doctor_records": 45,
    "top_diagnosis": "高血压",
    "top_diagnosis_count": 30,
    "trend_direction": "上升",
    "trend_pct_change": 12.5
  }
}
```

`summary` 字段是 AI 写文字解读时引用的摘要，体积 <2KB，单独存在以防会话压缩。

#### 3.2.4 校验断言

提取层输出 JSON 后，AI 写 HTML 前必须验证：

| 编号 | 断言 | 说明 |
|------|------|------|
| V1 | `quality.checksum.match === true` | 分组求和 = 总记录 - 过滤行 |
| V2 | `sum(risk_level_dist.pct) ≈ 100`（±0.5 容差） | 百分比闭合 |
| V3 | `sum(monthly_trend.count) === meta.total_records - quality.filtered_rows` | 时间趋势记录数闭合 |
| V4 | 每个 `numeric_stats.*.abnormal_rate` 在 [0, 100] 区间 | 异常率合理 |
| V5 | `top3_doctors.values[0] === doctor_workload[0].records` | Top1 一致 |

校验失败 → AI 报错并重新运行提取，不强行生成错误看板。

### 3.3 图表渲染层（ECharts 防截断）

#### 3.3.1 ECharts CDN

```html
<script src="https://cdn.jsdelivr.net/npm/echarts@5/dist/echarts.min.js"></script>
```

如用户要求离线，AI 下载 echarts.min.js 内嵌到 HTML。

#### 3.3.2 防截断配置规范

所有图表必须应用的配置（写入 SKILL.md 作为硬约束）：

```javascript
{
  grid: {
    containLabel: true,          // 网格自动包含轴标签
    left: "10%",
    right: "10%",
    top: "15%",
    bottom: "10%"
  },
  xAxis: {
    axisLabel: {
      interval: 0,               // 强制显示所有标签
      rotate: 30,                // 倾斜防重叠
      width: 80,                 // 限制宽度
      overflow: "truncate",      // 超长截断
      formatter: (v) => v.length > 8 ? v.slice(0,8)+"..." : v
    }
  },
  yAxis: {
    boundaryGap: ["0", "20%"]    // 顶部留20%空间给柱顶标签
  },
  series: [{
    label: {
      show: true,
      position: "top",
      formatter: "{c}",          // 显示数值
      fontSize: 11
    }
  }]
}
```

#### 3.3.3 图表类型 → 配置映射

| 图表类型 | 防截断关键配置 | 用途 |
|---------|--------------|------|
| 柱状图 | `yAxis.boundaryGap["0","20%"]` + `label.position:"top"` | 医生投入排行 |
| 横向条形图 | X/Y 轴反转 + `grid.right:"18%"` 留右侧给长标签 | 地区分布 |
| 饼图 | `label.formatter:"{b}: {c} ({d}%)"` + `label.overflow:"truncate"` | 风险等级构成 |
| 折线图 | `symbolSize:8` + `label.show:true` + `markPoint` 标注最大最小 | 月度趋势 |
| 堆叠柱状图 | 同柱状图 + `stack:"total"` | 风险等级月度趋势 |
| 热力图 | `label.show:true` 每格显示数值 + `visualMap` 自动着色 | 医生×地区矩阵 |

#### 3.3.4 容器与响应式

```html
<div id="chart1" style="width:100%;min-height:400px;"></div>
<script>
  window.addEventListener('resize', () => {
    charts.forEach(c => c.resize());
  });
</script>
```

每图最小高度 400px，防止矮图挤压标签。

### 3.4 数据准确性保障（四道防线）

#### 防线 1：聚合层零误差
- 所有统计量在提取引擎内用内置函数算（pandas groupby / PowerShell Measure-Object / Node reduce）
- AI 不做任何算术运算，只读取 JSON 字段值

#### 防线 2：HTML/Excel 数据绑定
- HTML 文件内嵌完整聚合 JSON：`<script>const DATA = {...聚合JSON原文...};</script>`
- ECharts 图表直接引用 `DATA.aggregations.xxx`
- HTML 表格通过 JS 从 `DATA` 填充
- 结果 Excel 文件每个 sheet 对应一个聚合表，数据从同一 JSON 写入
- 禁止 AI 在 HTML 或 Excel 中硬编码任何数字

#### 防线 3：交叉校验断言
- 见 3.2.4 节，5 条断言
- 校验失败中止生成

#### 防线 4：文字解读规范
AI 写文字解读时必须引用 JSON 精确值：
- ✓ "投入最多的医生为张伟明（45人次）" — 45 来自 `summary.top_doctor_records`
- ✗ "张医生是投入最多的" — 模糊
- ✗ "李医生排第一" — 与图表矛盾，禁止

### 3.5 抗会话压缩

#### 3.5.1 数据落地文件

聚合 JSON 写磁盘 `aggregate.json`，AI 上下文中不持有全量 JSON。

#### 3.5.2 脚本直接生成 HTML 骨架

临时脚本职责扩展：读 Excel → 聚合 → 写 JSON → 读 JSON → 生成完整 HTML 骨架（含 ECharts 图表+表格+KPI+筛选器+DATA嵌入）。

HTML 骨架生成不经 AI 上下文，脚本直接从 JSON 文件读取并写入 HTML。

#### 3.5.3 AI 上下文只保留摘要

AI 写文字解读时，只读 `aggregate.json` 中以下字段（合计 <2KB）：

```
- meta（文件名、总记录数、时间范围、区域列表、医生列表）
- summary（关键派生指标：总患者数、Top医生、Top诊断、趋势方向等）
- aggregations.top3_doctors / top3_diagnoses（排名+值）
- aggregations.numeric_stats（各指标均值+异常率，仅名称+均值+异常率三列）
- quality.checksum（校验结果）
```

AI 上下文中不持有 `aggregations` 中的明细数组（doctor_workload 全量、region_distribution 全量、monthly_trend 全量等），这些只存在于磁盘 JSON 和 HTML 嵌入的 DATA 对象中。总计 <2KB，远低于压缩阈值。

#### 3.5.4 AI 用 Edit 注入文字

AI 用 Edit 工具向 HTML 骨架中的占位符注入文字：
- `<!-- INSIGHT:chart1 -->` → 图表1的数据洞察文字
- `<!-- INSIGHT:chart2 -->` → 图表2的数据洞察文字
- `<!-- SITUATION_ANALYSIS -->` → 底部态势感知分析区块

### 3.6 HTML 看板结构

```
┌─────────────────────────────────────────────┐
│              标题栏                           │
│  健康管理态势感知看板 · 2025H1                 │
├─────────────────────────────────────────────┤
│  筛选器栏: [区域▼] [医生▼] [时间范围▼] [重置]  │
├─────────────────────────────────────────────┤
│  [总接诊] [风险人数] [咨询量] [随访完成率]      │  KPI 卡片(随筛选联动)
├──────────────────┬──────────────────────────┤
│  图表1: 医生投入   │  图表2: 地区分布          │
│  <!-- INSIGHT:1 -->│  <!-- INSIGHT:2 -->      │
├──────────────────┼──────────────────────────┤
│  图表3: 月度趋势   │  图表4: 风险等级          │
│  <!-- INSIGHT:3 -->│  <!-- INSIGHT:4 -->      │
├──────────────────┼──────────────────────────┤
│  图表5: 指标异常率                           │
│  <!-- INSIGHT:5 -->                          │
├─────────────────────────────────────────────┤
│  态势感知分析                                 │
│  <!-- SITUATION_ANALYSIS -->                 │
│  ├─ 关键发现 (3-5条，含精确数据)              │
│  ├─ 问题分析 (异常指标/趋势逆转/区域差异)     │
│  └─ 建议 (分优先级 P0/P1/P2)                 │
├─────────────────────────────────────────────┤
│  数据质量说明 (过滤行数/原因/校验结果)         │
└─────────────────────────────────────────────┘
```

#### 筛选器技术方案
- HTML 内嵌完整聚合 JSON（含各维度交叉聚合的明细）
- 筛选器 onChange 时 JS 重新从 `DATA` 中取子集，调用 `chart.setOption(newOption)` 刷新
- 纯前端，离线可用

#### 数据质量说明区块
- 显示 `quality.total_in` / `quality.filtered_rows` / `quality.filter_reasons`
- 显示 `quality.checksum.match` 校验结果
- 让用户可见数据完整性

### 3.6.1 结果 Excel 输出

临时脚本在生成 HTML 骨架的同时，输出一份结果 Excel 文件，与 HTML 同目录、同名 `.xlsx`。

**Sheet 结构**（每个聚合表一个 sheet）：

| Sheet 名 | 内容 | 数据源 |
|----------|------|--------|
| 概览 | 总记录数、时间范围、区域数、医生数、患者数 | `meta` + `summary` |
| 医生投入 | 医生名、记录数、患者数、覆盖地区数 | `aggregations.doctor_workload` |
| 地区分布 | 地区、记录数、患者数、医生数 | `aggregations.region_distribution` |
| 月度趋势 | 月份、记录数、患者数 | `aggregations.monthly_trend` |
| 风险等级 | 等级、人数、占比 | `aggregations.risk_level_dist` |
| 诊断分布 | 诊断、人数、占比 | `aggregations.diagnosis_dist` |
| 指标统计 | 指标名、均值、中位数、标准差、异常率、异常人数 | `aggregations.numeric_stats` |
| 医生×地区矩阵 | 行=医生、列=地区、值=记录数 | `aggregations.doctor_region_matrix` |
| 风险趋势 | 月份×风险等级堆叠 | `aggregations.risk_trend` |
| 数据质量 | 总行数、过滤行数、过滤原因、校验结果 | `quality` |

**生成规则**：
- Sheet 数量随用户选的视图动态增减（用户未选的视图不生成对应 sheet）
- 每个 sheet 首行为列名，后续为数据行
- 数值与 HTML 中 `DATA` 对象、MD 报告中的表格完全同源
- Excel 写出由临时脚本完成（pandas `to_excel` / Node `xlsx.writeFile` / PowerShell `Export-Excel` 或 COM），不经 AI 上下文

### 3.7 交互流程

```
1. 用户提供 Excel 路径
   │
   ▼
2. AI 扫描文件 + 语义理解列含义（自动）
   │
   ▼
3. AI 列出候选视图清单 → 用户多选确认：
   ┌───────────────────────────────────────┐
   │ 识别到 4 个文件、15 个维度列            │
   │ 建议生成以下视图（可多选/取消）：         │
   │ ☑ 1. 医生投入排行（柱状图）              │
   │ ☑ 2. 地区分布对比（横向条形图）          │
   │ ☑ 3. 月度工作量趋势（折线图）            │
   │ ☑ 4. 风险等级构成（饼图）               │
   │ ☑ 5. 关键指标异常率（柱状图+参考线）     │
   │ ☐ 6. 疾病诊断交叉分析（热力图）          │
   │ ☐ 7. 医生×地区投入矩阵（热力图）         │
   │                                        │
   │ 全选推荐 / 自定义选择？                  │
   └───────────────────────────────────────┘
   │
   ▼
4. 用户确认后，AI 全自动执行：
   生成临时脚本→聚合→校验→生成HTML骨架→注入文字→清理
   │
   ▼
5. 输出结果路径
```

### 3.8 大数据量处理（万~十万级）

| 阶段 | 处理 | 数据量控制 |
|------|------|-----------|
| 读取 | 全量读入内存 | 原始数据不输出给 AI |
| 聚合 | groupby/Measure-Object | 只输出聚合结果 |
| 输出 JSON | 只含聚合值 | 控制在 <500 个数字字段 |
| AI 读 JSON | 只读 summary 摘要 | 不接触原始行数据 |
| HTML 嵌入 | 只嵌入聚合 JSON | HTML 文件 <1MB |

不丢失数据的保障：
- 提取层先输出 `meta.total_records`（原始总行数）
- 聚合后输出 `quality.filtered_rows`（被过滤的行数+原因）
- 校验断言 V1：`sum(各分组count) === total_records - filtered_rows`
- 不满足则报错重跑

多文件合并：
- AI 比对各文件列名交集，相同结构自动 concat
- 不同结构各自独立聚合，JSON 中按文件类型分 key 存储

文件大小预警：
- 单文件 >20MB 时 AI 提示"数据量较大，处理需数秒"
- 单文件 >100MB 时提示"建议拆分或提供 CSV 格式"

### 3.9 SKILL.md 改造范围

| 章节 | 内容 | 改动程度 |
|------|------|---------|
| 概述 | 更新流程图 | 重写 |
| 何时使用 | 不变 | 保留 |
| Step 1: Excel扫描与语义理解 | 保留列识别逻辑，增加多引擎探测子步骤 | 小改 |
| Step 2: 数据提取与聚合（新） | 多引擎探测→临时脚本→聚合JSON→校验断言 | 全新 |
| Step 3: 动态看板决策 | 改为向用户展示候选清单+多选确认 | 重写 |
| Step 4: 生成HTML看板（重写） | ECharts配置规范+防截断+筛选器+态势总结+数据绑定规则 | 全新 |
| Step 5: 生成Markdown报告 | 增加数据校验结果说明 | 小改 |
| Step 6: 生成结果Excel（新） | 每聚合表一sheet，数据同源JSON | 全新 |
| 数据准确性红线（新） | 四道防线+校验断言+checksum | 全新 |
| 抗会话压缩（新） | 数据落地文件+脚本生成骨架+AI只补文字 | 全新 |
| 大数据量处理 | 全量读取+聚合输出+不丢失保障 | 重写 |
| 异常处理 | 增加引擎探测失败、校验失败的处理 | 扩充 |
| 验证检查清单 | 增加图表不截断、数据一致性、校验通过等条目 | 扩充 |

### 3.10 新增参考文件

| 文件 | 用途 |
|------|------|
| `reference/echarts-template.html` | ECharts 看板 HTML 骨架模板（含防截断配置、筛选器、占位符） |
| `reference/extract-pandas.py` | pandas 聚合脚本模板（含 Excel 写出） |
| `reference/extract-powershell.ps1` | PowerShell 聚合脚本模板（含 Excel 写出） |
| `reference/extract-node.js` | Node.js 聚合脚本模板（含 Excel 写出） |
| `reference/aggregate-json-schema.json` | 聚合 JSON 结构定义 |
| `reference/result-excel-template.xlsx` | 结果 Excel 文件模板（定义 sheet 结构） |

AI 生成临时脚本时从这些模板复制改写，减少出错。

---

## 4. 验收标准

| 编号 | 验收项 | 验证方法 |
|------|--------|---------|
| AC1 | 柱状图标签不被截断 | 生成的 HTML 在浏览器中打开，柱顶标签完整可见 |
| AC2 | 图表数值=表格数值=文字解读数值=Excel数值 | 四者均引用同一份聚合 JSON |
| AC3 | 态势感知分析 TopN 与图表 TopN 一致 | 文字引用 `summary.top_doctor` 等，图表引用 `DATA.aggregations.top3_doctors` |
| AC4 | 大数据量（十万行）不丢失数据 | `checksum.match === true`，HTML 底部显示校验通过 |
| AC5 | 无 Python 环境也能运行 | Windows 上仅 PowerShell 可用时正常生成 |
| AC6 | 用户不接触代码 | 临时脚本在临时目录运行后删除，用户只看到 HTML+MD+Excel |
| AC7 | 启动时用户可选视图 | AI 列出候选清单，用户确认后才执行 |
| AC8 | 会话压缩不影响数据准确性 | 聚合 JSON 落地文件，HTML 骨架由脚本生成，AI 只读 <2KB 摘要 |

---

## 5. 范围边界

**本次改造包含**：
- 重写 SKILL.md
- 新增 6 个 reference 文件
- 用生成的 4 个 Excel 文件做端到端验证

**本次改造不包含**：
- 不修改 opencode 核心代码
- 不开发独立可执行程序
- 不支持实时数据库数据源（仅 Excel）
- 不支持百万级以上数据（需分块加载，超出当前范围）
