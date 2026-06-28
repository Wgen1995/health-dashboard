---
name: sourcecpt
description: >
  白盒源码安全审计技能套件（opencode SKILL）。基于 LLM 语义驱动, 多语言通用,
  本地源码直读。通过 Phase 顺序 Pipeline 执行: 工程地图→入口面→鉴权面→漏洞匹配→
  交叉关联→模式匹配→LLM 推理→漏洞链→挑战者对抗→POC→报告→模式进化闭环。
  支持 300w+ 行规模(增量审计+智能派发), 攻击模式库自我进化。
  所有逻辑由 LLM 语义处理, 产出物是 Markdown 报告+JSON 知识图谱+Excel 跟踪表+SARIF。
  使用场景: 对本地源码项目做授权白盒安全审计。
  不使用场景: 未授权审计、对生产代码做破坏性修改、批量扫描非自有项目。
---

# SourceCPT — Pipeline 入口 SKILL

本 SKILL 是白盒源码审计套件的**唯一入口**, 负责参数收集、环境验证、工作目录初始化、按顺序调度各 Phase 子技能、展示摘要。**不直接执行任何检测逻辑**, 通过 Task(general) 依次调用各 Phase 子技能 SKILL.md。

---

## 参数格式定义

| 名称 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `project-path` | string | **是** | — | 本地源码根目录绝对路径 |
| `mode` | enum | 否 | `full` | `fast` / `full` / `increment` |
| `scope` | enum | 否 | `all` | `java`/`go`/`python`/`js`/`ts`/`all`(逗号多选) 或 `module=path` 或 `changed` 或 `owasp-a1..a10` |
| `changed-since` | string | 否 | — | increment 模式 git ref（仅 mode=increment 时必填） |
| `context-path` | string | 否 | — | 启用 Phase 1c 产品上下文的文档目录 |
| `--with-compliance` | flag | 否 | false | 启用 Phase 2b 编码合规 + Phase 8c 合规报告 |
| `chain-depth` | int | 否 | 5 | 漏洞链最大深度（硬上限 8） |
| `baseline` | string | 否 | — | 历次基线目录路径, 用于增量审计 diff 对比 |

### mode 参数选项

| 选项 | 说明 | 跳过的 Phase |
|------|------|------------|
| `fast` | 快速摸底出基线报告, 仅规则匹配+漏洞报告 | 跳 Phase 3/4/5/6/7/9 |
| `full` | 完整 9 Phase 全流程审计 | 无跳过 |
| `increment` | 增量审计, 仅审 git diff 变更 + 调用图邻居 | 按变更范围裁切 |

### scope 参数选项

| 选项 | 含义 |
|------|------|
| `java`/`go`/`python`/`js`/`ts` | 单语言（逗号多选如 `java,python`） |
| `all` | 自动取项目文件数 Top3 语言作为 effective_scope |
| `module=path` | 仅审计指定目录（依赖 Phase 0 模块清单） |
| `changed` | 仅审 git diff 触及文件（配合 mode=increment） |
| `owasp-a1..a10` | 仅审计某 OWASP 类（如 `owasp-a01` 仅审注入） |

---

## 第一步: 参数收集

使用 question 工具交互收集以下参数:

1. **project-path**（必填）: 本地源码根目录绝对路径
2. **mode**（默认 `full`）: fast / full / increment
3. **scope**（默认 `all`）: 语言或模块名或 changed
4. **changed-since**（仅 increment 必填）: git ref
5. **context-path**（可选）: 启用 Phase 1c
6. **--with-compliance**（可选, 默认否）: 启用 Phase 2b + 8c
7. **chain-depth**（默认 5, ≤8）: 漏洞链最大深度
8. **baseline**（可选）: 上次审计基线目录

若 project-path 缺失或路径无效，**立即终止**并报告错误。其他参数缺失用默认值，不阻塞。

---

## 第二步: 环境验证

1. 检查 `project-path` 存在且为目录 (Read)
2. Glob 项目根确认至少有源码文件, 否则报错"无可审计源码"
3. 工具可用性检查:
   - bash 可用（必须, 执行 grep/find/wc/sha256sum/git diff）
   - git 可用（mode=increment 必须才能 git diff）
   - python3 可用（仅 Phase 8d 产 xlsx 用, 缺则回退 csv）
4. 记录环境信息到 session_config.json 的 target 部分扩展:
   - project_path / scope / mode / context_path / with_compliance
5. 计算会话 ID（uuid 或时间戳）

若项目路径无效或找不到源码，**立即终止**。

---

## 第三步: 初始化工作目录

在 `/tmp/sourcecpt-{session_id}/` 下创建完整目录树（按 OUTPUT_STANDARD §1）:

```
/tmp/sourcecpt-{session_id}/
├── session_config.json          # 立即写入（含 budget 段）
├── progress.json                # 全 phase 置 pending
├── manifest.json                # Phase 0 写入
├── dispatch_plan.json           # Phase 0 写入
├── audit_log.json               # 空数组初值 []
├── chunk_manifest.json          # Phase 2b 用（可选）
├── sibling_scan_log.json        # Phase 5/6 用
├── evidence/
│   ├── phase0/raw/
│   ├── phase1a/  phase1b/  phase1c/  phase2a/  phase2b/
│   ├── cross-ref/  phase4a/  phase4b/
│   ├── phase5/  phase6/  phase6_loop/  phase7/  phase9/
├── knowledge_graph/
│   ├── nodes/{entries,sinks,modules,authz_configs,constraints,
│   │         compliance_violations,vuln_candidates,
│   │         pattern_matches,chains,verified_vulns}.json
│   └── edges/{calls_module,entry_in_module,entry_calls,
│             entry_covered_by,authz_deviation,data_flow,
│             cross_ref,upgrade,chain_steps,sibling_of,
│             business_flow,constraint_to_code}.json
├── reports/
│   ├── pentest_report.{md,json,sarif}     # Phase 8d
│   ├── compliance_report.{md,json}         # 可选 Phase 8c
│   ├── chain_report.{md,json}              # Phase 8b
│   ├── patch_suggestion.{md,json}          # Phase 8d
│   ├── qa_report.{md,json}                 # Phase 8d
│   ├── issue_tracking.csv                  # Phase 8d 必产
│   ├── issue_tracking.xlsx                 # Phase 8d 可选
│   └── budget_exceeded.md                  # 触硬上限时产
└── baseline/
    ├── {date}/                            # 历史基线
    ├── last_insights.md                    # Phase 9 累积
    ├── hit_count.json
    ├── stale_patterns.json
    └── suite_version.json
```

写入 session_config.json（含 budget 段见 OUTPUT_STANDARD §3.1 默认值）。
写入 progress.json（见 §3.2 全 phase 置 pending）。

---

## 第四步: 按顺序调度各 Phase 子技能

本 SKILL 通过 `Task(general)` 依次调用各 Phase 子技能。每个 Phase 作为独立的 general 子代理执行, Phase 间数据通过知识图谱文件传递。

### 调度原则

1. **顺序执行**: 按 Phase 0 → 1a → 1b → 1c(可选) → 2a → 2b(可选) → 3 → 4a → 4b → 5 → 6 → 7 → 8a → 8b → 8c(可选) → 8d → 9 顺序
2. **断点续传**: 每个 Phase 调用前读 progress.json, 状态为 complete 跳过, failed/in_progress 询问用户是否重做
3. **失败处理**: Phase 失败时 progress.json 标 failed, 询问用户是否继续（fast 模式跳失败 Phase 继续）
4. **数据传递**: Phase 间通过 knowledge_graph/ 目录传数据, 不直接传内存数据
5. **mode=fast 跳过**: 跳 Phase 3/4/5/6/7/9, 仅 0/1a/1b/2a/8a/8d
6. **mode=increment 调整**: Phase 0 manifest 仅含变更范围文件, 其他 Phase 按 manifest 裁切

### 调度顺序表

| 顺序 | Phase | 子技能 SKILL.md | 当前实现状态 |
|------|-------|-----------------|------------|
| 1 | Phase 0 | `skills/source-map/SKILL.md` | ✅ 已实现（子项目 A） |
| 2 | Phase 1a | `skills/entry-recon/SKILL.md` | 待子项目 B |
| 3 | Phase 1b | `skills/authz-map/SKILL.md` | 待子项目 B |
| 4 | Phase 1c | `skills/product-context/SKILL.md` | 待子项目 H (可选) |
| 5 | Phase 2a | `skills/vuln-rules/SKILL.md` | 待子项目 C |
| 6 | Phase 2b | `skills/compliance-audit/SKILL.md` | 待子项目 G (可选, --with-compliance) |
| 7 | Phase 3 | `skills/cross-ref/SKILL.md` | 待子项目 D |
| 8 | Phase 4a | `skills/attack-pattern/SKILL.md` | 待子项目 D |
| 9 | Phase 4b | `skills/attack-reasoning/SKILL.md` | 待子项目 D |
| 10 | Phase 5 | `skills/chain-builder/SKILL.md` | 待子项目 E |
| 11 | Phase 6 | `skills/chain-verify/SKILL.md` | 待子项目 E |
| 12 | Phase 7 | `skills/poc-generator/SKILL.md` | 待子项目 F |
| 13 | Phase 8a | `skills/report-finding/SKILL.md` | 待子项目 F |
| 14 | Phase 8b | `skills/report-chain/SKILL.md` | 待子项目 F |
| 15 | Phase 8c | `skills/compliance-report/SKILL.md` | 待子项目 G (可选) |
| 16 | Phase 8d | `skills/report-summary/SKILL.md` | 待子项目 F |
| 17 | Phase 9 | `skills/pattern-evolve/SKILL.md` | 待子项目 F |

### 每个 Phase 的调用方式

```
Task(general, prompt="
  你是 SourceCPT 的 {Phase名称} 子技能执行者。
  请读取 skills/{phase-skill}/SKILL.md 并严格遵循其指令执行。
  
  会话参数:
  - session_dir: {session_dir}
  - project-path: {project_path}
  - scope: {scope}
  - mode: {mode}
  - changed-since: {changed_since_or_null}
  - context-path: {context_path_or_null}
  - with-compliance: {true_or_false}
  - chain-depth: {chain_depth}
  - baseline: {baseline_path_or_null}

  引用共享规范路径:
  - skills/shared/SRC_ACCESS.md
  - skills/shared/OUTPUT_STANDARD.md
  - skills/shared/OFFSET_RATING.md
  - skills/shared/ANTI_HALLUCINATION.md
  - skills/shared/{其他相关规范}
  
  执行完成后返回 WU 摘要 (≤500 tokens), 含:
  - 处理的文件/节点数
  - 新增图谱节点/边类型与数量
  - 任何警示(如 0 命中 QA 报警)
  - 当前 Phase 完成状态 (complete / failed / partial)
")
```

### 断点续传

每个 Phase 调用前:
1. Read progress.json, 检查该 Phase 状态
2. complete → 跳过, 输出 "✅ Phase {N} 已完成, 跳过"
3. in_progress → 询问用户"上次崩溃于该 Phase, 是否恢复（从最后成功 WU 续）或重做"
4. failed → 询问用户"该 Phase 失败, 是否重做或跳过"
5. pending → 正常调用子代理执行
6. 执行前 progress.json 改 in_progress, last_updated 更新
7. 子代理返回后, 若成功改 complete, 失败改 failed

### Phase 间进度反馈

每个 Phase 完成后, 向用户输出一行进度:

```
✅ Phase {N} 完成 — {Phase名称} — {关键产出摘要}
```

示例:
```
✅ Phase 0 完成 — 工程地图 — manifest 1342文件/202639行/214 入口 /287 sink 已收集
✅ Phase 1a 完成 — 入口面侦察 — 214 入口补齐签名/参数树
...
```

---

## 第五步: 展示摘要

所有已实现的 Phase 完成后, 读取最终报告（若 Phase 8d 已跑）或当前进度（若中断）, 输出:

### 关键数字
- 总文件数 / 总行数（Phase 0）
- 入口面识别数 / sink 索引数
- C1/C2/C3 漏洞分布（Phase 6+）
- 覆盖矩阵完整率（Phase 8d）
- 已实现 Phase 数 / 失败 Phase 数

### 文件路径列表
- session_config: `{session_dir}/session_config.json`
- manifest: `{session_dir}/manifest.json`
- 知识图谱: `{session_dir}/knowledge_graph/`
- 全景报告: `{session_dir}/reports/pentest_report.md` (若 Phase 8d 完成)
- Excel 跟踪表: `{session_dir}/reports/issue_tracking.csv` (若 Phase 8d 完成)

---

## 子项目 A 当前实现状态

子项目 A 仅实现 Phase 0, 其他 Phase 调用框架已就绪但对应 SKILL.md 尚未编写（在对应子项目计划中实现）:
- 调用未实现的 Phase 时, 子代理 prompt 报错 "skills/{phase}/SKILL.md 文件不存在"
- Pipeline 入口会输出"该 Phase 待子项目 {X} 实现, 跳过"
- 用户只跑 Phase 0 时 Pipeline 自动检测 progress.json: 仅 phase0 完成即输出 Phase 0 摘要正常退出

---

## 禁止事项

本 SKILL **严格禁止**:
- 不执行检测命令（只负责参数收集、调度、展示, 不亲自跑 grep/find/Read 源码）
- 不读原始检测数据（原始数据由各 Phase 子技能读, 入口仅 Read session_config.json/progress.json 等元数据）
- 不跳过断点续传检查（每 Phase 前必读 progress.json）
- 不在 Phase 间直接传内存数据（必通过 knowledge_graph/ 持久层）