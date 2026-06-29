---
name: entry-recon
description: >
  白盒源码审计第二步: 入口面侦察。基于 Phase 0 的入口索引, 逐入口补齐
  签名/参数树/所属模块/参数校验状态, 适配任意 Web 框架（Spring/Struts/
  Flask/Django/Express 等)与 RPC/MQ/WS/GraphQL/CRON/CLI/脚本/反序列化/
  文件/WebService/自定义协议/事件监听 13 类入口。把 grep 命中行升级为
  结构化入口面清单, 为后续 Phase 2a 污点追踪的 source 提供完整数据。
  SKILL 不为特定框架定制——LLM 必 Read 源码后按实际框架模式提取字段。
  使用场景: Phase 0 完成后, Phase 2a/1b 之前。
  不使用场景: entries.json 已完整梳理的复检（increment 模式可跳过未变更入口）。
---

# entry-recon — Phase 1a 入口面侦察

本 SKILL 把 Phase 0 的入口索引（entries.json 中 13 通道 grep 命中行）升级为结构化入口面清单。每个 entry 节点由 Phase 0 产出签名/参数树/模块/鉴权状态全部留 `unknown` 占位，本 Phase 通过 Read 源码补齐 signature/param_tree/module/validated_summary。`authz_state` 由 Phase 1b 补齐，本 Phase 不动。

SKILL **不为任何特定框架定制**——Spring/Struts/Flask/Express/Django 等所有主流框架通过通用规则识别: LLM Read 源码后按实际使用的框架模式提取对应字段。

---

## MUST 输入

| 名称 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `session-dir` | string | **是** | 含 Phase 0 产出的 knowledge_graph/nodes/entries.json |
| `project-path` | string | **是** | 本地源码根目录绝对路径，与 Phase 0 一致 |
| `mode` | enum | 否 | `fast/full/increment`——increment 时跳过未变更入口 |
| `scope` | enum | 否 | `java/go/python/js/ts/all` 限定语言 |

独立运行示例: `$entry-recon --session-dir /tmp/sourcecpt-001 --project-path /abs/repo`

---

## 引用共享规范

| 规范文件 | 用途 |
|---------|------|
| `skills/shared/SRC_ACCESS.md` | 文件 Read 每跳证据 / max_parallel=3 / 按需加载防会话压缩 |
| `skills/shared/OUTPUT_STANDARD.md` | entries.json 节点字段 schema / WU 落盘规则 |
| `skills/shared/OFFSET_RATING.md` | 本 Phase 主要 T0+T1 深度 |
| `skills/shared/ANTI_HALLUCINATION.md` | 全局 #1 不凭记忆 / #2 不伪造行号 / #6 不残留 unknown 必改 / #9 不凭记忆列文件 |

---

## 反幻觉约束本 Phase 专属

| # | 约束 | 强制力 |
|---|------|-------|
| #1 | 不准凭记忆出签名——必 Read 源码后填，pattern 取自实际代码 | WU 输入必含 Read 调用 |
| #2 | signature/path/line 必对得上实际文件真实行号 | evidence/phase1a/*.md 留底 |
| #6 | signature 字段必改写为实际值，不许残留 "unknown"（真不知需 evidence 说明） | QA 第一层扫残留 unknown 即让 Phase 重跑 |
| #9 | module 必基于 actual path 推断，对照 modules.json，不许凭记忆 | 对照 must exist |
| 本 Phase 内联#1.1 | 不许跳过任何 entry 节点，每 entry 必有补齐记录入 audit_log | QA 第一层必须有对应 phase=1a 条目 |

---

## MUST 输入与 MUST 输出（OUTPUT_STANDARD 协议）

**MUST 输入**:
- `session-dir` 含 Phase 0 产出的 `knowledge_graph/nodes/entries.json`（必非空）
- `project-path` 必可达（Read 可访问）

**MUST 输出**（Phase 1a 完成判定）:
- `knowledge_graph/nodes/entries.json` 每节点更新:
  - `signature` 必非 "unknown"，必含方法签名/路由模板/Action 名等可读描述
  - `param_tree` 必含 post-Read 后的参数结构（可为空数组但必须审过，不能是 unknown 信号）
  - `module` 必非 "unknown"，必对照 modules.json 中的 module_id 之一
  - 新增 `validated_summary` 字段: confirmed/missing/unknown/mixed 四态之一
  - `authz_state` 在本 Phase 仍是 `unknown`（Phase 1b 改）
  - `audit_state` 仍是 `[ ]` 待 Phase 2a/4
- `knowledge_graph/edges/entry_in_module.json` 每 entry 一条边（from=entry_ref, to=module_ref）
- `knowledge_graph/edges/entry_calls.json` 若能识别入口直接调用的方法则产边（深度 1，不展开调用图——Phase 4a 污点追踪时临时展开）
- `evidence/phase1a/{path-sanitized}.md` 每入口一 markdown 证据摘要（含 Read 源码片段留底）
- `audit_log.json` 追加每 entry 一条 `{item_id: entry:{path}:{line}, phase: "1a", verdict: "signature_filled", ...}`
- `progress.json` phases.phase1a -> "complete"

---

## 核心工作流

### 步骤 1：加载 Phase 0 entries.json

```bash
jq length {session-dir}/knowledge_graph/nodes/entries.json
```

读全数 N（典型项目 50-500, 300w 行项目 500-2000）。每 entry 节点一个 WU。**不许**全量载 entries.json 到子代理上下文——按 dispatch_plan.batches 分批, max_parallel=3（SRC_ACCESS §3 按需加载）。

每批 WU 输入含:
- 该批 entries 子集（一定 ≤30 条防止上下文爆炸）
- project-path 绝对路径
- 必 Read 的文件路径列表（每 entry 必至少 Read 1 文件）

### 步骤 2：逐入口补齐结构 [T0-T1]

每 entry WU 必 Read 源码后按入口类型补齐字段。**通用规则**: 每种 entry_type 在通用规则上根据源码实际使用的框架识别其入口声明方式, SKILL 不硬编码特定框架字段名。

#### WU 输入规范

```
[LLM 输入 - 单 entry 补齐 WU]
entry 节点:
  node_type: entry
  entry_type: rest|rpc|mq|ws|graphql|cron|cli|script|deser|file|webservice|custom_proto|event
  path: src/main/java/com/foo/UserController.java
  line: 28
  audit_state: [ ]

LLM 指令:
1. Read 文件 {path} 第 {line} 行 ±50 行（≤2k token, 不读完整文件, 仅本方法上下文）
2. 识别实际使用的框架模式（注解/继承/包名/注解类名等线索）
3. 按入口类型通用规则提取字段:

   **entry_type=rest** 通用规则:
     * 注解路由（Spring/JAX-RS/Spring Boot/Jersey）: 提取 HTTP 方法 + 路径模板 + 方法签名
     * 继承式 Action（Struts 2 ActionSupport,Struts 1 Action）: 提取 execute()/doExecute() 方法签名 + 返回类型
     * 函数式路由（Flask @app.route, Django @api_view, Express app.get）: 提取路径 + handler 函数签名
   **entry_type=rpc** 通用规则:
     * 提取服务名 / 方法名 / 入参类型 / 序列化协议（gRPC/Dubbo/Thrift）
   **entry_type=mq** 通用规则:
     * 提取 topic/queue 名 / 消费方法入参类型 / 反序列化方式
   **entry_type=cron** 通用规则:
     * 提取调度表达式（注解/属性）+ 执行方法入参类型
   **entry_type=deser** 通用规则:
     * 识别触发方式（JSON/XML/原生）+ 入参类型 + gadget 链可能性（if 反序列化已知库）
   **entry_type=cli/script** 通用规则:
     * 提取 main 签名 + 参数解析方式（args/argparse/cobra/commander）
   **entry_type=ws/graphql/webservice/custom_proto/file/event**:
     * 按各自模式的通用规则提取（每种通用识别, 不预设单一框架）

4. 输出 JSON 节点更新:
   {
     "node_type": "entry",
     "entry_type": "(原值)",
     "path": "(原值)",
     "line": "(原值)",
     "signature": "GET /api/users/{id} → getUser(Long id) → String",
     "param_tree": [
       {"name": "id", "type": "Long", "source": "path", "validated": "missing|confirmed|unknown"}
     ],
     "module": "(从 path 前 2-3 段推; 必对照 modules.json 中的 module_id)",
     "authz_state": "unknown",
     "authz_ref": null,
     "validated_summary": "missing|confirmed|unknown|mixed",
     "audit_state": "[ ]"
   }
```

### 步骤 3：参数树标注 [T1]

每 `param_tree` 项标 `validated`:
- `confirmed` — 有显式校验注解或手写校验代码（`@NotNull @Size @Pattern @Valid` 或手写 if-throw 等）
- `missing` — 入参直接传入 sink 无任何校验（高风险候选）
- `unknown` — 校验在外层（Interceptor/Filter/中间件）, 本 Phase 不深查

`validated_summary` = 全 param 必须 confirmed 之外四态之一:
- 全 confirmed → `confirmed`
- 全 missing → `missing`
- 全 unknown → `unknown`
- 混合状态 → `mixed`

### 步骤 4：入口跨模块归属

对照 `modules.json` 模块清单，每 entry 的 `module` 字段必对照 path 推断到 modules.json 中已有 module_id 之一。若 path 不匹配任何模块, 在 evidence/phase1a/{path}.md 标 `[?] orphan_entry` 等待 Phase 0 modules.json 补全（但本 Phase 不重跑 Phase 0）。

写 `edges/entry_in_module.json` 边:
```json
{
  "edge_type": "entry_in_module",
  "from": "entry:src/controller/UserController.java:28",
  "to": "module:user-service"
}
```

### 步骤 5：entry_calls 边构建（深度 1, 防爆炸）

若入口类型可识别直接调用方法（如 REST Controller 方法体调 `userService.queryById(id)`）, 沿函数体 grep 同模块调用:

```bash
# 仅 grep 本入口所在文件的方法体, 不扩展全模块
grep -nE "(this|self)?\.\\w+\\(\\)|\\w+\\.\\w+\\(\\)" {path} 2>/dev/null | head
```

每条调用写 `edges/entry_calls.json` 边（不深入展开——后续 Phase 4a 污点追踪时再按 sink 临时扩展方法级图谱 5 跳，这里仅记录入口直接调用的方法名作骨架提示, 防止超预算）:

```json
{
  "edge_type": "entry_calls",
  "from": "entry:src/controller/UserController.java:28",
  "to": "method:src/service/UserService.queryById",
  "call_line": 32
}
```

### 步骤 6：写盘与图谱更新

每 entry WU 完成立即（防会话压缩丢数据, SRC_ACCESS §5）:
- 更新 `knowledge_graph/nodes/entries.json` 该节点（覆写该条, append 到原数组保持顺序）
- 追加 `edges/entry_in_module.json` 一边
- 若 entry_calls 命中, 追加 `edges/entry_calls.json` 一边
- 追加 `audit_log.json` 一条 `{item_id: entry:{path}:{line}, phase: "1a", verdict: "signature_filled", timestamp, wu_id}`
- 写 `evidence/phase1a/{path-sanitized}.md` 证据摘要（含 Read 源码片段留底, 反幻觉#2 验证）

全部 entries 完成后:
- 更新 `progress.json` phases.phase1a = "complete"
- 更新 `progress.json` current_phase = "phase1b" / last_updated = ISO8601

---

## 产出物完整清单

```
{session_dir}/
├── knowledge_graph/
│   ├── nodes/
│   │   └── entries.json                        # 每节点含完整 signature/param_tree/module/validated_summary
│   └── edges/
│       ├── entry_in_module.json               # 新增
│       └── entry_calls.json                   # 新增 (若有调用方法识别)
├── evidence/phase1a/                          # 每入口一 markdown 证据摘要
│   └── {path-sanitized}.md
└── audit_log.json                             # 追加每 entry 一条 phase=1a
```

---

## 300w 适配

按 dispatch_plan.batches 分批派发, max_parallel=3。每 WU 输入明确指定 entry 与源码路径, 不全量载 entries.json 全部到上下文。

| 项 | 值 |
|----|----|
| 单 WU token | 1-3k（Read 1 文件 + 输出） |
| 典型 entry 数 | 100-500（300w 项目 500-2000） |
| 单语言总 token | 0.5-2M |
| max 3 并行耗时 | 15-45 分钟 |

---

## 边界声明

Phase 1a **不审漏洞**——补齐 signature/param/module 仅提供 source 起点 + 可达性数据, 不判定安全性。authz_state 字段在本 Phase 始终是 "unknown", 必由 Phase 1b 补齐。validated_summary 标注仅作 Phase 2a 污点追踪参考（决定是否需追校验链）, 非 VULN 判定。

- 本 Phase 不识别越权候选（Phase 1b 责任）
- 本 Phase 不审 sink（Phase 0 已建索引, Phase 2a 污点追踪）
- 本 Phase 不识别全局鉴权配置（Phase 1b 责任）
- 本 Phase 通用支持任意 Web/RPC/MQ/WS 框架——LLM Read 源码后按实际模式提取, 不为单一框架定制字段