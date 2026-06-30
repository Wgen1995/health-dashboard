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
