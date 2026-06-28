# 文件读取与按需加载规范（SRC_ACCESS）

> 适用范围：所有 Phase 在读源码时必须遵循本规范。
> 约束力：**强制**。
> 这是白盒专属规范——黑盒运行时侦察的 SSH 命令规范全无借鉴价值，直接按白盒语义自构。

---

## §1 文件读取工具约定

| 工具 | 用途 | 输入约定 |
|------|------|---------|
| Glob | 列文件路径（按模式） | 模式必含语言扩展名 `*.java/*.go/*.{ts,tsx}`，相对 project-path |
| bash | 执行 grep/find/wc/sha256sum/git diff | 命令必含 `--include` 限定语言，`-not -path` 排除跳过目录 |
| Read | 读单文件内容 | 输入必须是绝对路径，offset/limit 用于分块 |
| Write | 写磁盘产出 | 输入必须是 `{session_dir}/` 工作目录内路径 |
| Grep | 内容搜索（如已知文件类型） | 模式 + include 限定；若 grep 工具有 -r 支持目录扫描 |

**强制要求**：
- Read 文件无编码时默认 UTF-8
- bash 命令在执行前必输出 echo "命令+时间戳" 留底到 evidence
- 所有路径使用绝对路径或相对 project-path 的相对路径，不许相对当前工作目录猜测

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