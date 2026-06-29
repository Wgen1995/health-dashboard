---
name: source-map
description: >
  白盒源码审计第一步: 工程地图。对目标项目建立全文件清单 manifest、语言指纹、
  入口面索引(13通道)、sink 索引(28类)、模块级调用图骨架、智能派发策略。
  300w+ 行项目必须先建立可审计工作集, 否则后续阶段无覆盖率可言。
  使用场景: 审计启动第一步, 任何 mode 都必须执行。
  不使用场景: 已有完整 manifest.json 的复检会话(--skip-phase0 跳过)。
---

# source-map — Phase 0 工程地图

本 SKILL 是白盒源码审计的第一步, 为后续 Phase 1a-9 建立可审计工作集。Phase 0 **不审漏洞**, 只建地图: 全文件清单 manifest / 入口面索引 / sink 索引 / 模块骨架 / 智能派发策略。

---

## MUST 输入

| 名称 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `project-path` | string | **是** | — | 本地源码根目录绝对路径 |
| `scope` | enum | 否 | `all` | `java`/`go`/`python`/`js`/`ts`/`all`(逗号多选) 或 `module=path` 或 `changed` 或 `owasp-a1..a10` |
| `mode` | enum | 否 | `full` | `fast` / `full` / `increment` |
| `changed-since` | string | 否 | — | increment 模式的 git ref(分支/tag/commit), 仅 mode=increment 必填 |
| `session-dir` | string | 否 | 自动 | 由 Pipeline 入口传入, 缺失时自建 `/tmp/sourcecpt-{uuid}/` |

独立运行示例: `$source-map --project-path /abs/repo --scope java --mode full`
通过 Pipeline 调度时本 SKILL 仅接收 session-dir 与 project-path。

---

## 引用共享规范

执行前必须读取以下规范(按需读，不全量加载):

| 规范文件 | 用途 |
|---------|------|
| `skills/shared/SRC_ACCESS.md` | 文件读取规范 / 分块策略 / 临时方法级图谱 / 防会话压缩丢数据三层兜底 |
| `skills/shared/OUTPUT_STANDARD.md` | 输出格式 / JSON schema / 五态标记 / WU 落盘规则 |
| `skills/shared/OFFSET_RATING.md` | 审计深度 T0-T3 分层（本 Phase 主要用 T0 + 局部 T1）|
| `knowledge-bases/entry-types.md` | 13 入口面 grep 模式定义 |
| `knowledge-bases/sink-types.md` | 28 sink 类 grep 模式定义 |

---

## 反幻觉约束本 Phase 专属

Phase 0 受以下反幻觉约束（详见 ANTI_HALLUCINATION.md）:

| # | 约束 |
|---|------|
| #1 | 不准凭记忆出命令——必按 entry-types/sink-types 的 grep 模式执行 bash |
| #2 | 不准伪造——entry/sink 节点 path/line 必来自实际 grep 输出 |
| #6 | 占位符必替换——节点字段不许残留 `{{}}` 或待定, 不知填 "unknown" |
| #9 | manifest 必 Glob/bash 实列, 禁凭记忆列文件 |

---

## 核心工作流

### 步骤 1: 语言指纹识别 [T0]

按 scope 列文件统计, scope=all 时取文件数 Top3 语言作为 effective_scope。

#### 1.1 按 scope 限定语言统计

**跨平台通用方式**（优先用 opencode 内置 Glob, 在 Glob 不便时调系统 shell）:

用 Glob 列文件、用 Grep 计数（opencode 内置跨平台）:

| 维度 | opencode 原生方式 |
|------|------------------|
| Java 文件数 | Glob pattern: `**/*.java`, 然后 LLM 排除 `*/test/*`、`*/target/*`、`*/build/*` 路径 |
| Go 文件数 | Glob `**/*.go`, 排除 `*/vendor/*` |
| Python 文件数 | Glob `**/*.py`, 排除 `*/__pycache__/*`、`*/.venv/*` |
| JS/TS 文件数 | Glob `**/*.{js,ts,tsx}`, 排除 `*/node_modules/*` |
| 总行数 | 对 Glob 结果逐文件 Read 或调本地 shell（跨平台见 SRC_ACCESS §1.2） |

**LLM 执行步骤**:
1. 调 opencode `Glob` 列出每语言文件清单（cross-platform 内置工具）
2. 在 LLM 内存过滤掉 `test/`、`target/`、`node_modules/` 等目录（用 Glob 模式或过滤返回结果）
3. 算总行数时如文件数不大（≤1000）用 Read 逐文件读 + 数行；若文件数大（>1000）调系统 shell 跨平台命令:

**Linux/macOS/WSL/Git Bash**:
```bash
find {project-path} -type f -name "*.java" -not -path "*/test/*" -not -path "*/target/*" -not -path "*/build/*" | wc -l
```

**Windows PowerShell**:
```powershell
(Get-ChildItem -Path {project-path} -Filter *.java -Recurse | Where-Object { $_.FullName -notmatch '/test/' -and $_.FullName -notmatch '/target/' -and $_.FullName -notmatch '/build/' }).Count
```

LLM 注意: 先检测平台（试 `$PSVersionTable` 不报错是 PowerShell, 否则 bash）, 然后选对应风格。**不许硬死一种 shell**。

#### 1.2 写 session_config.json language_fingerprint

```json
"language_fingerprint": {
  "java": {"file_count": 1342, "loc": 202639},
  "python": {"file_count": 56, "loc": 12000}
},
"total_loc": 214639
```

每个语言必先写入 session_config.json 后才进入下一步。

---

### 步骤 2: 全文件清单 manifest 生成 [T0]

#### 2.1 执行 Glob/bash 列全部源码文件

按 scope 语言 find 列全文件。完整跳过规则:

| 目录/文件 | skip 原因 | state 初值 | phase0_category |
|---------|----------|-----------|----------------|
| `*/test/*` `*/tests/*` `src/test/` | 测试代码 | `[-]` | `test` |
| `*/node_modules/*` `*/vendor/*` `*/third_party/*` `*/lib/*` (三方) | 三方库 | `[-]` | `third_party` |
| `*.min.js` `*.map` `*/dist/*` `*/build/*` `*/target/*` | 构建产物 | `[-]` | `build_artifact` |
| `*.properties` `*.yml` `*.yaml` `*.xml`(无代码) `*.json`(配置类) | 资源/配置 | `[ ]` | `resource` |
| `*.md` `*.txt` `docs/` `*/README/*` | 文档 | `[-]` | `doc` |
| `*.sh` `*.py`(脚本) | 脚本(Phase 0 类 entry-script 识别) | `[ ]` | `script` |
| 其他源码(按扩展名匹配 scope) | 正常源码 | `[ ]` | `source` |

#### 2.2 每文件计算 sha256 + loc

**跨平台方式（见 SRC_ACCESS §1.2）**:

LLM 检测平台后选对应命令:

**Linux/macOS/WSL/Git Bash**:
```bash
sha256sum {file}  # 取输出首列
sha256sum {file} | cut -d' ' -f1
wc -l < {file}
```

**Windows PowerShell**:
```powershell
(Get-FileHash -Algorithm SHA256 {file}).Hash
(Get-Content {file} | Measure-Object -Line).Lines
```

每文件立即写盘 manifest.json 一条记录（不准攒 libr 一堆再写，防子代理崩溃全丢）。

#### 2.3 manifest.json 立即写盘

按 OUTPUT_STANDARD §3.3 schema 追加写入 manifest.json:

```json
[
  {"path": "src/main/java/com/foo/UserController.java", "lang":"java","loc":45,"sha256":"abc...","state":"[ ]","phase0_category":"source","skip_reason":null,"audit_owner":null},
  {"path": "src/test/java/SkipThisTest.java","lang":"java","loc":12,"sha256":"def...","state":"[-]","phase0_category":"test","skip_reason":"测试代码","audit_owner":null}
]
```

**立即写盘规则**（SRC_ACCESS §5）: 每条文件记录立写, 不许攒满上下文再写, 防止子代理崩溃导致全清单丢失。

**反幻觉#9**: 不许凭记忆列文件, 必 Glob/bash find 实际列出。

---

### 步骤 3: 入口面索引 [T0]

#### 3.1 加载 entry-types.md

必读 `knowledge-bases/entry-types.md` 获取 13 通道的 grep 模式（每语言多模式）。**不许凭记忆出 grep 模式**，必须 Read 该文件后按表执行。

#### 3.2 按 13 通道执行 Grep（opencode 内置跨平台）

优先用 **opencode 内置 Grep 工具**做模式匹配。每通道 pattern 取自 `knowledge-bases/entry-types.md` 第 4 列, path 取 `{project-path}`, include 按 scope 限定语言。

对每 entry_type 调一次 Grep:

```
[LLM 调用 opencode Grep]
Grep(
  pattern: <取自 entry-types.md entry_type=rest 行第 4 列 grep 模式>,
  path: {project-path},
  include: "*.java",*.go,*.py,*.js... (按 scope),
  output_mode: "content",   ← 显示匹配行内容
  -n: true                  ← 行号
)
```

若 opencode Grep 不支持复杂模式或文件过滤缺失, 使用系统 shell fallback:

**Linux/macOS/WSL/Git Bash**:
```bash
grep -rn -E "@RequestMapping|@GetMapping|@PostMapping|@PutMapping|@DeleteMapping|@RestController" \
  --include="*.java" \
  {project-path} 2>/dev/null | grep -v "/test/"
```

**Windows PowerShell**:
```powershell
Get-ChildItem -Path {project-path} -Filter *.java -Recurse | Where-Object { $_.FullName -notmatch '/test/' } | Select-String -Pattern "@RequestMapping|@GetMapping|@PostMapping|@PutMapping|@DeleteMapping|@RestController"
```

正则模式取自 `entry-types.md` 第 4 列, 已用**双兼容正则子集**（SRC_ACCESS §1.3）, bash 与 PowerShell 通用。

每条命中立即写 `knowledge_graph/nodes/entries.json`:

```json
{
  "node_type": "entry",
  "entry_type": "rest",
  "path": "src/main/java/com/foo/UserController.java",
  "line": 18,
  "signature": "unknown",
  "param_tree": [],
  "module": "unknown",
  "authz_state": "unknown",
  "authz_ref": null,
  "audit_state": "[ ]"
}
```

- 路径与行号取自 grep 原始输出（反幻觉#2）
- signature/param_tree/module/authz_state 由 Phase 1a/1b 补齐, 本 Phase 仅留 "unknown"（反幻觉#6: `unknown` 不是占位符, 明确意味"待补齐"）

#### 3.3 grep 原始输出留底 evidence

每通道 grep 的完整 stdout 写入 `evidence/phase0/raw/entries_{entry_type}.log`（带时间戳头）。这是反幻觉#1/#2 的审计 trail。

#### 3.4 0 命中报警

若某通道在全项目 0 命中, 必写入 `evidence/phase0/raw/entries_{entry_type}.log` 头标 "ZERO_HITS_WARRANT_REVIEW" 作为 QA 警示。0 命中可能是 grep 模式漏覆盖或漏面, 不许"未发现即安全"。

---

### 步骤 4: sink 索引 [T0]

#### 4.1 加载 sink-types.md

必读 `knowledge-bases/sink-types.md` 获取 28 类 sink 的 grep 模式。**不许凭记忆出 grep 模式**, 必须 Read 该文件后按表执行。

#### 4.2 按 28 类执行 Grep（opencode 内置跨平台）

优先用 opencode 内置 Grep 工具按 sink-types.md 第 4 列模式扫描。若需 fallback 用系统 shell 跨平台命令（见 SRC_ACCESS §1.2）。

示例 - sink_type=sql_splice 在 Java:

**Linux/macOS/WSL/Git Bash**:
```bash
grep -rn -E 'prepareStatement\("[^"]*".*\+|createNativeQuery\(|fmt\.Sprintf\("SELECT' \
  --include="*.java" \
  {project-path} 2>/dev/null | grep -v "/test/"
```

**Windows PowerShell**:
```powershell
Get-ChildItem -Path {project-path} -Filter *.java -Recurse | Where-Object { $_.FullName -notmatch '/test/' } | Select-String -Pattern 'prepareStatement\("[^"]*".*\+|createNativeQuery\(|fmt\.Sprintf\("SELECT'
```

正则双兼容子集, 取自 sink-types.md (SRC_ACCESS §1.3).

每条命中立即写 `knowledge_graph/nodes/sinks.json`:

```json
{
  "node_type": "sink",
  "sink_type": "sql_splice",
  "owasp": "A01",
  "cwe": "CWE-89",
  "path": "src/main/java/com/foo/dao/UserDao.java",
  "line": 102,
  "code_snippet": "PreparedStatement st = conn.prepareStatement(\"SELECT * FROM users WHERE id=\" + userId);",
  "audit_state": "[ ]"
}
```

- `code_snippet` 是 sink 行原文（截断到 ≤200 字），不是周边代码
- `owasp`/`cwe` 取自 sink-types.md 表对应行（反幻觉#2）

#### 4.3 grep 原始输出留底

每类 sink 的完整 stdout 写入 `evidence/phase0/raw/sinks_{sink_type}.log`。

#### 4.4 0 命中报警

某 sink 类 0 命中即写 "ZERO_HITS_WARRANT_REVIEW" 头标, QA 警示。

---

### 步骤 5: 模块级调用图骨架 [T1]

**只建模块级**，不建方法级全图（SRC_ACCESS §4 临时方法级仅在 Phase 4a 触达 sink 时反向展开 5 跳用后弃）。

#### 5.1 顶层模块识别

按顶层目录切分模块, 每模块对应一个源码顶级目录。**跨平台**:

**opencode Glob 方式（优先）**:
```
Glob(pattern: "{project-path}/*/")  ← 列顶层目录
或 Glob(pattern: "{project-path}/src/main/java/com/*/{module}/")  ← Java 包结构
```

**Linux/macOS/WSL/Git Bash fallback**:
```bash
ls -d {project-path}/src/main/java/com/*/{module} 2>/dev/null   # Java 包结构
ls -d {project-path}/{module} 2>/dev/null                        # 通用顶层目录
```

**Windows PowerShell fallback**:
```powershell
Get-ChildItem -Path {project-path} -Directory   # 顶层目录
Get-ChildItem -Path {project-path}/src/main/java/com -Directory -Recurse -Depth 1
```

或按 manifest 中 path 的前 2-3 段聚类（LLM 内部逻辑）: 假设 manifest 中的 path 前缀决定模块归属。

#### 5.2 模块节点写盘

```json
{
  "node_type": "module",
  "module_id": "user-service",
  "path": "src/main/java/com/foo/user/",
  "loc": 280000,
  "lang": "java",
  "entry_count": 0,
  "sink_count": 0
}
```
`entry_count`/`sink_count` 初值 0, 步骤 6 聚合回填（Phase 0 结束前必填, 不许残留 0 - 若真为 0 则写 [!] zero 实情）

#### 5.3 模块间调用关系

跨模块引用关系提取。跨平台方法（优先用 opencode Grep + LLM 解析）:

**opencode 原生方式**（推荐）:
```
LLM 用 Grep 扫 "^import" 模式 path={project-path} include=*.java
得到每文件 import 清单, LLM 在内存中按模块聚类找跨模块 import, 写 calls_module 边
```

**Linux/macOS/WSL/Git Bash fallback**:
```bash
grep -rn "^import com\." --include="*.java" {project-path} 2>/dev/null | \
  awk -F: '{print $1" "$3}' | awk '{print $1, $3}' | \
  grep -v "test"
```

**Windows PowerShell fallback**:
```powershell
Get-ChildItem -Path {project-path} -Filter *.java -Recurse | Where-Object { $_.FullName -notmatch '/test/' } | Select-String -Pattern '^import com\.' | ForEach-Object { "$($_.Path):$($_.LineNumber): $($_.Line)" }
```

每跨模块 import 写一条 calls_module 边:

```json
{
  "edge_type": "calls_module",
  "from": "user-service",
  "to": "auth-service",
  "evidence": "import com.foo.auth.*; in src/main/java/com/foo/user/UserController.java"
}
```

evidence 必填，含 grep 或 Grep 工具原始输出（反幻觉#2）。

---

### 步骤 6: 智能派发策略 [T0]

按 total_loc 决定 strategy:

| total_loc | strategy | batches 结构 |
|-----------|----------|------------|
| ≤100000 (10w 行) | `single-batch` | 1 batch 包含全 manifest source 文件 |
| 100000-500000 (10-50w) | `module-batched` | 按 modules.json 的 module_id 分批, 每批 ≤30w 行 |
| >500000 (>50w) | `subdir-dispatch` | 递归切分大模块至每子代理负载 ≤30w 行 |
| mode=increment | `increment` | 仅 `git diff --name-only {changed-since}..HEAD` + 调用图邻居 5%-10% 文件 |

#### 6.1 dispatch_plan.json 写盘

```json
{
  "strategy": "single-batch",
  "total_loc": 202639,
  "batches": [
    {"id": "B1", "modules": "all", "loc": 202639, "owner_phase2": "todo"}
  ],
  "max_parallel": 3
}
```

#### 6.2 increment 模式特殊处理

**跨平台**（git 在所有平台通用）:

**Linux/macOS/WSL/Git Bash**:
```bash
git -C {project-path} diff --name-only {changed-since}..HEAD 2>/dev/null
```

**Windows PowerShell**:
```powershell
git -C {project-path} diff --name-only {changed-since}..HEAD
```

git 命令在所有平台原生支持, 仅错误重定向差异（`2>/dev/null` vs 默认）。

取变更文件清单 + 沿 calls_module.json 边找调用图邻居（深度 2 模块级）。manifest 中未变更/非邻居文件 state 设 `[-]` skip_reason="increment 未变更"（但已审计基线除外, 见 BASELINE_DIFF.md）。

---

### 步骤 7: 写盘与图谱初始化 [T0]

#### 7.1 全部产出物落盘

按 OUTPUT_STANDARD §2 Phase 0 MUST 输出清单完整写盘:

- session_config.json（含 language_fingerprint + total_loc + dispatch_strategy + budget 见 OUTPUT_STANDARD §3.1）
- manifest.json
- dispatch_plan.json
- audit_log.json — 每文件审计记录追加, 每入口/sink 节点也追加条目:
  ```json
  [
    {"item_id": "src/com/foo/UserController.java", "phase": "0", "verdict": "indexed", "timestamp": "ISO8601", "wu_id": "phase0-1"},
    {"item_id": "entry:src/com/foo/UserController.java:18", "phase": "0", "verdict": "entry_rest_added", "timestamp": "...", "wu_id": "phase0-2"},
    ...
  ]
  ```
- progress.json — `phases.phase0 = "complete"`, `current_phase = "phase1a"`, `last_updated = ISO8601`
- knowledge_graph/nodes/entries.json + sinks.json + modules.json
- knowledge_graph/edges/calls_module.json
- evidence/phase0/raw/ 全部 grep 原始输出留底

#### 7.2 聚合模块 entry_count/sink_count

Phase 0 结束前, 遍历 entries.json/sinks.json 按 module 字段聚合每模块 entry_count/sink_count, 更新 modules.json 同字段。

若模块 entry_count=0 或 sink_count=0, 不许置之不问——必标 `[!]` 留 QA 审查原因（可能是模块无源码, 也可能 Phase 0 grep 漏面）。

---

## 产出物完整清单

```
{session_dir}/
├── session_config.json           # §3.1 schema
├── manifest.json                 # §3.3 schema
├── dispatch_plan.json            # §3.4 schema
├── audit_log.json                # §3.5 schema
├── progress.json                 # §3.2 schema, phase0=complete
├── knowledge_graph/
│   ├── nodes/
│   │   ├── entries.json          # §3.6 entries schema
│   │   ├── sinks.json            # §3.6 sinks schema
│   │   └── modules.json          # §3.6 modules schema
│   └── edges/
│       └── calls_module.json     # §3.7 schema
└── evidence/
    └── phase0/raw/
        ├── entries_{entry_type}.log   # 每 grep 输出留底
        └── sinks_{sink_type}.log       # 每 grep 输出留底
```

---

## 300w 规模验收门

Phase 0 完成必过 3 项验收（如不过即不标 complete, 触发重跑）:

| 验收门 | 阈值 | 不通过后果 |
|--------|------|----------|
| manifest 覆盖率 | 100% source 文件入清单（含已跳过必须有 skip_reason） | manifest 重生成 |
| 13 通道至少 N>0 命中 | 每通道至少识别 1 个 entry（0 必 QA 报警） | QA 审查 grep 模式或确认项目真的无该通道 |
| 28 sink 类每类 N>0 命中 | 每类至少识别 1 个 sink（0 必 QA 报警） | QA 审查 grep 模式或确认项目真的无该 sink |
| 派发颗粒 | 每子代理负载 ≤30w 行 | 调整 strategy 或拆细 batches |

**所有 [ ] 残留** manifest 中 state=`[ ]` 是正常初始态（等后续 Phase 改),但 audit_log 必须有对应 entry 记录该文件已 indexed。`audit_log` 无对应条目 = 该文件实际未审 = manifest 不算覆盖。

---

## 边界声明

Phase 0 **不审漏洞**——识别 sink 不代表漏洞, 识别"无 sanitizer"也不代表漏洞（无 sink 周边代码）。Phase 0 仅建工作集:
- 入口面: 是后续 Phase 1a 补齐签名/Phase 4a 污点追踪 source
- sink: 是后续 Phase 2a/4a 执行污点追踪的目标
- 模块骨架: 是后续跨模块路径可达性判断基础

Phase 0 产出被后续 Phase 1a/1b/2a/3/4a/5 引用, 通过 `knowledge_graph/nodes/` 与 `edges/` 齐。
本 Phase 不分析代码语义、不做证据链判定、不产出 VULN/CLUE。