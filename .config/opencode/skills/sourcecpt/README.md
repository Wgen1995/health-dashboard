# SourceCPT

> 白盒源码安全审计技能套件 — 基于 LLM 语义驱动的全流程安全评估框架

通过本地源码直读, 对 Java/Go/Python/JS/TS 多语言项目执行 **工程地图 → 入口面 → 鉴权面 → 漏洞匹配 → 交叉关联 → 模式匹配 → LLM 推理 → 漏洞链 → 挑战者对抗 → POC 生成 → 报告交付 → 模式进化闭环** 全流程白盒安全审计。所有逻辑由 LLM 语义驱动, 产出物是 Markdown 报告 + JSON 知识图谱 + Excel 跟踪表 + SARIF。

支持 300w+ 行代码规模（增量审计 + 智能派发 + 模式库自我进化）。

---

## 当前实现状态

| 子项目 | 实现范围 | 状态 |
|--------|---------|------|
| **A 基础设施+Phase 0** | 8 共享规范 + Pipeline 入口 + source-map SKILL + entry-types/sink-types 知识库 | ✅ 完成 |
| B 侦察 Phase 1a/1b | entry-recon + authz-map | 待实现 |
| C Phase 2a 漏洞匹配 | vuln-rules + vuln-rules/ 知识库 | 待实现 |
| D Phase 3+4 推理引擎 | cross-ref + attack-pattern + attack-reasoning + 3 库 | 待实现 |
| E Phase 5+6 链与挑战 | chain-builder + Sibling-Scan + chain-verify + 挑战者对抗 | 待实现 |
| F Phase 7+8+9 交付进化 | poc-generator + 3 报告 + pattern-evolve + patch-patterns | 待实现 |
| G 可选合规插件 | compliance-audit + compliance-report + compliance-rules/ | 待实现 |
| H 可选持续审计+Phase 1c | baseline-diff + product-context + cron 调度模板 | 待实现 |

> 子项目 A 跑通后可对任意本地源码项目执行 `$sourcecpt` 命令产出 Phase 0 的 manifest/入口/sink 索引, 但漏洞判定需 B-F 完整。

---

## 目录结构

```
~/.config/opencode/skills/sourcecpt/
├── SKILL.md                              # Pipeline 入口: 参数收集 + 环境验证 + 调度各 Phase
├── skills/
│   ├── shared/                           # 8 个共享规范（被各 Phase 引用）
│   │   ├── SRC_ACCESS.md                  # 文件读取/分块/按需加载/临时方法级图谱规范
│   │   ├── OUTPUT_STANDARD.md             # 输出格式/五态/图谱 JSON schema/命名规范
│   │   ├── OFFSET_RATING.md               # 审计深度 T0-T3 分层定义
│   │   ├── CHAIN_STANDARD.md              # 链节点格式/可达性判定规则
│   │   ├── LOOP_POLICY.md                 # 局部 loop 策略/硬停止/Token 预算
│   │   ├── TRACKING_SHEET.md              # Excel 跟踪表字段定义 + 类型三列映射表
│   │   ├── BASELINE_DIFF.md               # 基线对比协议(持续审计)
│   │   └── ANTI_HALLUCINATION.md          # 反幻觉 56 条全局约束
│   └── source-map/
│       └── SKILL.md                       # Phase 0 工程地图（A 已实现）
├── knowledge-bases/
│   ├── entry-types.md                     # 13 入口面 grep 模式定义
│   └── sink-types.md                      # 28 sink 类 grep 模式定义
└── README.md                              # 本文件
```

---

## 前提条件

- opencode（或兼容的 AI coding agent, 支持 Skill 加载 + Task(general)）
- 本地源码项目路径（必填）
- 系统 shell（**跨平台支持**）——Linux/macOS/WSL/Git Bash 的 bash 或 Windows 的 PowerShell 均可。工具调用时 LLM 自动检测平台选对应命令（详见 `skills/shared/SRC_ACCESS.md` §1.2）
- 可选: python3（Phase 8d 产 xlsx 时调用, 无则回退 csv）
- 可选: git（mode=increment 时调用 git diff, 跨平台原生支持）

---

## 安装

将整个 `sourcecpt` 目录放置在 opencode 可识别的 skills 目录:
```
~/.config/opencode/skills/sourcecpt/
```

每个 Skill 以自己的 `SKILL.md` 为加载入口, `skills/{phase}/SKILL.md` 按需加载, `skills/shared/*.md` 被 Phase 子技能引用（按需读不全量载, 见 SRC_ACCESS §3 按需加载原则）。

---

## 触发方式

在 opencode 对话中触发（自然语言或显式 skill 名）:

```
使用 $sourcecpt 审计 /path/to/project
使用 $sourcecpt 对 /path/to/repo 做 fast 模式审计, scope=java
使用 $sourcecpt 增量审计 /path/to/repo, changed-since=main
使用 $source-map /path/to/project                    # 仅跑 Phase 0 工程地图
```

---

## 参数说明

| 参数 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `project-path` | string | 是 | — | 本地源码根目录绝对路径 |
| `mode` | enum | 否 | `full` | `fast` / `full` / `increment` |
| `scope` | enum | 否 | `all` | 语言 / `module=path` / `changed` / `owasp-a1..a10` |
| `changed-since` | string | 否 | — | increment 模式 git ref |
| `context-path` | string | 否 | — | 启用 Phase 1c 产品上下文文档目录 |
| `--with-compliance` | flag | 否 | false | 启用 Phase 2b + 8c 合规 |
| `chain-depth` | int | 否 | 5 | 漏洞链最大深度（硬上限 8） |
| `baseline` | string | 否 | — | 上次基线目录路径（增量审计对比） |

---

## 设计文档

完整设计规格与设计哲学:
- [设计文档主规格](../../../docs/superpowers/specs/2026-06-28-sourcecpt-design.md) — 9 章完整规格说明
- [设计哲学速查](../../../docs/superpowers/specs/2026-06-28-sourcecpt-design-philosophy.md) — 8 类 32 项机制资产表, 11 节速查导航
- [实现计划 子项目 A](../../../docs/superpowers/plans/2026-06-28-sourcecpt-A-infra-and-phase0.md)

---

## 安全边界

本项目仅用于授权代码审计、企业内部安全评估、学习和研究:
- 报告中的 POC 请求包应使用授权测试环境和占位符凭据（`{{test_token}}`）, 不应包含真实敏感数据或生产环境凭据
- 报告不应包含批量利用脚本、持久化 payload、横向移动步骤或破坏性操作
- 生成 POC 时不加门禁（白盒 POC 是代码路径触发说明 + 模板化 Burp HTTP 包, 非可执行攻击脚本）
- 不许对生产代码做破坏性修改

详见设计哲学 §10 借鉴与原创标注 + TRACKING_SHEET §3 反幻觉引用。

---

## 许可证

Private — 内部使用, 未经授权禁止分发。

---

## 相关链接

- [OWASP Top 10](https://owasp.org/Top10/)
- [CWE - Common Weakness Enumeration](https://cwe.mitre.org/)
- [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
- [SARIF Specification](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html)
- [opencode](https://opencode.ai)