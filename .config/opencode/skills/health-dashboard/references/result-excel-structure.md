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
