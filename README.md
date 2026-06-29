# Health Dashboard Skill

OpenCode 技能：健康管理态势感知看板

## 简介

自动分析员工健康管理 Excel 数据，生成多维度态势感知交互式 HTML 看板 + Markdown 分析报告。

AI 通过 LLM 语义理解自主识别表格列含义，动态决策分析维度，无需预设数据结构。

## 适用数据

支持门诊记录、健康咨询、风险人群、随访记录等多种 Excel 表格，不预设列结构——列名和数据可能变化，AI 自适应语义分析。

## 安装

将 `SKILL.md` 放到 opencode skills 目录：

```bash
mkdir -p ~/.config/opencode/skills/health-dashboard
cp SKILL.md ~/.config/opencode/skills/health-dashboard/
```

## 使用方式

向 opencode 发送消息触发，例如：

- "帮我分析这些健康数据"
- "生成医疗态势感知看板"
- "对 /path/to/data 下的 Excel 做趋势分析"

AI 会自动匹配 skill 并按以下流程执行：

```
扫描文件 → 语义理解列含义 → 动态决策看板方案 → 用户确认 → 生成 HTML + Markdown
```

## 文件结构

```
~/.config/opencode/skills/health-dashboard/
└── SKILL.md                 # Skill 指令文件
```
