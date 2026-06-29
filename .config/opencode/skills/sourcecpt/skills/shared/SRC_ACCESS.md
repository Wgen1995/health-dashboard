# 文件读取与按需加载规范（SRC_ACCESS）

> 适用范围：所有 Phase 在读源码时必须遵循本规范。
> 约束力：**强制**。
> 这是白盒专属规范——黑盒运行时侦察的 SSH 命令规范全无借鉴价值，直接按白盒语义自构。

---

## §1 文件读取工具约定（跨平台）

### §1.1 工具优先级（跨平台优先用 opencode 原生工具）

SourceCPT 必须支持在 Linux / macOS / Windows 任一平台运行。为实现跨平台, **优先用 opencode 内置工具（Glob/Grep/Read/Write）, 系统 shell 命令仅在 opencode 工具不足以完成任务时调用**。

| 任务 | 优先工具 | 必要 fallback |
|------|---------|--------------|
| 列文件路径（按模式递归） | **Glob**（opencode 内置, 无平台差异） | — |
| 内容搜索（按正则递归） | **Grep**（opencode 内置, 无平台差异） | — |
| 读文件内容 | **Read**（opencode 内置） | — |
| 写产出文件 | **Write**（opencode 内置） | — |
| 算文件 SHA256 | — | **必须调系统 shell**（无 opencode 内置 hash 工具） |
| 算文件行数 | — | **必须调系统 shell**（无 opencode 内置 wc 工具） |
| git diff 文件清单（increment 模式） | — | **必须调系统 shell + git** |

### §1.2 系统 shell 调用约定（跨平台自适应）

LLM 执行系统 shell 时必**先检测平台**, 然后选对应命令。SKILL.md 中给出的命令示例默认以 bash 风格书写（便于 Linux/macOS/WSL/Git Bash 用户）, 但 LLM 在 Windows 平台运行时必改用 PowerShell 风格。

#### 跨平台命令映射表

| 任务 | bash 风格 (Linux/macOS/WSL/Git Bash) | PowerShell 风格 (Windows) |
|------|------------------------------------|--------------------------|
| 列文件（含排除） | `find {project-path} -type f -name "*.java" -not -path "*/test/*"` | `Get-ChildItem -Path {project-path} -Filter *.java -Recurse \| Where-Object { $_.FullName -notmatch '/test/' -and $_.FullName -notmatch '/target/' }` |
| 内容 grep | `grep -rn -E "pattern" --include=*.java {project-path}` | `Get-ChildItem -Path {project-path} -Filter *.java -Recurse \| Select-String -Pattern "pattern"` |
| 算文件 SHA256 | `sha256sum {file}` 或 macOS `shasum -a 256 {file}` | `(Get-FileHash -Algorithm SHA256 {file} \|[Select-Object -ExpandProperty Hash)` |
| 文件行数 | `wc -l < {file}` | `(Get-Content {file} \| Measure-Object -Line).Lines` |
| git diff 文件清单 | `git -C {project-path} diff --name-only {changed-since}..HEAD` | 同 bash 风格（git 在 Windows PowerShell 原生支持） |

#### 平台检测指令

LLM 必在调用系统 shell 前**先检测平台**:
- bash 环境检测: `uname` 或 `echo "$OS"` 输出含 Linux/Darwin/MSYS/MINGW 即用 bash 风格
- PowerShell 检测: `echo $PSVersionTable` 不报错即用 PowerShell 风格

或更简单: LLM 先试 `$PSVersionTable` 命令。不报错是 PowerShell, 报错（如 "command not found"）是 bash。然后选对应命令。

**重要**: 不许硬编码单一 shell。每条 SKILL.md 命令示例必默认是 bash 但提供 PowerShell 等价。LLM 执行时实时检测平台选对应。

### §1.3 跨平台正则兼容性

`entry-types.md` 与 `sink-types.md` 中的 grep 模式必用**双兼容正则子集**, 在 bash `grep -E` 与 PowerShell `Select-String` 下都能识别:

- 必用基础字符: `|` (or)、`[]` 字符集、`\.` 字面量、`(...)` 分组、`* + ? {n,m}` 量词
- **禁用** POSIX-only 特性: `\b` 边界、`\w` 字符类、`-P` PCRE 模式
- **禁用** POSIX 字符类: `[[:space:]]` 等, 改显式 `[ ]` 或 `[ \\t]`
- 复杂正则需要时拆成多 grep 子模式（避免平台差异）

---

## §2 分块策略（适配 300w 行项目）

### 2.1 按 500 行/块切

---

## §2 分块策略（适配 300w 行项目）

### 2.1 按 500 行/块切

| 文件行数 | 策略 |
|---------|------|
| <500 行 | 1 块，整文件送 LLM |
| 500-3000 行 | 按 `import / class / def / function / package` 等函数边界切，每块 ≤500 行，块间保 5 行重叠防跨块漏 |
| >3000 行 | 递归切分至每块 ≤500 行，递归深度 ≤3 |

### 2.2 分块输出 chunk_manifest.json

每文件 chunk 切分后写：

```json
{
  "file": "src/Service.java",
  "loc": 1240,
  "chunks": [
    {"chunk_id": "src/Service.java#1", "start_line": 1, "end_line": 487, "context": "Class header + field decl + m1()"},
    {"chunk_id": "src/Service.java#2", "start_line": 482, "end_line": 950, "context": "m2() + m3()"},
    {"chunk_id": "src/Service.java#3", "start_line": 945, "end_line": 1240, "context": "m4() + inner class"}
  ]
}
```

5 行重叠是关键，防违规跨块切断（如 `return` 语句和 sink 调用被切到上下两块）。

### 2.3 反幻觉引用

被 #16（每函数必标审计结果）、#17（跨块切断不许直接判 false）引用。

---

## §3 按需加载原则（防会话压缩丢数据）

| 维度 | 加载策略 |
|------|---------|
| 规则库(Phase 2a/4a) | 仅 sink_type/候选命中规则文件，不预载全 vuln-rules/ |
| 模式库(Phase 4a) | 仅条件触发表命中的模式 8 段 SKILL.md，不读全部 25+ 模式 |
| Phase 2b 分组 | 仅读该组规则文件(G1-*.md)，不读其他分组 |
| Phase 3 三库 | 仅读命中的 CHK-CAND/ATK-HYP 卡片，全库不超 50k token |
| Phase 6 假阳性库 | 仅读该候选涉及的假阳性场景，不读 full non-vuln-scenarios.md |
| 图谱(全 Phase) | 按 sink/entry/candidate 查询过滤，不全量载 knowledge_graph/ |
| 子代理 WU | 输入明确指定路径，不依赖会话记忆 |

**强制**：子代理 WU 输入必须明确：
- 该 WU 处理的 chunk_id / sink_id / candidate_id
- 需 Read 的文件绝对路径清单（≤5 文件/WU）
- 需查询的图谱节点类型（按 node_type 过滤）

**不许**：WU 输入说"分析 user-service 模块所有文件"——这是全量载入，违反按需加载。

---

## §4 临时方法级图谱展开

### 4.1 双层架构

| 层 | 存储 | 用途 |
|---|------|------|
| 持久层 | 模块级（knowledge_graph/nodes/modules.json + edges/calls_module.json） | 知模块边界、调用关系，适合架构级 |
| 临时层 | 方法级（随 sink 展开，用后弃） | 适合污点追踪精度 |

**不建全方法级持久图**——300w 行项目方法级图是 Gb 级，LLM 上下文无意义且耗尽 token。

### 4.2 临时展开时机

Phase 4a 污点追踪触达 sink 时临时反向递归建方法链：

1. 沿模块级边找到 path 可能跨模块
2. 进入 sink 所在模块时，Read 该模块相关文件（按 condition 触发，非全模块）
3. 临时反向递归建调用链（仅 sink 上游 5 跳）
4. 方法链仅本次 WU 内存，**完成即弃**
5. 不入 knowledge_graph 持久层

### 4.3 成本与收益

| 项 | 值 |
|----|----|
| 每次污点追踪额外读 | 5-10 文件（2-5k token） |
| 收益 | 方法级精度不丢，避免全图占内存 |
| 实现 | LLM 沿 path 边临时反向 Read 文件，不存为持久边 |

### 4.4 反幻觉引用

被 #18（不许凭 sink 直判）、#20（propagation 每跳必 Read）引用。临时方法级建立必须每跳 Read 证据，不许凭记忆推方法链。

---

## §5 防会话压缩丢数据三层兜底

| 层 | 机制 | 实现 |
|---|------|------|
| 1 | 任何 trace 立即写盘，不靠 LLM 上下文维持 | audit_log.json / sibling_scan_log.json / chunk_manifest.json 每条立即落盘 |
| 2 | 子代理下个 WU 不依赖上轮记忆，只重新读磁盘状态 | WU 输入明确路径，不靠"上轮我审了什么" |
| 3 | QA 第一层强制校验：audit_log + manifest + sibling_scan_log 完整性，无对应条目即拒收报告 | QA 第一层扫描该三类日志文件与 manifest 状态字段 |

**设计哲学**："内存不可信，磁盘可信"原则贯穿全 Pipeline。

---

## §6 反幻觉引用总览

本规范被以下约束条目引用：

| # | 引用位置 |
|---|---------|
| #1 | 不准凭记忆出攻击命令——本规范 §1 Read/Glob/bash 前置 |
| #2 | 不准伪造——本规范 §1 输出必引文件实际路径/行号 |
| #20 | propagation 每跳必 Read——本规范 §4.4 临时方法级 |
| #43 | 按需加载——本规范 §3 强制规则库/图谱不全量载入 |

违反即对应 Phase 内的 WU 重跑或候选降级。