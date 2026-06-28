# SourceCPT 子项目 A: 基础设施 + Phase 0 工程地图

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 搭建 SourceCPT 白盒源码审计套件的基础设施——8 个共享规范 + Pipeline 入口 SKILL + Phase 0 工程地图 SKILL。跑通后可对任意本地源码项目执行 `$sourcecpt` 命令，产出 manifest.json + dispatch_plan.json + 入口/sink 知识图谱索引。

**架构：** 纯 Markdown SKILL.md 指令文件 + Markdown 知识库 + JSON 格式约定。无任何代码脚本。LLM 按 SKILL.md 中文指令执行 Glob/bash/Read/Write，产出 JSON 图谱节点。按 Anthropic SKILL 标准结构编写：YAML frontmatter（name/description）+ 概述 + MUST 输入/输出契约 + 步骤化工作流 + 引用 shared 规范 + 专属反幻觉约束 + 验收门 + 边界声明。该结构为 SKILL 通用最佳实践, 非任一项目特有。

**技术栈：** opencode SKILL 插件格式（YAML frontmatter + Markdown 指令）、JSON 图谱、bash/Glob/Grep（opencode 原生工具）。

---

## 文件结构

本计划创建以下文件，按职责拆分，每个文件一个明确的职责：

```
~/.config/opencode/skills/sourcecpt/
├── SKILL.md                              # Pipeline 入口: 参数收集 + 环境验证 + 工作目录初始化 + 调度各 Phase
├── skills/
│   ├── shared/
│   │   ├── SRC_ACCESS.md                  # 文件读取/分块/按需加载/临时方法级图谱规范
│   │   ├── OUTPUT_STANDARD.md             # 输出格式/五态/图谱 JSON schema/命名规范
│   │   ├── OFFSET_RATING.md               # 审计深度 T0-T3 分层定义
│   │   ├── CHAIN_STANDARD.md              # 链节点格式/可达性判定规则(Phase 5 预备)
│   │   ├── LOOP_POLICY.md                 # 局部 loop 策略/硬停止/Token 预算
│   │   ├── TRACKING_SHEET.md              # Excel 跟踪表字段定义 + 类型三列映射表
│   │   ├── BASELINE_DIFF.md               # 基线对比协议(持续审计)
│   │   └── ANTI_HALLUCINATION.md          # 反幻觉 56 条全局约束
│   └── source-map/
│       └── SKILL.md                       # Phase 0 工程地图: manifest + 入口索引 + sink 索引 + 派发策略
├── knowledge-bases/
│   ├── entry-types.md                     # 13 入口面 grep 模式定义
│   └── sink-types.md                      # 28 sink 类 grep 模式定义
└── README.md                              # 套件说明 + 安装 + 触发方式
```

**设计原则：**
- 每个 shared 规范独立可引用，被多个 Phase SKILL.md 引用
- source-map 是唯一执行的 Phase，它引用 shared 规范 + knowledge-bases
- Pipeline 入口 SKILL.md 不执行检测逻辑，只做参数收集 + 初始化 + 调度
- knowledge-bases 的 entry-types / sink-types 是 Phase 0 的规则输入，Phase 0 不自行定义 grep 模式

---

## 验证方式

与 writing-skills 技能的 TDD 适配——SKILL.md 是"源码"，跑 SKILL 产生的 JSON 是否符合预定 schema 是"测试"：

1. **写完一个 SKILL.md 后**：在 opencode 中开个会话，给出一个测试项目路径，调用 `$source-map`（或通过 Pipeline 入口），观察产出
2. **验收标准**：产出文件存在且 JSON schema 符合 OUTPUT_STANDARD 定义
3. **反幻觉校验**：抽查产出中是否有"凭记忆"内容（如 sink 节点行号与实际文件不符）
4. **无占位符扫描**：产出的 JSON 中无 `TODO`/`TBD`/`{{}}` 残留

测试项目用 `/tmp/opencode/sourcecpt-test-project/`，内含一个小型 Java + Python 混合项目（~500 行），预先准备入口和 sink 供验证。

---

## 任务列表

### 任务 1：创建目录结构

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/` 及子目录

- [ ] **步骤 1：创建完整目录树**

```bash
mkdir -p ~/.config/opencode/skills/sourcecpt/skills/shared
mkdir -p ~/.config/opencode/skills/sourcecpt/skills/source-map
mkdir -p ~/.config/opencode/skills/sourcecpt/knowledge-bases
```

- [ ] **步骤 2：验证目录结构**

运行：`ls -R ~/.config/opencode/skills/sourcecpt/`
预期：显示 `skills/shared/`、`skills/source-map/`、`knowledge-bases/` 三个子目录

- [ ] **步骤 3：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/ && git commit -m "feat(sourcecpt): scaffold directory structure for skill suite A"
```

---

### 任务 2：编写 OUTPUT_STANDARD.md（输出格式标准）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/shared/OUTPUT_STANDARD.md`

这是所有 Phase 写数据的强制标准——文件命名、五态标记、JSON schema、知识图谱节点/边格式。节点与边的 schema 按白盒语义重新设计（entry/sink/vuln_candidate 节点 + data_flow/cross_ref/chain_steps 边）, 不照搬任何黑盒套件的结构——白盒污点追踪的数据模型与运行时侦察差异极大, 不应套用。

- [ ] **步骤 1：编写 OUTPUT_STANDARD.md 全文**

写入文件 `~/.config/opencode/skills/sourcecpt/skills/shared/OUTPUT_STANDARD.md`，内容含：

```markdown
# 输出格式标准（OUTPUT_STANDARD）

> 适用范围：所有 SourceCPT 子技能在向工作目录写数据时必须遵循本标准。
> 约束力：**强制**。违反将导致 QA 结构校验失败、断点续传失败或下游 Phase 读取异常。

---

## 1. 文件命名规范

### 1.1 通用规则

- 文件名一律 **snake_case**：仅小写字母、数字、下划线；禁止空格（`cross-ref/` 为约定例外）、中文、特殊符号。
- 所有产出文件落点严格限定在 **`{session_dir}/`** 工作目录树内，禁止写入会话目录以外。
- 同一类产出的多种格式同名同根：`xxx.md` 与 `xxx.json` 必须共用同名。
- 进度/任务/配置类文件位于工作目录根：`session_config.json`、`progress.json`、`manifest.json`、`audit_log.json`。
- 临时文件只能写入 `{session_dir}/tmp/`，不得污染 `evidence/`、`knowledge_graph/`、`reports/`。

### 1.2 证据目录按 Phase 分子目录

| 子目录 | 责任 Phase | 内容 |
|--------|-----------|------|
| `evidence/phase0/` | 0 | 工程地图原始 grep 输出 |
| `evidence/phase1a/` | 1a | 入口面补齐证据 |
| `evidence/phase1b/` | 1b | 鉴权面梳理证据 |
| `evidence/phase1c/` | 1c | 产品上下文证据（可选） |
| `evidence/phase2a/` | 2a | 漏洞规则匹配证据 |
| `evidence/phase2b/` | 2b | 编码合规证据（可选） |
| `evidence/cross-ref/` | 3 | 交叉关联证据 |
| `evidence/phase4a/` | 4a | 模式匹配证据 |
| `evidence/phase4b/` | 4b | LLM 推理证据 + insights.md |
| `evidence/phase5/` | 5 | 链 + sibling 扫描证据 |
| `evidence/phase6/` | 6 | 挑战者对抗证据 |
| `evidence/phase7/` | 7 | POC 包 |
| `evidence/phase9/` | 9 | 进化报告 |

### 1.3 知识图谱文件

`knowledge_graph/` 下分 `nodes/`、`edges/`：

- 节点（JSON 数组，每文件一类）：
  - `nodes/entries.json` — 入口面节点
  - `nodes/sinks.json` — sink 索引节点
  - `nodes/modules.json` — 模块骨架节点
  - `nodes/authz_configs.json` — 全局鉴权配置节点
  - `nodes/constraints.json` — 业务约束节点（Phase 1c 可选）
  - `nodes/compliance_violations.json` — 合规违规节点（Phase 2b 可选）
  - `nodes/vuln_candidates.json` — 漏洞候选节点
  - `nodes/pattern_matches.json` — 模式匹配结果
  - `nodes/chains.json` — 漏洞链节点
  - `nodes/verified_vulns.json` — Phase 6 终态节点

- 边（JSON 数组，每文件一类）：
  - `edges/calls_module.json` — 模块间调用关系
  - `edges/entry_in_module.json` — 入口归属模块
  - `edges/entry_calls.json` — 入口调用方法（Phase 1a 产出）
  - `edges/entry_covered_by.json` — 入口被鉴权配置覆盖
  - `edges/data_flow.json` — source→sink 数据流
  - `edges/cross_ref.json` — 交叉关联
  - `edges/upgrade.json` — 合规→漏洞候选升级
  - `edges/chain_steps.json` — 链步骤
  - `edges/sibling_of.json` — 兄弟关系
  - `edges/business_flow.json` — 业务流程（Phase 1c 可选）
  - `edges/constraint_to_code.json` — 约束→代码关联（Phase 1c 可选）
  - `edges/authz_deviation.json` — 鉴权偏离（Phase 1b 产出）

### 1.4 报告文件

`reports/` 下报告成对产出 `.md` + `.json` + 可选 `.sarif`：
- `pentest_report.md` + `.json` + `.sarif`（Phase 8d）
- `compliance_report.md` + `.json`（Phase 8c 可选）
- `chain_report.md` + `.json`（Phase 8b）
- `patch_suggestion.md` + `.json`（Phase 8d）
- `qa_report.md` + `.json`（Phase 8d）
- `issue_tracking.csv` + `.xlsx`（Phase 8d）

---

## 2. 每个 Phase 的 MUST 输出清单

> 下列文件为对应 Phase 完成判定的硬性检查项，任一缺失或为空即视为该 Phase 未完成。

### Phase 0 — 工程地图

**MUST 输入**：`project-path`、`scope`、`mode`

**MUST 输出**：
- `session_config.json`（含 language_fingerprint / total_loc / dispatch_strategy / budget）
- `manifest.json`（非空，全文件清单含 path/lang/loc/sha256/state/phase0_category/skip_reason）
- `dispatch_plan.json`（含 strategy / batches / max_parallel）
- `audit_log.json`（初值空数组 `[]`）
- `progress.json`（phases.phase0 = complete）
- `knowledge_graph/nodes/entries.json`
- `knowledge_graph/nodes/sinks.json`
- `knowledge_graph/nodes/modules.json`
- `knowledge_graph/edges/calls_module.json`
- `evidence/phase0/raw/`（非空，每个 grep 输出带来源与时间戳）

（其余 Phase 的 MUST 输出在对应子项目计划中定义）

---

## 3. JSON Schema 定义

### 3.1 session_config.json

```json
{
  "session_id": "uuid",
  "created_at": "ISO8601",
  "target": {
    "project_path": "/abs/path",
    "scope": "java,python",
    "mode": "full",
    "context_path": null,
    "with_compliance": false
  },
  "language_fingerprint": {
    "java": {"file_count": 234, "loc": 89000},
    "python": {"file_count": 56, "loc": 12000}
  },
  "total_loc": 101000,
  "dispatch_strategy": "single-batch",
  "budget": {
    "total_token_limit": 5000000000,
    "phase2b_semantic": 3000000000,
    "loop_rounds_per_loop": 3,
    "loop_token_per_loop": 50000,
    "total_loop_tokens": 500000000,
    "hard_stop_action": "write budget_exceeded.md + save state + human intervene"
  },
  "suite_version": "V1.0",
  "auto_high_risk_exec_count": 0
}
```

### 3.2 progress.json

```json
{
  "phases": {
    "phase0": "pending",
    "phase1a": "pending",
    "phase1b": "pending",
    "phase1c": "pending",
    "phase2a": "pending",
    "phase2b": "pending",
    "phase3": "pending",
    "phase4a": "pending",
    "phase4b": "pending",
    "phase5": "pending",
    "phase6": "pending",
    "phase7": "pending",
    "phase8a": "pending",
    "phase8b": "pending",
    "phase8c": "pending",
    "phase8d": "pending",
    "phase9": "pending"
  },
  "current_phase": null,
  "last_updated": "ISO8601"
}
```

### 3.3 manifest.json

```json
[
  {
    "path": "src/com/foo/BarController.java",
    "lang": "java",
    "loc": 234,
    "sha256": "abc123...",
    "state": "[ ]",
    "phase0_category": "source",
    "skip_reason": null,
    "audit_owner": null
  }
]
```

state 取值：
- `[ ]` — 未检查（初值）
- `[x]` — 已确认（后续 Phase 标）
- `[?]` — 疑似
- `[-]` — 不适用（测试/三方库/构建产物等，必填 skip_reason）
- `[!]` — 环境干扰

### 3.4 dispatch_plan.json

```json
{
  "strategy": "single-batch|module-batched|subdir-dispatch|increment",
  "total_loc": 3124567,
  "batches": [
    {"id": "B1", "modules": ["user-svc", "auth-svc"], "loc": 280000, "owner_phase2": "todo"}
  ],
  "max_parallel": 3
}
```

### 3.5 audit_log.json

```json
[
  {
    "item_id": "src/com/foo/BarController.java",
    "phase": "0",
    "verdict": "indexed",
    "timestamp": "ISO8601",
    "wu_id": "phase0-batch-1"
  }
]
```

### 3.6 知识图谱节点 schemas

#### entries.json 节点

```json
{
  "node_type": "entry",
  "entry_type": "rest|rpc|mq|ws|graphql|cron|cli|script|deser|file|webservice|custom_proto|event",
  "path": "src/controller/UserController.java",
  "line": 45,
  "signature": "GET /api/users/{id} → getUser(@PathVariable Long id)",
  "param_tree": [
    {"name": "id", "type": "Long", "source": "path", "validated": "unknown"}
  ],
  "module": "user-service",
  "authz_state": "unknown",
  "authz_ref": null,
  "audit_state": "[ ]"
}
```

#### sinks.json 节点

```json
{
  "node_type": "sink",
  "sink_type": "sql_splice|cmd_exec|deser|xxe|ssrf|...",
  "owasp": "A01",
  "cwe": "CWE-89",
  "path": "src/dao/UserDao.java",
  "line": 102,
  "code_snippet": "PreparedStatement st = conn.prepareStatement(\"SELECT * FROM users WHERE id=\" + userId)",
  "audit_state": "[ ]"
}
```

#### modules.json 节点

```json
{
  "node_type": "module",
  "module_id": "user-service",
  "path": "src/user-service/",
  "loc": 280000,
  "lang": "java",
  "entry_count": 45,
  "sink_count": 12
}
```

### 3.7 知识图谱边 schemas

#### calls_module.json 边

```json
{
  "edge_type": "calls_module",
  "from": "user-service",
  "to": "auth-service",
  "evidence": "import com.foo.auth.*; in UserController.java"
}
```

---

## 4. 五态标记闭环

| 标记 | 含义 | 合规语义 | 漏洞语义 |
|------|------|---------|---------|
| `[x]` | 已确认 | fail 确认 | VULN confirmed |
| `[?]` | 疑似 | 可疑发现需深审 | SUSPENDED 待人工 |
| `[-]` | 不适用 | na / pass | disproved 已证伪 |
| `[!]` | 环境干扰 | 命令失败 | 被安全机制阻断 |
| `[ ]` | 未检查 | **必须消灭** | **必须消灭** |

**强制规则**：QA 第一层校验扫描 manifest / 候选 / 漏洞终态 JSON 中所有 `audit_state` / `state` 字段，残留 `[ ]` 即拒收报告。所有 `[ ]` 必须在对应 Phase 结束前升级为 `[x]/[?]/[-]/[!]` 之一。

---

## 5. WU 落盘规则

- 每个 WU 完成后**立即**写盘 audit_log.json（追加 JSON 数组条目）
- WU 产出 evidence 文件立即写 `evidence/phase{N}/` 对应子目录
- WU 产出图谱节点/边立即写 `knowledge_graph/nodes|edges/` 对应 JSON 文件
- **不依赖子代理会话上下文维持状态**——下个 WU 输入明确指定文件路径，不靠"上轮记忆"
- 子代理崩溃不影响已完成 WU 结果（落盘即持久）

---

## 6. 反幻觉相关条目（本规范引用 ANTI_HALLUCINATION.md）

本规范被反幻觉 #2、#3、#6、#16、#34、#44 引用。违反即触发对应约束。
```

- [ ] **步骤 2：验证文件存在且非空**

运行：`test -s ~/.config/opencode/skills/sourcecpt/skills/shared/OUTPUT_STANDARD.md && echo OK || echo FAIL`
预期：OK

- [ ] **步骤 3：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/skills/shared/OUTPUT_STANDARD.md && git commit -m "feat(sourcecpt): add OUTPUT_STANDARD shared spec — file naming, JSON schemas, five-state closure"
```

---

### 任务 3：编写 ANTI_HALLUCINATION.md（反幻觉 56 条全局约束）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/shared/ANTI_HALLUCINATION.md`

集中记录 56 条反幻觉约束（全局 10 + Phase 专属 46 含 3 占位），每个 Phase SKILL.md 引用对应编号。这是 SourceCPT 防止 LLM 编造漏洞证据的核心规范。

- [ ] **步骤 1：编写 ANTI_HALLUCINATION.md 全文**

写入文件，内容含 56 条约束按 Phase 分组表列。约束文本取自设计规格第八部分（无需重新构思），分四节：

**全局继承 GenCPT（10 条）** —— #1 不准凭记忆出攻击命令 / #2 不准伪造代码行号输出 / #3 无证据不写确认态 / #4 超审批立即停 / #5 省略词零容忍 / #6 占位符必替换 / #7 baseline 永不替代当前 / #8-10 保留 GenCPT 同前义对应白盒。

**Phase 专属（49 条含 3 占位）** —— 按 Phase 分组：

- 5/6: #11 不许跳过 Sibling-Scan
- 2b 旧 grep 版占位: #12 #13 #14 (弃用保留占位, 实际约束见 #15-17)
- 2b 语义版: #15 不许只扫字面量关键字 / #16 每函数必标审计结果 / #17 跨块切断违规不许直接判 false
- 2a: #18 不许凭 sink 直判 VULN / #19 不许跳过证伪条件 / #20 propagation 每跳必 Read / #21 source 不可控不许硬判
- 3: #22 三库新假设必引原文 / #23 静态库未命中不许跳过 LLM 推理
- 4a: #24 不许凭模式命中直判 C1 / #25 证伪必检 / #26 模式不命中不许跳过推理
- 4b: #27 ReAct 链必完整 / #28 推理新候选标 llm_reasoning / #29 insights 不许直接晋升
- 全局: #30 不许循环重跑已完成 Phase / #31 增量审计不许降级 baseline / #43 规则库/知识库/图谱必须按需加载
- 5: #32 链深超 5 必报截断声明 / #33 sibling-scan 禁跳任何兄弟 / #34 已扫兄弟必留 audit_state / #35 推翻原候选不许隐藏
- 6: #36 挑战者不许读分析方推理 / #37 六项逐项必输出 verdict / #38 对抗 loop 每轮必新 Read 证据 / #39 推翻结论不许隐藏 / #40 C1 必须六项全 pass / #41 SUSPENDED 不许擅升 VULN
- 1c: #42 文档约束不许单独判 VULN
- 8a: #44 每发现必含五段证据链 / #45 每产出必并出 md+json
- 7: #46 C3/NOVULN/SUSPENDED 不许生成 POC / #47 POC 不许填真实凭证 / #48 payload 必沿 evidence_chain 验证可达 / #49 非 Web 入口不许伪 HTTP 包
- 8d: #50 占位符必替换 / #51 覆盖矩阵不可 Override / #52 盲区必须显式列 / #53 修复建议必基于实际代码 / #58 跟踪表必含完整字段状态初值"未修复" / #59 跟踪表类型必含三列
- 9: #54 Insights 不许跳过 4 门槛 / #55 拒绝后不许再晋升 / #56 自净剔除不许隐藏 / #57 _learned 模式 8 段必完整

每条约束格式（表式陈列）：

```markdown
| # | Phase | 约束 |
|---|------|------|
| 1 | 全局 | 不准凭记忆出攻击命令——必须 Read 规则文件/grep 输出后才执行 |
```

文件结尾加说明:

```markdown
## 引用方式

各 Phase SKILL.md 中对应违反场景引用本文件条目编号.
本规范被所有 Phase 子 SKILL.md 引用, 是反幻觉的核心防线.
```

- [ ] **步骤 2：验证条目计数**

运行：`grep -c "^| [0-9]" ~/.config/opencode/skills/sourcecpt/skills/shared/ANTI_HALLUCINATION.md`
预期：56 (含 3 占位 #12-14)

- [ ] **步骤 3：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/skills/shared/ANTI_HALLUCINATION.md && git commit -m "feat(sourcecpt): add ANTI_HALLUCINATION 56 rules (10 global GenCPT + 46 phase-specific + 3 reserved)"
```

---

### 任务 4：编写 SRC_ACCESS.md（文件读取与按需加载规范）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/shared/SRC_ACCESS.md`

定义白盒文件读取规范——大项目分块、按需加载、临时方法级图谱展开、防会话压缩丢数据三层原则。这是白盒专属规范, 不存在 SSH 限速等内容——黑盒的 SSH 规范全无借鉴价值，直接按白盒语义自构。

- [ ] **步骤 1：编写 SRC_ACCESS.md 全文**

写入文件，内容含六节：

**§1 文件读取工具约定** — Read/Glob/Grep 工具优先级+输入路径必须绝对路径+UTF-8 强制。

**§2 分块策略** — 按 500 行/块切，函数边界对齐，5 行重叠防跨块漏；大于 3000 行递归切分深度 ≤3。

**§3 按需加载原则**:
- 规则库(2a/4a): 仅 sink_type/候选命中规则文件，不预载全 vuln-rules/
- 模式库(4a): 仅条件触发表命中的模式 8 段 SKILL.md
- Phase 2b 分组: 仅读该组规则(G1-*.md)
- 图谱(全Phase): 按 sink/entry/candidate 查询过滤，不全量载 knowledge_graph/
- 子代理 WU: 输入明确指定路径，不依赖会话记忆

**§4 临时方法级图谱**:
- 持久层: 模块级长期存图谱(知模块边界,调用量)
- 临时层: 方法级随 sink 展开(污点追踪触达 sink 时反向递归建方法调用链 5 跳)，用后弃
- 不入 knowledge_graph 持久层

**§5 防会话压缩丢数据三层兜底**:
1. 任何 trace 立即落盘不靠会话上下文维持
2. 子代理下个 WU 不依赖上轮记忆，只重读磁盘状态
3. QA 第一层强制校验 audit_log + manifest + sibling_scan_log 完整性

设计哲学："内存不可信，磁盘可信"。

**§6 反幻觉引用**:
被反幻觉 #1、#20、#43 引用。不对应用 Write/Read/Glob/Grep 凭记忆调用，必按本规范分块/按需。

- [ ] **步骤 2：验证文件存在**

运行：`test -s ~/.config/opencode/skills/sourcecpt/skills/shared/SRC_ACCESS.md && echo OK || echo FAIL`
预期：OK

- [ ] **步骤 3：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/skills/shared/SRC_ACCESS.md && git commit -m "feat(sourcecpt): add SRC_ACCESS spec — file chunking, on-demand loading, temp method graph, anti-session-compression"
```

---

### 任务 5：编写 OFFSET_RATING.md（审计深度 T0-T3 分层）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/shared/OFFSET_RATING.md`

定义白盒审计深度分层，替代 GenCPT 的 L0/L1/L2/L3 黑盒语义。

- [ ] **步骤 1：编写 OFFSET_RATING.md 全文**

写入文件，内容含：

```markdown
# 审计深度分层（OFFSET_RATING）

> 适用范围：所有 Phase 在标注执行深度时使用 T0-T3 分层。
> 对照：GenCPT L0-L3 黑盒语义，T0-T3 是白盒语义，不执行命令只读代码。

---

## T0 — 单文件模式匹配

**执行方式**：Glob/Grep + Read ±20 行
**用途**：识别入口面、sink 索引、grep 模式命中
**Phase 使用**：0、2b 第 1 层
**产出**：节点清单、raw 命中行

---

## T1 — 跨函数污点追踪

**执行方式**：Read sink 所在函数 + 沿调用链反查 2-5 跳
**用途**：source → propagation → sink 路径追踪
**Phase 使用**：2a、3、4a、5
**产出**：evidence_chain propagation 链

---

## T2 — 跨模块可达性验证

**执行方式**：Read 跨模块文件 + 临时方法级图谱展开
**用途**：链节点跨模块可达性、鉴权配置覆盖、业务约束对照
**Phase 使用**：4b、5、6
**产出**：可达性判定、节点间可达证据

---

## T3 — 业务逻辑推断

**执行方式**：Read 业务文档(若有 1c) + 代码 + 推理
**用途**：业务逻辑漏洞、状态机偏离、权限矩阵推断
**Phase 使用**：4b 业务漏洞推理、6 挑战者业务挑战
**产出**：deviated 候选、业务漏洞证据

---

## 与 GenCPT L0-L3 对照

| GenCPT 黑盒 | SourceCPT 白盒 | 本质差异 |
|------------|---------------|---------|
| L0 宿主机观察(SSH) | T0 单文件模式匹配(本地 grep) | 不远程不执行 |
| L1 容器内观察(kubectl exec) | T1 跨函数污点追踪(Read 调用链) | 不执行只读 |
| L2 容器内攻击验证(差分) | T2 跨模块可达性(Read 邻居) | 不需复现运行 |
| L3 理论验证(破坏性) | T3 业务逻辑推断(语义) | 白盒强项 |

---

## 反幻觉引用

被 #18、#20、#24 引用。深度判定必须有 Read 证据支撑，不许凭记忆标深度。
```

- [ ] **步骤 2：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/skills/shared/OFFSET_RATING.md && git commit -m "feat(sourcecpt): add OFFSET_RATING T0-T3 audit depth layers (replaces GenCPT L0-L3)"
```

---

### 任务 6：编写 CHAIN_STANDARD.md（链节点格式与可达性）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/shared/CHAIN_STANDARD.md`

预备 Phase 5 漏洞链构建使用的标准——链节点格式、可达性判定规则、深度上限。

- [ ] **步骤 1：编写 CHAIN_STANDARD.md 全文**

写入文件，内容含：

```markdown
# 链标准（CHAIN_STANDARD）

> 适用范围：Phase 5 链构建 + Phase 6 链验证 + Phase 8b 链报告。
> 此文件预备 Phase 5 使用，本子项目 A 不直接验证链构建。

---

## 1. 链节点格式

每条链 CHAIN-xxx 包含：

```json
{
  "node_type": "chain",
  "chain_id": "CHAIN-001",
  "steps": [
    {"step": 1, "node_ref": "CAND-001", "verdict": "C2_condition_met", "evidence_ref": "..."},
    {"step": 2, "node_ref": "compliance_violation/G1-01 + ATK-HYP-008", ...}
  ],
  "total_impact": "unauth RCE",
  "reachability": "confirmed_partial|confirmed_full|theoretical",
  "depth": 4,
  "audit_state": "[ ]"
}
```

## 2. 可达性判定规则

| reachability | 含义 | 判定条件 |
|-------------|------|---------|
| confirmed_full | 全步骤路径已 Read 验证 | 每步骤 evidence_chain 都有完整 propagation 链 |
| confirmed_partial | 部分步骤已验证 + 部分 T2 推断 | 主路径已 Read, 部分远端假设 |
| theoretical | 全部 T3 推断 | 无 Read 证据, 仅业务逻辑推演 |

## 3. 深度上限

- 默认 chain-depth = 5
- 硬上限 chain-depth = 8 (防组合爆炸)
- 超过 5 必须在链 markdown 中写"深度截断声明"
- 超过 8 即停止扩展, 标 [!] 截断

## 4. 反幻觉引用

被 #32 引用。链深超 5 必报截断声明。
```

- [ ] **步骤 2：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/skills/shared/CHAIN_STANDARD.md && git commit -m "feat(sourcecpt): add CHAIN_STANDARD for future Phase 5/6/8b chain building"
```

---

### 任务 7：编写 LOOP_POLICY.md（局部 loop 策略与硬停止）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/shared/LOOP_POLICY.md`

定义局部 loop 适用范围、禁用范围、硬停止条件。这是 loop engineering 借鉴后的审慎落地规范。

- [ ] **步骤 1：编写 LOOP_POLICY.md 全文**

写入文件，内容含：

```markdown
# Loop 使用策略（LOOP_POLICY）

> 适用范围：所有 Phase 子流程中是否启用循环。
> 借鉴：loop engineering (Addy Osmani, Boris Cherny) 部分采纳——主体不 loop, 局部≤3 轮。
> 对照：GenCPT 无 loop 概念, 是顺序 Pipeline + 断点续传。

---

## 1. 三个适用 loop 的子流程(局部, ≤3 轮, Token 硬上限)

| 子流程 | loop 触发条件 | 验收方 | 停止条件 |
|--------|-------------|--------|---------|
| 1b 鉴权遗漏反查 | Phase 1b 全局识别完后 | 独立子代理反查是否有遗漏全局过滤 | 新增项≤0 或 ≤3 轮 |
| 2b uncertain 扩读 | Phase 2b uncertain 命中项 | 子代理扩读 ±100 行 | verdict 收敛或 ≤3 轮 |
| 6 对抗仲裁 | Phase 6 第一轮挑战后争议 | 第三方仲裁子代理 | 共识达成或 ≤3 轮 |

每轮 self-contained — 下轮子代理只读上轮结论+源码, 不读推理过程.
每轮必输出新 Read 证据, 防 LLM 持续加固幻觉.

---

## 2. 禁用 loop 的核心判定

- Phase 4a 证伪条件检查: 单次精确判定, 不许循环
- Phase 4b ReAct 已是循环(Thought-Action-Observation), 不嵌套外部 loop
- Phase 5 链构建: 组合优化, 不"不合格重来", 重定义链路更贵
- C1/C2/C3 升级: 必须人审 Echo, 不许 loop 自升
- Pipeline 顶层: 9 Phase 顺序调度, 不许 Agent 自动跳 Phase (破坏审计 trail)

---

## 3. Loop 硬停止条件

| 限制 | 值 | 触发动作 |
|------|----|---------|
| 单子流程 loop 轮次 | ≤3 | 写 evidence/phase{N}_loop/loop_state.json |
| 单 loop Token | ≤50000 | 触及写阻塞报告 |
| 全 Pipeline loop 累计 Token | ≤5亿 | 触及写 budget_exceeded.md |
| 全 Pipeline 总 Token | ≤50亿 | 触及写 budget_exceeded.md + 状态保存 + 人介入 |

---

## 4. 反幻觉引用

被 #30 (不许循环重跑已完成 Phase)、#38 (对抗 loop 每轮必新 Read) 引用.
```

- [ ] **步骤 2：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/skills/shared/LOOP_POLICY.md && git commit -m "feat(sourcecpt): add LOOP_POLICY — local loop scope, hard-stop, token budget"
```

---

### 任务 8：编写 TRACKING_SHEET.md（Excel 跟踪表字段与类型映射）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/shared/TRACKING_SHEET.md`

定义 Phase 8d 输出 Excel 跟踪表字段格式 + 类型三列(编号+中文名+子类型)映射表。预备 Phase 8d 使用。

- [ ] **步骤 1：编写 TRACKING_SHEET.md 全文**

写入文件，内容含两节：

**§1 跟踪表字段定义** — 表列 19 列: ID / 类型编号(OWASP+CWE) / 类型名称(中文) / 子类型 / 标题 / 可信度 / 位置(文件:行) / 入口 / 鉴权依赖 / 影响 / 链 ID / Sibling / POC 引用 / 修复优先级 / 修复建议引用 / 状态(初值"未修复") / 残余风险 / 审计日期。含 xlsx 3 sheet 表结构(Issues/Chains/Summary)。

**§2 类型编号→类型名称→子类型映射表** — 完整 20 类映射表内容取自设计规格 `docs/superpowers/specs/2026-06-28-sourcecpt-design.md` Phase 8d 章节中已定义的表(CWE-89→SQL注入→字符串拼接式/ORM原生SQL式/MyBatis ${}占位符式；CWE-78→命令注入→Runtime.exec拼接/ProcessBuilder拼接/Shell -c注入式;... 共 20 类)。执行者从该规格直接复制粘贴该表, 不需重新构思。

文件结尾说明：

```markdown
## 反幻觉引用
被 #58、#59 引用。跟踪表必含完整字段状态初值"未修复"; 类型必从映射表查填, 不许凭记忆瞎填。
```

- [ ] **步骤 2：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/skills/shared/TRACKING_SHEET.md && git commit -m "feat(sourcecpt): add TRACKING_SHEET spec — 19 columns + 20 CWE type mapping for Excel tracking"
```

---

### 任务 9：编写 BASELINE_DIFF.md（基线对比协议）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/shared/BASELINE_DIFF.md`

定义持续审计模式下基线对比机制。预备子项目 H 使用，本子项目仅写规范不执行。

- [ ] **步骤 1：编写 BASELINE_DIFF.md 全文**

写入文件，内容含：

```markdown
# Baseline 对比协议（BASELINE_DIFF）

> 适用范围：持续审计模式（mode=increment）。
> 预备：子项目 H（持续审计 + Phase 1c）使用。

---

## 1. 基线建立

每次 full 审计完成后，复制核心产物到 `baseline/{date}/`:
- audit_log.json
- vuln_candidates.json
- compliance_violations.json (若启用)
- pattern_matches.json
- verified_vulns.json

写 `baseline/suite_version.json` 记录版本。

## 2. 增量对比

下次 increment 审计时:
Phase 0 对比当前状态 vs baseline/{last_date}/:
- diff 文件清单 (git diff 覆盖)
- diff 已确认漏洞 (过去确认的不再重报, 只报新出现/消失)
- diff 合规违规 (过去已记录 fail 项不再重报)

## 3. 报告对 diff 章节

Phase 8d 全景报告必含 baseline diff 章节:
"新增漏洞: N / 修复漏洞: M / 仍存在未修: K / 新增违规: L"

## 4. 版本兼容

suite_version 不一致 → 降级为趋势对比, baseline 不替代当前审计.
(base line 永不替代当前审计结果, 即使版本一致也是如此)

## 5. 反幻觉引用

被 #7 (baseline 永不替代当前)、#31 (增量审计不许降级 baseline) 引用。
```

- [ ] **步骤 2：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/skills/shared/BASELINE_DIFF.md && git commit -m "feat(sourcecpt): add BASELINE_DIFF protocol — baseline persistence + diff + version compat"
```

---

### 任务 10：编写 entry-types.md（13 入口面定义）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/knowledge-bases/entry-types.md`

Phase 0 引用此文件做 grep 模式识别 13 入口面。这是 Phase 0 的规则输入。

- [ ] **步骤 1：编写 entry-types.md 全文**

写入文件，内容含 13 行表格 + 每条目的：grep 模式（按语言列出 Java/Go/Python/JS 的精确模式）+ 正例 + 反例 + authz_state 默认约定。

完整 13 项(内容取自设计规格 `2026-06-28-sourcecpt-design.md` 第三部分"13 入口面通道"表, 直接抄录):

1. REST — `@RequestMapping @GetMapping @PostMapping @PutMapping @DeleteMapping` (Java) / `app.get( router.get( flask.route` (其他语言)
2. RPC — `@DubboService grpc.Service impl Service` (Java) / `.proto service` (跨语言)
3. MQ — `@RabbitListener @KafkaListener @StreamListener` (Java) / `consumer.subscribe on_message=` (Python)
4. WebSocket/SSE — `@ServerEndpoint WebSocketHandler socket.on sse.send`
5. GraphQL — `@QueryMapping @MutationMapping type Query @Schema`
6. 定时任务 — `@Scheduled @Cron celery.schedule apscheduler`
7. 命令行 — `main( cobra.Command @CommandLineRunner argparse`
8. 脚本入口 — `*.sh *.py 入口文件`
9. 反序列化 — `@JsonView @XmlElement ObjectInputStream readObject XMLDecoder`
10. 文件类 — `MultipartFile Part Commons FileUpload openReadStream + 用户控制路径的 read`
11. WebService/SOAP — `@WebService @SOAPBinding JAX-WS @WebMethod wsdl`
12. 自定义协议 — `ChannelInitializer Netty Mina IoHandler Decoder`
13. 事件监听 — `@EventListener ApplicationListener @Subscribe EventListener`

每条说明表式:

```markdown
| # | 通道 | grep 模式 | 正例 | 反例 | authz 默认 |
|---|------|---------|------|------|-----------|
| 1 | REST | `@RequestMapping @GetMapping` (Java) / ... | `@GetMapping("/api/users/{id}")` | 普通 `@Bean` 注解 | unknown (必 Phase 1b 梳理) |
```

文件结尾加说明：

```markdown
## 引用方式
Phase 0 SKILL.md 必读此文件执行 Bash grep 识别入口面.
每条命中立即写知识图谱 entry 节点, 不许凭记忆判入口.
被反幻觉#1引用.
```

- [ ] **步骤 2：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/knowledge-bases/entry-types.md && git commit -m "feat(sourcecpt): add entry-types knowledge base — 13 entry channels with grep patterns"
```

---

### 任务 11：编写 sink-types.md（28 sink 类定义）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/knowledge-bases/sink-types.md`

Phase 0 引用此文件做 grep 模式识别 28 类 sink。OWASP Top10 × CWE 双轨组织。

- [ ] **步骤 1：编写 sink-types.md 全文**

写入文件，28 类(内容取自设计规格 `2026-06-28-sourcecpt-design.md` 第三部分"28 sink 类(按 OWASP Top10 × CWE 双轨)"表, 直接抄录扩展为 grep Java/Go/Python 多语言模式):

格式每条:
```markdown
| sink_type | owasp | cwe | grep(Jave/Go/Python) | Source示例 | Sink示例 |
|-----------|-------|-----|---------------------|-----------|---------|
| sql_splice | A01 | CWE-89 | `${} +.query sql % cursor.execute(%)` | @RequestParam | prepareStatement("SELECT..."+id) |
```

28 项(从规格第三部分抄录):
1. 路径遍历文件读 (CWE-22)
2. 反射类加载 (CWE-470)
3. 弱哈希 MD5/SHA1 (CWE-328)
4. 弱加密 DES/RC4 (CWE-327)
5. 硬编码密钥 (CWE-798)
6. 弱随机 (CWE-330)
7. SQL 拼接 (CWE-89)
8. 命令执行 (CWE-78)
9. LDAP 注入 (CWE-90)
10. NoSQL 注入 (CWE-943)
11. XPath 注入 (CWE-643)
12. 表达式注入 (CWE-917)
13. 模板注入 SSTI (CWE-1336)
14. XSLT 引擎 (CWE-91)
15. HTTP 头注入 CRLF (CWE-93)
16. 邮件头部注入 (CWE-93)
17. 日志注入 Log4Shell (CWE-502)
18. 调试端点暴露 (CWE-489)
19. 详细错误回显 (CWE-209)
20. 组件 CVE (CWE-1035)
21. 鉴权缺失 (CWE-862)
22. 越权 IDOR (CWE-639)
23. Java 原生反序列化 (CWE-502)
24. fastjson/XStream (CWE-502)
25. JNDI 注入 (CWE-74)
26. HTTP/Socket SSRF (CWE-918)
27. DNS 解析 (CWE-918)
28. XXE 解析 (CWE-611) + 文件上传 (CWE-434) + 开放重定向 (CWE-601) + XSS 输出 (CWE-79)

文件结尾说明:

```markdown
## 引用方式
Phase 0 SKILL.md 必读此文件执行 Bash grep 识别 sink.
每条命中立即写知识图谱 sink 节点（含 sink_type/owasp/cwe）, 不许凭记忆.
被反幻觉#1引用.
```

- [ ] **步骤 2：Commit**

```bash
cd ~/.config/opencode && git add knowledge-bases/sink-types.md && git commit -m "feat(sourcecpt): add sink-types knowledge base — 28 sink types OWASP x CWE dual-track"
```

---

### 任务 12：编写 source-map/SKILL.md（Phase 0 工程地图——核心执行 SKILL）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/source-map/SKILL.md`

这是子项目 A 的核心交付——Phase 0 工程地图。执行后产出 manifest/入口/sink 索引/派发策略。本 SKILL.md 按白盒语义设计——黑盒运行时侦察（SSH kubectl/docker exec 拉取集群信息）与白盒工程地图（读本地源码建文件清单/调用图骨架）差异极大, 不应照搬黑盒侦察 SKILL 写法。本 SKILL 关注: Glob 列文件 / bash grep 找入口-sink / 智能派发 300w 行规模, 这些都是白盒独有需求。

- [ ] **步骤 1：编写 source-map/SKILL.md 全文（采用 YAML frontmatter + 中文指令）**

写入文件，结构（frontmatter + MUST 输入 + 引用共享规范 + 反幻觉引用 + 7 步骤核心工作流 + 产出物 + 反幻觉约束 + 验收门 + 边界声明；具体每节内容见下方步骤 1 详细描述，内容不必另行构思）:

```yaml
---
name: source-map
description: >
  白盒源码审计第一步: 工程地图。对目标项目建立全文件清单 manifest、语言指纹、
  入口面索引、sink 索引、模块级调用图骨架、智能派发策略。300w+ 行项目必须先建立
  可审计工作集, 否则后续阶段无覆盖率可言。
  使用场景: 审计启动第一步, 任何 mode 都必须执行。
  不使用场景: 已有完整 manifest.json 的复检会话(--skip-phase0 跳过)。
---
```

后接正文。本文档主体按 7 步骤工作流:

**§1 MUST 输入** — 表列 project-path(必)/scope(默认all)/mode(默认full)/changed-since(可选)

**§2 引用共享规范** — 必读三表: SRC_ACCESS.md / OUTPUT_STANDARD.md / OFFSET_RATING.md, 知识库: knowledge-bases/entry-types.md / sink-types.md

**§3 反幻觉引用** — 本 Phase 受反幻觉 #1(不准凭记忆)/#2(不准伪造)/#6(占位符必替换)约束

**§4 核心工作流**（7 步骤）:

```
#### 步骤 1: 语言指纹识别 [T0]
按 scope 列文件统计, 若 scope=all 取文件数 Top3 语言
- bash 命令示例:
  find {project-path} -type f -name "*.java" -not -path "*/test/*" -not -path "*/node_modules/*" | wc -l
  find {project-path} -type f -name "*.go" -not -path "*/vendor/*" | wc -l
  ...
- 写入 session_config.json language_fingerprint
- 计算 total_loc

#### 步骤 2: 全文件清单 manifest 生成 [T0]
执行 Glob/bash 列全源码文件, 跳过以下目录:
  */test/* */tests/*  -> state=([-] skip_reason="测试代码")
  */node_modules/* */vendor/* */third_party/* -> ([ Skip_it 跳过] skip_reason="三方库")
  *.min.js *.map */dist/* */build/* -> skip_reason="构建产物"
  *.properties *.yml *.xml -> phase0_category="resource"
其余 -> phase0_category="source", state="[ ]"
每文件必含 sha256 (sha256sum 命令) + loc (wc -l)
立即写盘 manifest.json (数组, 每文件一条)

#### 步骤 3: 入口面索引 [T0]
必读 knowledge-bases/entry-types.md
按 13 通道执行 bash grep:
  grep -rn "@RequestMapping\|@GetMapping\|..." --include="*.java" --include="*.go" {project-path}
注意: --include 按 scope 语言限定
每条命中立即写 knowledge_graph/nodes/entries.json (追加):
  {node_type:"entry", entry_type:"rest", path, line, signature:"unknown", 
   param_tree:[], module:"unknown", authz_state:"unknown", audit_state:"[ ]"}
不许凭记忆, 必须有 grep 原始输出
0 命中必须报告(QA 报警可能漏面)

#### 步骤 4: sink 索引 [T0]
必读 knowledge-bases/sink-types.md
按 28 类执行 bash grep, 模式取自知识库
每条命中立即写 knowledge_graph/nodes/sinks.json:
  {node_type:"sink", sink_type:"sql_splice", owasp:"A01", cwe:"CWE-89", 
   path, line, code_snippet:"(截 sink 行)", audit_state:"[ ]"}
0 命中必须 QA 报警(漏面)

#### 步骤 5: 模块级调用图骨架 [T1]
按顶层目录识别模块, 每模块比对模块间引用关系:
  grep -rn "^import .* com\." --include="*.java" {project-path} | awk '...'
  或跨语言按文件夹级别识别
写 knowledge_graph/nodes/modules.json:
  {node_type:"module", module_id:"user-svc", path:"src/user-svc/",
   loc:280000, lang:"java", entry_count:0, sink_count:0}
写 knowledge_graph/edges/calls_module.json:
  {edge_type:"calls_module", from:"user-svc", to:"auth-svc", 
   evidence:"import com.foo.auth.* in UserController.java"}
不建方法级全图 (内存爆炸). 方法级在 Phase 4a 触达 sink 时临时展开.

#### 步骤 6: 智能派发策略 [T0]
按 total_loc 决定 strategy:
  ≤10w: single-batch
  10w-50w: module-batched (按顶层模块分批)
  >50w: subdir-dispatch (子目录派发)
  mode=increment: 仅 git diff + 调用图邻居
写 dispatch_plan.json

#### 步骤 7: 写盘与图谱初始化
- session_config.json (含 language_fingerprint + total_loc + strategy + budget)
- manifest.json + dispatch_plan.json + audit_log.json ([])
- knowledge_graph/nodes/entries.json + sinks.json + modules.json
- knowledge_graph/edges/calls_module.json
- progress.json phases.phase0 -> "complete"
- evidence/phase0/raw/ (每个 grep 输出原始文件留底)
```

§5 产出物 — 文件树清单

**§6 反幻觉约束本 Phase 专属**:
- 不许跳过 manifest 文件, 每跳过必填 skip_reason 且属跳过白名单
- 不许凭记忆判入口/sink, 必须有 bash grep 原始命中行输出
- 占位符必替换, entry path/line 必来自实际文件
- 0 命中(某通道) 必发出 QA 报警, 不许"未发现即安全"

**§7 300w 规模验收门** — 表列 three 验收门:
- manifest 覆盖率 100% (无未跳过未审计文件)
- 13 通道至少识别 N>0 (0 必 QA 报警)
- 28 sink 类每类 N>0 (0 必 QA 报警)
- 派发颗粒每子代理负载 ≤30w 行

**§8 边界声明** — Phase 0 不审漏洞, 只建地图. 识别 sink 不代表漏洞.

- [ ] **步骤 2：验证文件存在且结构合规**

运行：
```bash
test -s ~/.config/opencode/skills/sourcecpt/skills/source-map/SKILL.md && echo OK
head -10 ~/.config/opencode/skills/sourcecpt/skills/source-map/SKILL.md | grep "^name: source-map"
```
预期：OK + 含 frontmatter

- [ ] **步骤 3：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/skills/source-map/SKILL.md && git commit -m "feat(sourcecpt): add source-map Phase 0 SKILL — manifest + entry/sink index + dispatch (300w scale)"
```

---

### 任务 13：编写 Pipeline 入口 SKILL.md

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/SKILL.md`

这是整个套件的入口——参数收集 + 环境验证 + 工作目录初始化 + 调度各 Phase。**不直接执行检测逻辑**——通过 Task(general) 调用各 Phase 子技能 SKILL.md（Pipeline 模式是 SKILL 套件常见架构, 非特定项目专属）。子项目 A 只实现到 Phase 0 调度，其他 Phase 占位标"在对应子项目计划实现"。

- [ ] **步骤 1：编写 Pipeline 入口 SKILL.md 全文**

写入文件，结构按白盒套件入口需求设计:

```yaml
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
```

后接正文，含：

**§1 参数格式定义** — 表（project-path 必填 / mode 默认 full / scope 默认 all / changed-since 可选 / context-path 可选 / --with-compliance flag / chain-depth 默认 5 / baseline 可选）

**§2 mode 选项说明** — fast/full/increment 三档表

**§3 scope 选项说明** — java/go/python/js/ts/all/module=path/changed/owasp-a1..a10 表

**§4 第一步: 参数收集** — 使用 question 工具交互收集 6 参数。project-path 缺失拒绝继续。

**§5 第二步: 环境验证**
1. 检查 project-path 存在 (Read/Glob)
2. 收集环境信息: 语言指纹、总行数
3. tool 可用性检查: 检查 cat / find / grep / sha256sum / git 可用
4. 记录到 session_config.json
若项目路径无效立即终止报告错误。

**§6 第三步: 初始化工作目录**
创建 `/tmp/sourcecpt-{session_id}/` 目录树（完整列出 OUTPUT_STANDARD §1.2 结构）。

**§7 第四步: 按顺序调度各 Phase 子技能**
通过 Task(general) 依次调用。**子项目 A 只实现 Phase 0**，其他 Phase 在对应子项目计划实现:
- 调度方式说明: 每个 Phase 调用方式 = `Task(general, prompt="你是 SourceCPT 的 {Phase 名称} 子技能执行者。请读取 skills/{phase-skill}/SKILL.md 并严格遵循其指令执行。\n会话参数：\n- session_dir: {session_dir}\n- project-path: {path}\n- scope: {scope}\n- mode: {mode}\n执行完成后返回 WU 摘要(≤500 tokens)。")`
- 调度顺序表 (Phase 0-9 共 17 项, 标明当前实现状态: 已实现 / 待子项目计划实现)
- 断点续传原则: 每 Phase 前读 progress.json 检查状态, complete 跳过, in_progress/failed 询问重做

子项目 A 当前 Pipeline 入口只调度 Phase 0 通过 Task(general), 其他 Phase 调度代码框架已写好但标 "[待对应子项目实现]" 在 prompt 中。

**§8 第五步: 展示摘要**
所有已实现 Phase 完成后, 输出关键数字与文件路径列表。

**§9 禁止事项**
- 不执行检测命令 (只参数收集+调度+展示)
- 不跳过断点续传检查
- Phase 间通过知识图谱文件传, 不直接传内存数据

- [ ] **步骤 2：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/SKILL.md && git commit -m "feat(sourcecpt): add Pipeline entry SKILL — param collect + env verify + Phase 0 dispatch (other phases stub for future subprojects)"
```

---

### 任务 14：编写 README.md

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/README.md`

套件说明：定位、文件结构、安装、触发方式、当前实现状态（子项目 A 完成）。

- [ ] **步骤 1：编写 README.md**

写入文件，含：

**§1 项目定位** — 一句话概述 + 设计哲学链接到 `/root/docs/superpowers/specs/2026-06-28-sourcecpt-design-philosophy.md`

**§2 当前实现状态**

```markdown
| 子项目 | 实现状态 |
|--------|---------|
| A 基础设施+Phase 0 | ✅ 完成 |
| B 侦察 Phase 1a/1b | 待实现 |
| C Phase 2a 漏洞匹配 | 待实现 |
| D Phase 3+4 推理引擎 | 待实现 |
| E Phase 5+6 链与挑战 | 待实现 |
| F Phase 7+8+9 交付进化 | 待实现 |
| G 可选合规插件 | 待实现 |
| H 可选持续审计+Phase 1c | 待实现 |
```

**§3 目录结构** — 列出当前实现的文件树

**§4 前提条件** — opencode / 本地源码项目路径

**§5 触发方式**

```
在 opencode 对话中触发:
  使用 $sourcecpt 审计 /path/to/project
  使用 $sourcecpt 对 /path/to/repo 做 fast 模式审计, scope=java
  使用 $sourcecpt 增量审计 /path/to/repo, changed-since=main
```

**§6 参数说明** — 表列

**§7 设计文档** — 链接到 `docs/superpowers/specs/2026-06-28-sourcecpt-design.md` 和 `*-design-philosophy.md`

**§8 安全边界** — 仅授权审计

- [ ] **步骤 2：Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/README.md && git commit -m "docs(sourcecpt): add README — project positioning + impl status + install + trigger"
```

---

### 任务 15：准备测试项目并端到端验证

**文件：**
- 创建：`/tmp/opencode/sourcecpt-test-project/` 内含小 Java + Python 测试项目

准备一个最小可测项目验证 Phase 0 跑得通、产出符合 schema。

- [ ] **步骤 1：创建测试项目**

```bash
mkdir -p /tmp/opencode/sourcecpt-test-project/src/main/java/com/foo/controller
mkdir -p /tmp/opencode/sourcecpt-test-project/src/main/java/com/foo/dao
mkdir -p /tmp/opencode/sourcecpt-test-project/src/test/java
mkdir -p /tmp/opencode/sourcecpt-test-project/src/main/python
```

创建测试文件:

`src/main/java/com/foo/controller/UserController.java`:
```java
package com.foo.controller;

import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id) {
        return userService.queryById(id);
    }
}
```

`src/main/java/com/foo/dao/UserDao.java`:
```java
package com.foo.dao;

import java.sql.*;

public class UserDao {
    public String findById(Long id) throws SQLException {
        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost/db", "admin", "password123");
        PreparedStatement st = conn.prepareStatement("SELECT * FROM users WHERE id=" + id);
        ResultSet rs = st.executeQuery();
        return rs.getString(1);
    }
}
```

`src/main/python/api.py`:
```python
from flask import Flask, request
import subprocess

app = Flask(__name__)

@app.route("/exec")
def exec_cmd():
    cmd = request.args.get("cmd")
    return subprocess.check_output(cmd, shell=True)
```

`src/test/java/ShouldBeSkippedTest.java`:
```java
// 测试代码应被跳过
```

- [ ] **步骤 2：在 opencode 中验证调用**

新开 opencode 会话，输入:
```
使用 $source-map 审计 /tmp/opencode/sourcecpt-test-project
```
或通过 Pipeline 入口:
```
使用 $sourcecpt 审计 /tmp/opencode/sourcecpt-test-project, mode=fast
```

观察执行流程:
- Phase 0 应建立工作目录在 `/tmp/sourcecpt-{session_id}/`
- 应产出 manifest.json 含 3 个 source + 1 skipped (test)
- entries.json 应含至少 2 个入口 (UserController GET + Flask /exec)
- sinks.json 应含至少 3 个 sink (SQL 拼接 + Runtime/subprocess + 硬编码口令)
- modules.json 应有 1-2 个模块
- progress.json phase0 = complete

- [ ] **步骤 3：校验产出符合 schema**

人工对照 OUTPUT_STANDARD.md §3 的 JSON schema 校验:
- manifest.json 每条含 path/lang/loc/sha256/state/phase0_category/skip_reason
- entries.json 每条含 node_type/entry_type/path/line/signature/param_tree/module/authz_state/audit_state
- sinks.json 每条含 node_type/sink_type/owasp/cwe/path/line/code_snippet/audit_state
- progress.json phase0 = "complete"

**反幻觉抽查**:
- 抽 entries.json 中@RestController行号是否与测试文件实际行号一致(反幻觉#2)
- 抽 sinks.json 中 code_snippet 是否是测试文件中的真实代码片段

- [ ] **步骤 4：发现问题修正 SKILL.md**

若产出不符合 schema 或反幻觉违反:
- 修正对应的 SKILL.md 指令
- 重新跑验证
- 直至通过

- [ ] **步骤 5：Commit 验证结果**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/ && git commit -m "test(sourcecpt): verify Phase 0 end-to-end with test project — manifest/entries/sinks/modules produced"
```

---

### 任务 16：自检与交接

- [ ] **步骤 1：规格覆盖度自查**

对照设计规格 Phase 0 部分（§4 中"Phase 0 — 工程地图"）逐项:
- MUST 输入 4 项参数? ✅
- 7 步骤核心工作流? ✅
- 5 项产出物? ✅
- 4 项反幻觉约束? ✅
- 3 项 300w 规模验收门? ✅
- 边界声明"不审漏洞"? ✅

对照 shared 7 个规范文件:
- OUTPUT_STANDARD? ✅
- ANTI_HALLUCINATION? ✅
- SRC_ACCESS? ✅
- OFFSET_RATING? ✅
- CHAIN_STANDARD? ✅
- LOOP_POLICY? ✅
- TRACKING_SHEET? ✅
- BASELINE_DIFF? ✅

知识库 2 个:
- entry-types.md? ✅
- sink-types.md? ✅

- [ ] **步骤 2：占位符扫描**

检查所有写出的 SKILL.md 与 shared 规范:
- 无 TODO/TBD/XX/待定字样?
- 无未展开的"类似任务 N"或"后续实现"?
- 反幻觉约束文本完整无 `...` 省略?

修正任何发现。

- [ ] **步骤 3：类型一致性**

- entries.json 字段名 vs OUTPUT_STANDARD 定义一致?
- sinks.json 字段名 vs OUTPUT_STANDARD 定义一致?
- manifest.json 字段名 vs OUTPUT_STANDARD 定义一致?
- 引用的 entry-types/sink-types grep 模式与 source-map SKILL.md 中实际 grep 使用的模式一致?

- [ ] **步骤 4：最终 Commit**

```bash
cd ~/.config/opencode && git add skills/sourcecpt/ && git commit -m "chore(sourcecpt): self-check pass — spec coverage 100%, no placeholders, type consistency verified"
```

- [ ] **步骤 5：子项目 A 完成声明**

输出至此子项目 A 全部交付. 后续 7 个子项目计划依次编写.