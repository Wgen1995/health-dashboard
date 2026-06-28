# 输出格式标准（OUTPUT_STANDARD）

> 适用范围：所有 SourceCPT 子技能在向工作目录写数据时必须遵循本标准。
> 约束力：**强制**。违反将导致 QA 结构校验失败、断点续传失败或下游 Phase 读取异常。
> 节点与边的 schema 按白盒语义重新设计——白盒污点追踪的数据模型与运行时侦察差异极大, 不照搬任何黑盒套件结构。

---

## 1. 文件命名规范

### 1.1 通用规则

- 文件名一律 **snake_case**：仅小写字母、数字、下划线；禁止空格（`cross-ref/` 为约定例外）、中文、特殊符号
- 所有产出文件落点严格限定在 **`{session_dir}/`** 工作目录树内，禁止写入会话目录以外
- 同一类产出的多种格式同名同根：`xxx.md` 与 `xxx.json` 必须共用同名
- 进度/任务/配置类文件位于工作目录根：`session_config.json`、`progress.json`、`manifest.json`、`audit_log.json`、`dispatch_plan.json`
- 临时文件只能写入 `{session_dir}/tmp/`，不得污染 `evidence/`、`knowledge_graph/`、`reports/`

### 1.2 证据目录按 Phase 分子目录

| 子目录 | 责任 Phase | 内容 |
|--------|-----------|------|
| `evidence/phase0/raw/` | 0 | 工程地图原始 grep/bash 输出 |
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
| `evidence/phase6_loop/` | 6 | 对抗 loop 状态文件 |
| `evidence/phase7/` | 7 | POC 包 |
| `evidence/phase9/` | 9 | 进化报告 |

### 1.3 知识图谱文件

`knowledge_graph/` 下分 `nodes/` 与 `edges/`：

#### 节点 nodes/*.json（JSON 数组，每文件一类）

| 文件 | 节点类型 | 责任 Phase | 是否可选 |
|------|---------|-----------|---------|
| `entries.json` | entry | 0/1a | 必需 |
| `sinks.json` | sink | 0 | 必需 |
| `modules.json` | module | 0 | 必需 |
| `authz_configs.json` | authz_config | 1b | 必需 |
| `constraints.json` | constraint | 1c | 可选(启用1c) |
| `compliance_violations.json` | compliance_violation | 2b | 可选(--with-compliance) |
| `vuln_candidates.json` | vuln_candidate | 2a/3/4 | 必需 |
| `pattern_matches.json` | pattern_match | 4a | 必需 |
| `chains.json` | chain | 5 | 必需 |
| `verified_vulns.json` | verified_vuln | 6 | 必需 |

#### 边 edges/*.json（JSON 数组，每文件一类）

| 文件 | 边类型 | 责任 Phase | 是否可选 |
|------|-------|-----------|---------|
| `calls_module.json` | calls_module | 0 | 必需 |
| `entry_in_module.json` | entry_in_module | 1a | 必需 |
| `entry_calls.json` | entry_calls | 1a | 必需 |
| `entry_covered_by.json` | entry_covered_by | 1b | 必需 |
| `authz_deviation.json` | authz_deviation | 1b | 必需 |
| `data_flow.json` | data_flow | 2a | 必需 |
| `cross_ref.json` | cross_ref | 3 | 必需 |
| `upgrade.json` | upgrade | 3 | 必需 |
| `chain_steps.json` | chain_steps | 5 | 必需 |
| `sibling_of.json` | sibling_of | 5/6 | 必需 |
| `business_flow.json` | business_flow | 1c | 可选(启用1c) |
| `constraint_to_code.json` | constraint_to_code | 1c | 可选(启用1c) |

### 1.4 报告文件

`reports/` 下报告按 §TRACKING_SHEET 配套产出 `.md + .json + 可选 .sarif`：

| 文件 | 责任 Phase | 是否可选 |
|------|-----------|---------|
| `pentest_report.md + .json + .sarif` | 8d | 必需 |
| `compliance_report.md + .json` | 8c | 可选(--with-compliance) |
| `chain_report.md + .json` | 8b | 必需 |
| `patch_suggestion.md + .json` | 8d | 必需 |
| `qa_report.md + .json` | 8d | 必需 |
| `issue_tracking.csv + .xlsx` | 8d | 必需 |
| `budget_exceeded.md` | 任意（触发硬上限时） | 仅触发时产 |

### 1.5 基线目录

`baseline/{date}/` 存历史审计核心产物供增量审计对比；`baseline/last_insights.md` `hit_count.json` `stale_patterns.json` `suite_version.json` 由 Phase 9 维护。

---

## 2. 每个 Phase 的 MUST 输出清单

> 下列文件为对应 Phase 完成判定的硬性检查项。任一缺失或为空即视为该 Phase 未完成，将触发重新执行。

### Phase 0 — 工程地图

**MUST 输入**：`project-path`、`scope`、`mode`

**MUST 输出**：
- `session_config.json`（含 language_fingerprint / total_loc / dispatch_strategy / budget）
- `manifest.json`（非空，全文件清单含 path/lang/loc/sha256/state/phase0_category/skip_reason）
- `dispatch_plan.json`（含 strategy / batches / max_parallel）
- `audit_log.json`（初值空数组 `[]`，Phase 0 完成后追加条目）
- `progress.json`（phases.phase0 = "complete"）
- `knowledge_graph/nodes/entries.json`
- `knowledge_graph/nodes/sinks.json`
- `knowledge_graph/nodes/modules.json`
- `knowledge_graph/edges/calls_module.json`
- `evidence/phase0/raw/`（非空，每个 bash/grep 输出留底）

### Phase 1a/1b/1c/2a/2b/3/4a/4b/5/6/7/8a-8d/9

> MUST 输出由对应子项目计划定义。本子项目 A 仅支持 Phase 0。

---

## 3. JSON Schema 定义

### 3.1 session_config.json

```json
{
  "session_id": "uuid-字符串",
  "created_at": "ISO8601 时间戳",
  "target": {
    "project_path": "/绝对路径",
    "scope": "java,python",
    "mode": "full|fast|increment",
    "context_path": null,
    "with_compliance": false
  },
  "language_fingerprint": {
    "java": {"file_count": 234, "loc": 89000},
    "python": {"file_count": 56, "loc": 12000}
  },
  "total_loc": 101000,
  "dispatch_strategy": "single-batch|module-batched|subdir-dispatch|increment",
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

状态取值：`pending` → `in_progress` → `complete` | `failed`

### 3.3 manifest.json（JSON 数组）

每条目结构：

```json
{
  "path": "src/com/foo/BarController.java",
  "lang": "java",
  "loc": 234,
  "sha256": "abc123def456...",
  "state": "[ ]",
  "phase0_category": "source",
  "skip_reason": null,
  "audit_owner": null
}
```

**字段说明**：
- `path`：相对 project-path 的路径
- `lang`：编程语言（java/go/python/js/ts/其他）
- `loc`：行数（wc -l）
- `sha256`：文件 SHA256 哈希
- `state`：五态初值 `[ ]`，由后续 Phase 改写
- `phase0_category`：`source` / `resource` / `test` / `third_party` / `build_artifact` / `config`
- `skip_reason`：若 state=`[-]`，必填跳过原因字符串
- `audit_owner`：由后续 Phase 改写，标记哪个 WU 审计过

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

`batches` 数组在 `single-batch` 模式下仅 1 条目；`increment` 模式下含 git diff + 调用图邻居文件清单。

### 3.5 audit_log.json（JSON 数组）

每条目：

```json
{
  "item_id": "src/com/foo/BarController.java",
  "phase": "0",
  "verdict": "indexed",
  "timestamp": "ISO8601",
  "wu_id": "phase0-batch-1"
}
```

每 WU 完成立即追加一条，**不依赖会话上下文维持**（反幻觉#2、SRC_ACCESS §5 第一层兜底）。

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
    {"name": "id", "type": "Long", "source": "path|query|body|header|file", "validated": "confirmed|missing|unknown"}
  ],
  "module": "user-service",
  "authz_state": "unknown|secured|unsecured|partial",
  "authz_ref": null,
  "audit_state": "[ ]"
}
```

**字段演进**：Phase 0 写 `entry_type/path/line/audit_state`，`signature/param_tree/module` 初值为空或 "unknown" 由 Phase 1a 补齐；`authz_state` 由 Phase 1b 补齐。

#### sinks.json 节点

```json
{
  "node_type": "sink",
  "sink_type": "sql_splice|cmd_exec|deser_native|deser_fastjson|xxe|ssrf|path_traversal|weak_hash|weak_crypto|hardcoded_cred|weak_random|ldap_injection|nosql_injection|xpath_injection|spel_ognl_el|ssti|xslt|crlf_header|mail_header|log4shell_jndi|debug_endpoint|info_leak|sbom_cve|missing_authz|idor|jndi_injection|dns_resolve|file_upload|open_redirect|xss_output",
  "owasp": "A01|A02|A03|A04|A05|A06|A07|A08|A09|A10",
  "cwe": "CWE-89|CWE-78|...",
  "path": "src/dao/UserDao.java",
  "line": 102,
  "code_snippet": "PreparedStatement st = conn.prepareStatement(\"SELECT * FROM users WHERE id=\" + userId)",
  "audit_state": "[ ]"
}
```

`code_snippet` 是 sink 行原文（截断到 ≤200 字），不是周边代码；周边留待 Phase 2a 污点追踪再读。

#### modules.json 节点

```json
{
  "node_type": "module",
  "module_id": "user-service",
  "path": "src/user-service/",
  "loc": 280000,
  "lang": "java",
  "entry_count": 0,
  "sink_count": 0
}
```

`entry_count`/`sink_count` 初值 0，Phase 0 结束时由 entries/sinks 聚合回填。

### 3.7 知识图谱边 schemas

#### calls_module.json 边

```json
{
  "edge_type": "calls_module",
  "from": "user-service",
  "to": "auth-service",
  "evidence": "import com.foo.auth.*; in src/user-service/UserController.java"
}
```

`evidence` 必填，引 grep/import 证据原文（反幻觉#2）。

---

## 4. 五态标记闭环

| 标记 | 含义 | 合规语义 | 漏洞语义 |
|------|------|---------|---------|
| `[x]` | 已确认 | fail 确认 | VULN confirmed |
| `[?]` | 疑似 | 可疑发现需深审 | SUSPENDED 待人工 |
| `[-]` | 不适用 | na / pass | disproved 已证伪 |
| `[!]` | 环境干扰 | 命令失败 | 被安全机制阻断 / 资源耗尽 |
| `[ ]` | 未检查 | **必须消灭** | **必须消灭** |

**强制规则**：QA 第一层校验扫描 manifest / 候选 / 漏洞终态 中所有 `state` / `audit_state` 字段，残留 `[ ]` 即拒收报告。所有 `[ ]` 必须在对应 Phase 结束前升级为 `[x]/[?]/[-]/[!]` 之一。

**五态映射**：(对照 GenCPT 五态概念继承，由证据决定状态由位置)

---

## 5. WU 落盘规则（防会话压缩丢数据）

- 每个 WU（工作单元）完成后**立即**写盘 audit_log.json（追加 JSON 数组条目）
- WU 产出的 evidence 文件立即写 `evidence/phase{N}/` 对应子目录
- WU 产出的图谱节点/边立即写 `knowledge_graph/nodes|edges/` 对应 JSON 文件
- **不依赖子代理会话上下文维持状态**——下个 WU 输入明确指定文件路径，不靠"上轮记忆"
- 子代理崩溃不影响已完成 WU 结果（落盘即持久）

设计哲学："内存不可信，磁盘可信"（SRC_ACCESS §5）。

---

## 6. 反幻觉引用

本规范被以下反幻觉条目引用：

| 条目 | 引用点 |
|------|-------|
| #2 不准伪造代码行号/输出 | §3.6 节点 path/line/code_snippet 必来自真实文件 |
| #3 无证据不写确认态 | §4 五态中 `[x]` 必须有证据 |
| #6 占位符必替换 | §3 各 schema 中 actual 值必填 |
| #16 每函数必标审计结果 | manifest 中每 source 文件最终 state 必≠`[ ]` |
| #34 已扫兄弟必留 audit_state | sibling_scan 后被扫节点 audit_state 必改 |
| #44 每产出必含五段证据链 | verified_vulns 节点 schema 在子项目 E 定义 |

违反即触发对应约束条目（见 ANTI_HALLUCINATION.md）。