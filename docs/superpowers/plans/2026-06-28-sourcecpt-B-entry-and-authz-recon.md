# SourceCPT 子项目 B: 侦察 Phase 1a 入口面 + Phase 1b 鉴权面

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 实现 SourceCPT 白盒审计套件的 Phase 1a 入口面侦察 + Phase 1b 鉴权面梳理两个 SKILL.md。跑通后可对 Phase 0 已建图谱中所有 entry 节点补齐签名/参数树/模块归属（1a）+ authz_state（1b），为后续 Phase 2a 污点追踪的 source + 可达性判断提供完整数据基础。

**架构：** 延续子项目 A 的纯 Markdown SKILL.md 指令风格。每个 SKILL.md 引用 shared 规范 + 反幻觉约束 + 知识库，通过 LLM 按指令执行 Read/Glob/Grep/Write 完成产出。**不许使用 Python 助手脚本**——纯 bash + LLM 指令操作（A 验证阶段的作弊记入永久约束）。

**技术栈：** opencode SKILL 插件格式、JSON 图谱、bash/Glob/Grep/read（opencode 原生工具）。

**前置：** 子项目 A 已完成（8 shared 规范 + Phase 0 source-map SKILL + entry-types/sink-types 知识库）。

---

## 文件结构

本计划创建/修改以下文件：

```
~/.config/opencode/skills/sourcecpt/
├── SKILL.md                              # 修改: 调度表中 Phase 1a/1b 状态从 "待子项目 B" 改为 "已实现"
└── skills/
    ├── entry-recon/
    │   └── SKILL.md                      # 创建: Phase 1a 入口面侦察子技能
    └── authz-map/
        └── SKILL.md                      # 创建: Phase 1b 鉴权面梳理子技能
```

**设计原则：**
- 两个 SKILL.md 独立可调度，也支持独立调用（`$entry-recon` 或 `$authz-map`）
- entry-recon 依赖 Phase 0 的 entries.json 节点
- authz-map 依赖 Phase 1a 的 entries.json（含补齐 module）+ Phase 0 的 modules.json
- 两个 SKILL 引用同一套 shared 规范与反幻觉约束
- Phase 1c（产品上下文）**不在本计划范围**——属子项目 H

---

## 验证方式

继续用 A 已下载的 Struts 2.5.16 真实源码（/tmp/opencode/sourcecpt-test-real/struts-2.5.16/src）作为测试目标。子项目 A 跑 Phase 0 已产生 knowledge_graph/nodes/entries.json（78 个 entry 节点）+ sinks.json（40 个 sink）+ modules.json（29 个模块），存在 /tmp/sourcecpt-test-001/ 中。

本计划完成 entry-recon + authz-map 后：
1. 在已有 session_dir 上跑 Phase 1a（覆盖式更新 entries.json 含完整签名+参数树+模块）
2. 接着跑 Phase 1b（写入 authz_configs.json + 更新 entries.json 的 authz_state + 产 authz_deviation 边）
3. 验收标准：
   - entries.json 78 个节点都有非 "unknown" 的 signature 与 module
   - entries.json 78 个节点的 authz_state 都非 "unknown"（必有明确 verdict）
   - authz_configs.json 至少捕获到 Struts 全局鉴权配置（Struts 2.x 用 struts.xml + Interceptor 配置）
   - 反幻觉抽查：随机 3 个 entry 的 signature 与实际文件代码比对一致（反幻觉#2）

---

## 任务列表

### 任务 1：编写 entry-recon/SKILL.md（Phase 1a 入口面侦察）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/entry-recon/SKILL.md`

这是 Phase 1a 的核心 SKILL——把 Phase 0 留下的 entries.json 节点逐个补齐 HTTP 方法/路径/签名/参数树/模块归属/param_tree/validated 状态。

- [ ] **步骤 1：创建 entry-recon 目录**

```bash
mkdir -p ~/.config/opencode/skills/sourcecpt/skills/entry-recon
```

- [ ] **步骤 2：编写 entry-recon/SKILL.md 全文**

写入 `~/.config/opencode/skills/sourcecpt/skills/entry-recon/SKILL.md`，全文采用 YAML frontmatter + 中文指令结构：

```yaml
---
name: entry-recon
description: >
  白盒源码审计第二步: 入口面侦察。基于 Phase 0 的入口索引, 逐入口补齐
  HTTP 方法/路径模板/签名/参数树/所属模块/参数校验状态。把 grep 命中行
  升级为结构化入口面清单, 为后续 Phase 2a 污点追踪的 source 提供完整数据。
  覆盖 13 类入口: REST/RPC/MQ/WS/GraphQL/CRON/CLI/脚本/反序列化/
  文件/WebService/自定义协议/事件监听。每入口按入口类型补齐对应字段。
  使用场景: Phase 0 完成后, Phase 2a/1b 之前。
  不使用场景: entries.json 已完整梳理的复检(increment 模式可跳过未变更入口)。
---
```

正文结构（按 A 的 source-map 同等完整度）：

**§1 MUST 输入** — 表：session-dir(必) / project-path(必) / mode / scope
独立运行示例：`$entry-recon --session-dir /tmp/sourcecpt-001 --project-path /abs/repo`

**§2 引用共享规范** — 表（按需读不全量载）：
- `skills/shared/SRC_ACCESS.md`（文件读取每跳 Read 证据）
- `skills/shared/OUTPUT_STANDARD.md`（entries.json 节点字段 schema）
- `skills/shared/OFFSET_RATING.md`（本 Phase 主要用 T0+T1）
- `skills/shared/ANTI_HALLUCINATION.md`（#1 不凭记忆 / #2 不伪造行号 / #6 不残留 unknown 必改 / #9 不凭记忆列文件）

**§3 MUST 输入与 MUST 输出（按 OUTPUT_STANDARD 协议）**

MUST 输入:
- session-dir 含 Phase 0 产出的 `knowledge_graph/nodes/entries.json`
- project-path 与 Phase 0 一致

MUST 输出（Phase 1a 完成判定）:
- `knowledge_graph/nodes/entries.json` 每节点更新：signature 必非 "unknown"，param_tree 必含明示结构（可为空数组但必须审过），module 必非 "unknown"
- `knowledge_graph/nodes/entries.json` 新增字段 `validated_summary`：confirmed/missing/unknown/mixed 四态之一
- `knowledge_graph/edges/entry_in_module.json`（每 entry 一条边）
- `knowledge_graph/edges/entry_calls.json`（若能识别到调用方法, 入口→下游方法）
- `evidence/phase1a/{entry_id}.md`（每入口一 markdown 证据摘要，含 Read 源码片段）
- `audit_log.json` 追加每入口一条 verdict=signature_filled
- `progress.json` phases.phase1a -> "complete"

**§4 反幻觉约束本 Phase 专属**

| # | 约束 | 验证方式 |
|---|------|---------|
| #2 | signature/path 必引 Read 原文行号 | QA 抽 3 条对照 |
| #6 | signature 字段必从 "unknown" 改为实际值；若真不知则需在 evidence 说明 | QA 扫残留 unknown |
| #9 | module 必基于 actual path 推断，不许凭记忆 | 对照 manifest |
| 新增（本 Phase 内联）| 不许跳过任何 entry，每节点必有补齐记录 | audit_log 必每 entry 有条 |

**§5 核心工作流（5 步骤）**

##### 步骤 1：加载 Phase 0 entries.json

```bash
jq length {session-dir}/knowledge_graph/nodes/entries.json
```
读全数 N。每个 entry 节点是一个 WU 单元。设 max_parallel=3（SRC_ACCESS §3 按需加载）。

##### 步骤 2：逐入口补齐结构 [T0-T1]

对每 entry 按 entry_type 分派补齐逻辑。每条 1 个 WU（按 13 入口类型分类）。**每 WU 必须实际 Read 源码文件**（反幻觉#2）。

**WU 输入示例**：
```
[LLM 输入 - 单 entry 补齐 WU]
entry 节点:
  node_type: entry
  entry_type: rest
  path: src/main/java/com/foo/UserController.java
  line: 28
  audit_state: [ ]

LLM 指令:
1. Read 文件 src/main/java/com/foo/UserController.java 第 28 行 ±50 行（≤2k token）
2. 按入口类型补齐字段（通用规则, 不为特定框架定制）:
   - 每种 entry_type 在通用规则上根据源码实际使用的框架识别其入口声明方式
   - 例：entry_type=rest
     * 若是 Spring 注解: 提取 HTTP 方法（@GetMapping 等）+ 路径模板（@RequestMapping 值）+ 方法签名
     * 若是 Struts ActionSupport: 提取 execute() 方法签名 + Action 类返回类型(如 String/SUCCESS)
     * 若是 Flask/Django: 提取 @app.route 路径 + 函数签名
     * 若是 Express: 提取 app.METHOD 路径 + handler 函数签名
     * 不许针对单一框架写死规则——LLM 必 Read 源码后按实际框架模式提取
   - 例：entry_type=cron 跨框架通用: 提取调度表达式(注解/attr) + 执行方法入参类型(若用户输入可达则风险候选)
   - 例：entry_type=deser 通用: 识别触发方式(JSON/XML/原生) + 入参类型 + gadget 链可能性(若反序列化已知库)
   - 每类入口的补齐流程都是 Read + 提取, SKILL.md 不硬编码特定框架字段名

3. 输出 JSON:
   {
     "node_type": "entry",
     "entry_type": "rest",
     "path": "(原值)",
     "line": "(原值)",
     "signature": "GET /api/users/{id} → getUser(Long id) → String",
     "param_tree": [
       {"name": "id", "type": "Long", "source": "path", "validated": "missing|confirmed|unknown"}
     ],
     "module": "(从 path 前 2 段推; 或匹配 modules.json)",
     "authz_state": "unknown",   ← 本 Phase 不动, 1b 填
     "authz_ref": null,
     "validated_summary": "missing|confirmed|unknown|mixed",   ← 新增
     "audit_state": "[ ]"
   }

4. 禁凭记忆推断——必 Read 后填, 行号必对得上文件真实行
```

##### 步骤 3：参数树标注 [T1]

每 param_tree 项标 `validated`：
- `confirmed` — 有显式校验注解或代码（`@NotNull @Size @Pattern @Valid` 或手写校验）
- `missing` — 入参直接传入 sink 无任何校验（高风险）
- `unknown` — 校验在外层（Interceptor/Filter），本 Phase 不深查

**validated_summary** = 全 paramconfirmed=mixed 之外四态之一：
- 全 confirmed → `confirmed`
- 全 missing → `missing`
- 全 unknown → `unknown`
- 混合 → `mixed`

##### 步骤 4：入口跨模块归属

对照 `modules.json`，每 entry 的 `module` 字段必从 path 推断（例如 `src/main/java/com/foo/user/X.java` → `user` 模块）。

写 `edges/entry_in_module.json`：
```json
{
  "edge_type": "entry_in_module",
  "from": "entry:src/controller/UserController.java:28",
  "to": "module:user-service"
}
```

##### 步骤 5：entry_calls 边构建（可选，深度 1）

若入口类型可识别调用方法（如 REST controller 方法体调 service.query()），沿函数体 grep 同模块调用：
```bash
grep -n "self\\.\\|\\..*(\\)\\|\\..* = .*(" src/controller/UserController.java 2>/dev/null | head
```
每条调用写 `edges/entry_calls.json` 边（不深入展开——后续 Phase 4a 污点追踪时再按 sink 临时扩展 5 跳方法级图谱, 这里仅记录入口直接调用的方法名作骨架提示, 防止超预算）。

##### 步骤 6：写盘与图谱更新

每 entry WU 完成立即：
- 更新 `knowledge_graph/nodes/entries.json` 对应节点（覆写该条）
- 追加 `audit_log.json` 一条 `{item_id: entry:{path}:{line}, phase: "1a", verdict: "signature_filled", timestamp, wu_id}`
- 写 `evidence/phase1a/{path-sanitized}.md` 摘要（含 Read 源码片段留底）

全部 entry 完成后：
- 更新 `progress.json` phases.phase1a = "complete"
- 整合 `entry_in_module.json` 和 `entry_calls.json` 边文件

**§6 产出物完整清单**

```
{session_dir}/
├── knowledge_graph/
│   ├── nodes/entries.json                 # 更新含完整 signature/param_tree/module/validated_summary
│   └── edges/
│       ├── entry_in_module.json           # 新增
│       └── entry_calls.json               # 新增
├── evidence/phase1a/                      # 每入口一 markdown 证据摘要
│   └── {path-sanitized}.md
└── audit_log.json                         # 追加每 entry 一条
```

**§7 300w 适配**

按 dispatch_plan.batches 分批派发, max_parallel=3。每 WU 输入明确指定 entry 与源码路径, 不全量载 entries.json 全部 78 entry 到上下文（SRC_ACCESS §3 按需加载, 本 Phase 一次 WU 仅处理 1 个 entry）。

300w 项目典型 entry 数 ~500-2000, 每 WU ~1-3k token, 总耗 ~500 WU × 2k = 1M token 单语言, max 3 并行约 30 分钟。

**§8 边界声明**

Phase 1a **不审漏洞**——补齐 signature/param/module 仅提供 source 起点 + 可达性数据, 不判定安全性。authz_state 字段在本 Phase 始终是 "unknown", 必由 Phase 1b 补齐。validated_summary 标注仅作 Phase 2a 污点追踪参考, 非 VULN 判定。

- [ ] **步骤 3：验证文件存在且结构合规**

运行：
```bash
test -s ~/.config/opencode/skills/sourcecpt/skills/entry-recon/SKILL.md && echo OK
head -10 ~/.config/opencode/skills/sourcecpt/skills/entry-recon/SKILL.md | grep "^name: entry-recon"
```
预期：OK + frontmatter 含 `name: entry-recon`

- [ ] **步骤 4：Commit**

```bash
cd /root && git add .config/opencode/skills/sourcecpt/skills/entry-recon/SKILL.md && \
git commit -m "feat(sourcecpt): add entry-recon Phase 1a SKILL — signature/param_tree/module population"
```

---

### 任务 2：编写 authz-map/SKILL.md（Phase 1b 鉴权面梳理）

**文件：**
- 创建：`~/.config/opencode/skills/sourcecpt/skills/authz-map/SKILL.md`

Phase 1b 为每入口补 authz_state + 识别全局鉴权配置 + 越权面候选。比 1a 复杂——要三层鉴权（方法/类/全局）依次判定 + Struts 等框架的 XML 配置识别。

- [ ] **步骤 1：创建 authz-map 目录**

```bash
mkdir -p ~/.config/opencode/skills/sourcecpt/skills/authz-map
```

- [ ] **步骤 2：编写 authz-map/SKILL.md 全文**

写入 `~/.config/opencode/skills/sourcecpt/skills/authz-map/SKILL.md`，采用 YAML frontmatter + 中文指令结构：

```yaml
---
name: authz-map
description: >
  白盒源码审计第三步: 鉴权面梳理。为 Phase 1a 每个入口补齐 authz_state:
  方法级注解鉴权 / 类级过滤 / 全局框架过滤 (Spring Security/Interceptor/
  Struts Interceptor/XML 配置)。识别鉴权缺失、鉴权绕过、水平越权 IDOR、
  垂直越权候选。LLM 易漏全局层——只看方法注解未见就判无鉴权, 但全局已有。
  这是误报重灾区, 必须三层依次判定。
  使用场景: Phase 1a 完成后, Phase 2a 之前。鉴权状态决定 sink 可达性。
  不使用场景: 仅审技术漏洞不涉鉴权 (罕见, 通常都要做)。
---
```

正文结构：

**§1 MUST 输入** — 表：session-dir(必, 含 Phase 1a 产出的完整 entries.json) / project-path(必)

**§2 引用共享规范** — 表：
- `skills/shared/SRC_ACCESS.md`（按需读, 防 1000 entry 全量入上下文）
- `skills/shared/OUTPUT_STANDARD.md`（authz_configs/entries/authz_deviation schemas）
- `skills/shared/OFFSET_RATING.md`（T0 全局配置识别, T2 路径覆盖判定）
- `skills/shared/ANTI_HALLUCINATION.md`（#1 不凭记忆 / #2 不伪造 / #8 不只看方法级 / #36 不许只读分析方推理——挑战者视角待 Phase 6 处理, 本 Phase 仅做识别）

**§3 MUST 输入与 MUST 输出**

MUST 输入:
- session-dir 含 Phase 0 + Phase 1a 产出的 entries.json（含 module 字段补齐）
- project-path 必须可访问 struts.xml/web.xml/SecurityConfig.java 等全局配置文件

MUST 输出（Phase 1b 完成判定）:
- `knowledge_graph/nodes/authz_configs.json` — 全局鉴权配置节点
- `knowledge_graph/nodes/entries.json` 每节点 `authz_state` 更新为 `secured|unsecured|partial|unknown` 之一（必非 "unknown"，若真无法判改 `[?]` 配 reason, 需显式说明）
- `knowledge_graph/edges/entry_covered_by.json` — 入口被全局规则覆盖
- `knowledge_graph/edges/authz_deviation.json` — 鉴权偏离候选（路径暗示角色与实际鉴权不一致）
- `evidence/phase1b/{entry_id}.md` — 每入口鉴权判定证据
- `evidence/phase1b/global_configs.md` — 全局鉴权配置总览摘要
- `audit_log.json` 追加每 entry 一条 verdict=authz_determined
- `progress.json` phases.phase1b -> "complete"

**§4 反幻觉约束本 Phase 专属**

| # | 约束 | 强制力 |
|---|------|-------|
| #1 | 不凭记忆判全局配置类型——必 Glob/Read 实际配置文件 | 命中即校验, 不在文件不存在不标 secured |
| #2 | 全局配置 path/covers_pattern 必引原文 | 文件路径+行号 |
| #8 | 必须先查全局后查方法——禁只看方法注解判鉴权 | WU 执行顺序强制全局优先 |
| 新增本 Phase 内联 | unsecured 必须三层层层排查后才能下, 任一层未见证据不算 | evidence/phase1b/{entry}.md 必含三层检查记录 |
| 新增 | 越权无业务约束不下 hard 结论标 [?] | IDOR/垂直越权必须标 authz_state="partial" 配 reason |
| 新增 | 全局规则路径匹配必须显式引原文 | covers_pattern 字段必来自实际 SecurityConfig.java 行号 |

**§5 核心工作流（6 步骤）**

##### 步骤 1：全局鉴权配置识别 [T0]

**先全局后方法**（反幻觉#8）。先 Glob 整项目的全局鉴权配置文件：

| 框架类型 | Glob 模式 |
|---------|----------|
| Spring Security | `SecurityConfig.java` `WebSecurityConfig.java` + `@EnableWebSecurity` `@EnableMethodSecurity` `@EnableGlobalMethodSecurity` `SecurityFilterChain` |
| Servlet Filter | `implements Filter` `extends OncePerRequestFilter` `WebMvcConfigurer` `addInterceptors` |
| Struts 2 Interceptor | `struts.xml` `struts-*.xml` + `<interceptor-ref name="..."/>` `<interceptor-stack>` |
| Struts 1 Plugins | `struts-config.xml` `web.xml` 中 `<filter-mapping>` |
| WebFilter 注解 | `@WebFilter("/path")` `@WebListener` |

每条命中写 `authz_configs.json` 节点：
```json
{
  "node_type": "authz_config",
  "config_type": "global_filter|method_secured|interceptor|gateway|webfilter",
  "framework": "spring_security|struts2_interceptor|servlet_filter|other",
  "path": "src/main/resources/struts.xml 或 SecurityConfig.java",
  "line": 45,
  "content_snippet": "(配置原文截 ≤200 字)",
  "covers_pattern": "(该配置声称覆盖的 URL/action 命名空间模式, 引配置原文)",
  "authentication_required": true,
  "audit_state": "[ ]"
}
```

**通用规则**：
- `config_type` 必为通用枚举 `global_filter|method_secured|interceptor|gateway|webfilter` 之一, 不许预定义特定框架名如 `struts_interceptor` 作 config_type（Struts 也归 `interceptor` 类, framework 字段才是细分）
- `framework` 字段标实际使用的鉴权框架, 可能值: `spring_security|struts2_interceptor|servlet_filter|shiro|oauth|other`
- `covers_pattern` 必引配置文件原文（反幻觉#2） — Spring Security 是 antMatchers() 抗, Struts 2 是 `<action name="*">` 命名空间, 用具体值, 不许凭记忆瞎填
- 若配置含 interceptor-stack 列出全部 interceptor 名（Struts）或 antMatchers 路径列表（Spring）供步骤 2 入口比对

##### 步骤 2：逐入口鉴权状态判定 [T0-T1]

对每 entry 按三层依次判定, **顺序强制**: 全局→类级→方法级。LLM 必须先 Review authz_configs 全清单, 再读 entry 所在类文件, 再读方法所在行附近, 不许跳过任一层。

**判定矩阵**:

| 判定 | 条件 | authz_state |
|------|------|-------------|
| `secured` | 三层中任一有效鉴权覆盖 | secured + authz_ref 引具体 config_id 或 file:line |
| `unsecured` | 三层全无任何鉴权声明 | unsecured + 配三层检查记录 |
| `partial` | 全局覆盖但方法特别声明（@SkipAuth / no interceptor on method）→ 冲突/绕过候选 | partial + conflict_reason |
| `unknown` | 全局规则无法判定是否覆盖（如全局用 wildcard 但方法路径复杂） | unknown + [?] reason |

每 entry 的 evidence/phase1b/{entry}.md 必含三层检查记录：
```markdown
# Authz Determination: entry:src/controller/UserController.java:28

## 全局层检查 [T0]
- search authz_configs.json for path match
- 命中 authz_configs#5: `authc` interceptor 覆盖 /user/*
- 或: 全局无任何配置覆盖此路径

## 类级检查 [T1]
- Read UserController.java class header (lines 10-30)
- 是否有 @PreAuthorize / @Secured 在类签名上
- 是否在 @RestController 但全局 @EnableMethodSecurity 关闭

## 方法级检查 [T1]
- Read UserController.java line 28 ± 20
- 是否有 @PreAuthorize/@Secured/@RolesAllowed 在方法签名上
- 是否有手写角色检查代码 if(!user.isAdmin()) throw ...

## 综合判定
authz_state = secured|unsecured|partial|unknown
authz_ref = config:5|file:src/main/java/.../UserController.java:28
reason = (引实际判据)
```

##### 步骤 3：越权面识别 [T1-T2]

无业务约束来源时（Phase 1c 未启用）, IDOR/越权只能做浅识别:

```markdown
水平越权 IDOR 浅识别:
  入参含资源 ID（{id}、userId、orderId...）且
  authz_state in {unsecured, partial} 且
  param_tree 含 path 或 query 类型的 ID 参数
  
判定: 若全满足标 candidate.type=idor, verdict="C3_high_risk_clue"
      若只在非 secured 状态下, 加 "[?] idor_possible" 到 entry audit_state
      不许硬判 VULN（无业务约束, Phase 6 可能在 Phase 1c 启用时复核升级）

垂直越权浅识别:
  路径暗示角色（/admin/, /internal/, /api/admin/*）且
  authz_state != secured 且
  无显式鉴权面向 admin token
  
  标 candidate.type=vertical_privilege_escalation, verdict="C3_high_risk_clue"
```

写 `edges/authz_deviation.json` 边：
```json
{
  "edge_type": "authz_deviation",
  "from": "entry:src/controller/AdminController.java:35",
  "to": null,                          // 未对应到现有 candidate, 本 Phase 仅产出候选
  "deviation_type": "idor|vertical_privesc|authz_bypass",
  "rationale": "路径 /admin/users 但无 @PreAuthorize, 全局无覆盖"
}
```

**反幻觉**: 不许直接判 VULN, 仅作 candidate 留待 Phase 4b LLM 推理或 Phase 1c 业务约束对照。

##### 步骤 4：Sibling-Scan 协同（深度 1，本 Phase 内触发）

发现 `unsecured` 入口立即触发Sibling-Scan（同 Controller 其他方法, 同模块同 entry_type 复查, 同 covers_pattern 全局规则漏洞）:

```
入口 X authz_state=unsecured
  → 同 Controller 其余方法是否也 unsecured? (Read 类内全方法)
  → 同 module 同 entry_type 的其他 Controller 方法?
  → 全局规则声称覆盖该路径模式但本入口未匹配？- 报潜在 bug
```

写 `evidence/phase1b/sibling_scan/{trigger_entry_id}.json` 记录扫描结果。不递归深度 2, 留 Phase 5/6 处理。

##### 步骤 5：写盘与图谱更新

每 entry WU 完成立即:
- 更新 `entries.json` 该节点的 `authz_state` `authz_ref`
- 若新增越权候选，同时写 `edges/authz_deviation.json` 一边
- 若该 entry 被全局规则覆盖写 `edges/entry_covered_by.json` 一边
- 追加 `audit_log.json` 一条 `{item_id: entry:{path}:{line}, phase: "1b", verdict: "authz_determined", timestamp, wu_id}`

全局 configs 写盘集中在步骤 1 完成后一次性写 `authz_configs.json`。

`progress.json` phases.phase1b -> "complete"

**§6 产出物完整清单**

```
{session_dir}/
├── knowledge_graph/
│   ├── nodes/
│   │   ├── authz_configs.json           # 新增
│   │   └── entries.json                 # 更新 authz_state
│   └── edges/
│       ├── entry_covered_by.json       # 新增
│       └── authz_deviation.json        # 新增
├── evidence/phase1b/
│   ├── global_configs.md               # 全局鉴权总览
│   ├── {entry_id}.md                   # 每入口证据
│   └── sibling_scan/                   # 兄弟排查
└── audit_log.json                      # 追加每 entry 一条
```

**§7 300w 适配**

authz_configs 通常 ≤30 全局文件, 单子代理 30 分钟识别。逐 entry 判定按 dispatch_plan max并行 3 子代理。

典型项目 entry 1000 → 1000 WU ≈ 3000k token → 30 分钟。

**§8 边界声明**

Phase 1b 标的 authz_state 仅是"该入口是否有鉴权声明"。**不代表漏洞**：
- `unsecured` 在内部接口场景可能不构成漏洞（被网关/反向代理挡）
- `partial` 需 Phase 6 挑战者确认是否真绕过
- 越权候选必标 `C3_high_risk_clue`, 不许升 C1/C2

最终可信度由 Phase 6 综合判定。

- [ ] **步骤 3：验证文件存在**

运行：
```bash
test -s ~/.config/opencode/skills/sourcecpt/skills/authz-map/SKILL.md && echo OK
head -10 ~/.config/opencode/skills/sourcecpt/skills/authz-map/SKILL.md | grep "^name: authz-map"
```

- [ ] **步骤 4：Commit**

```bash
cd /root && git add .config/opencode/skills/sourcecpt/skills/authz-map/SKILL.md && \
git commit -m "feat(sourcecpt): add authz-map Phase 1b SKILL — 3-layer authz determination + IDOR detection"
```

---

### 任务 3：更新 Pipeline 入口 SKILL.md 中 1a/1b 实现状态

**文件：**
- 修改：`~/.config/opencode/skills/sourcecpt/SKILL.md`

子项目 A 写的入口 SKILL 中调度顺序表标 "Phase 1a/1b 待子项目 B", 现改为 "已实现"。

- [ ] **步骤 1：编辑调度表**

将 SKILL.md 中：
```
| 2 | Phase 1a | `skills/entry-recon/SKILL.md` | 待子项目 B |
| 3 | Phase 1b | `skills/authz-map/SKILL.md` | 待子项目 B |
```
改为：
```
| 2 | Phase 1a | `skills/entry-recon/SKILL.md` | ✅ 已实现（子项目 B） |
| 3 | Phase 1b | `skills/authz-map/SKILL.md` | ✅ 已实现（子项目 B） |
```

- [ ] **步骤 2：验证**

```bash
grep "Phase 1a" ~/.config/opencode/skills/sourcecpt/SKILL.md
```

- [ ] **步骤 3：Commit**

```bash
cd /root && git add .config/opencode/skills/sourcecpt/SKILL.md && \
git commit -m "feat(sourcecpt): mark Phase 1a/1b as implemented in Pipeline entry dispatch table"
```

---

### 任务 4：用 Struts 2.5.16 验证 Phase 1a 入口面补齐

**文件：**
- 无需修改, 在 /tmp/sourcecpt-test-001/ 已有 session 上跑

A 已在 /tmp/sourcecpt-test-001/ 产生 Phase 0 数据（78 entry 节点, 40 sink, 29 module）。本任务跑 entry-recon SKILL 实际执行 Phase 1a。

**重要**: 子代理执行 SKILL 时**不许 Python 助手脚本**（记入永久约束, 从 A 的 Python 作弊污染永久修正）, 仅用 bash + Read + Write + LLM 自然语言指令。

- [ ] **步骤 1:在 session 中触发 Phase 1a**

在 opencode 会话输入：
```
使用 $entry-recon 对 session /tmp/sourcecpt-test-001 继续, project-path /tmp/opencode/sourcecpt-test-real/struts-2.5.16/src
```

观察（Struts 仅作测试, SKILL 应通用支持任意框架）：
- SKILL 应读 entries.json 中 78 个节点
- 逐个 entry Read 源码文件（path 行 ±50 行）
- LLM 按 Read 后识别实际框架模式提取对应字段:
  * Struts 测试中, 多数 entry 是 ActionSupport 类, signature 应匹配 "execute() → String" 或 "execute(): Action 状态码" 模式
  * 也可能存在 REST-like 入口（@SMDMethod 等）, signature 应按其实际声明提取
  * param_tree: execute() 通常无显式参, 但 ActionContext.getContext().getParameters() 可提 request parameters; 或 Action 类 setter 字段（如 setId(Long)）作为隐式参数
  * module: 按 modules.json 中 module_id 之一
  * validated_summary: 标 confirmed/missing/mixed 之一
- 输出 evidence/phase1a/{path-sanitized}.md
- **这是测试用例, SKILL 本身不针对 Struts 优化** — SKILL 应在任何项目（Spring/Flask/Express/Django...）上都通用

- [ ] **步骤 2：验收产出**

人工对照 schema 校验：
- entries.json 每节点 signature 非 "unknown"
- entries.json 每节点 module 非 "unknown"（含模块必来自 modules.json 中的 module_id 之一）
- entries.json 每节点 validated_summary 非 "unknown" 之一（必须显式 confirmed/missing/mixed）
- entry_in_module.json 边数 = entry 数
- audit_log.json 必每 entry 有 phase=1a 条目
- progress.json phases.phase1a = "complete"
- 反幻觉抽查：随机 3 个 entry 的 signature 与实际源码方法签名对照一致

发现问题修正 entry-recon SKILL.md, 重跑。

- [ ] **步骤 3:Commit']

```bash
cd /root && git add .config/opencode/skills/sourcecpt/ && \
git commit -m "test(sourcecpt): verify Phase 1a entry-recon on Struts 2.5.16 — signatures/param_tree/modules populated for 78 entries"
```

---

### 任务 5：用 Struts 2.5.16 验证 Phase 1b 鉴权面

**文件：**
- 无需修改, 继续在 /tmp/sourcecpt-test-001/ 跑

接任务 4, 跑 authz-map SKILL 执行 Phase 1b.

- [ ] **步骤 1:触发 Phase 1b**

在 opencode 会话：
```
使用 $authz-map 对 session /tmp/sourcecpt-test-001 继续, project-path /tmp/opencode/sourcecpt-test-real/struts-2.5.16/src
```

观察（Struts 仅作测试用例, SKILL 应通用支持任意鉴权框架）：
- SKILL 应 Glob 项目所有全局鉴权配置文件（按 AUTHZ_MAP §5 步骤1 通用清单, Spring Security 配置类/Struts Interceptor 配置/Servlet Filter/其他框架 xml 等, 不预设框架）
- Struts 2.5.16 测试中具体期望: struts.xml 中 `<interceptor-ref>` 命中, defaultStack 默认无 authc interceptor → 多 Action 是 unsecured 候选, 显式声明 authc interceptor 的 Action 是 secured
- 判定每个 entry 经过三层鉴权（全局/类/方法）产生 authz_state
- SKILL 本身应不针对 Struts 优化 — 任何项目的 Spring SecurityConfig/Servlet filter/Django middleware 都应能识别

- [ ] **步骤 2：验收**

- authz_configs.json 至少捕获 1 个 Struts 全局配置（struts.xml 中 interceptor 配置+ Struts 已知默认 stack）
- entries.json 每节点 authz_state 非 "unknown" 之一（必 secured/unsecured/partial/unknown 含 reason，不允许仅字串 "unknown"）
- 越权候选（authz_deviation edges）含 reasonable 路径暗示分析
- evidence/phase1b/global_configs.md 含全局配置总览
- evidence/phase1b/{entry_id}.md 至少 5 条（抽样, 不需 78 条但需 N 条代表性）
- audit_log.json 追加 phase=1b 条目
- progress.json phases.phase1b = "complete"
- 反幻觉抽查：随机 3 个 entry 的 authz_state 与实际 struts.xml 中该 action 配置对照

- [ ] **步骤 3：Commit**

```bash
cd /root && git add .config/opencode/skills/sourcecpt/ && \
git commit -m "test(sourcecpt): verify Phase 1b authz-map on Struts 2.5.16 — 3-layer authz determination + IDOR candidates"
```

---

### 任务 6：自检与交接

- [ ] **步骤 1：规格覆盖度自查**

对照设计规格 Phase 1a/1b 部分（§4 主规格 line 206-235）：

Phase 1a 5 步骤（§4 主规格 行 211-215）:
- 加载 Phase 0 入口索引 ✅
- 逐入口补齐结构 ✅
- 参数树标注 ✅
- 入口跨模块归属 ✅
- 写盘与图谱更新 ✅

Phase 1b 4 步骤(行 223-231):
- 全局鉴权配置识别 ✅
- 逐入口鉴权状态判定 + 三层判定 ✅
- 越权面识别 ✅
- 写盘与图谱更新 ✅

Phase 1b 反幻觉约束 "必须先全局后方法 / 不许凭记忆判路径覆盖 / unsecured 必须三层层层排查后才能下 / 越权无业务约束不下结论标 [?]" 全部含 ✅

对照 shared 7 个规范是否被正确引用:

| shared 文件 | entry-recon 引用 | authz-map 引用 |
|------------|------------------|----------------|
| SRC_ACCESS | ✅ 按需读 / WU Read 证据 | ✅ 按 dispatch_plan 分批 |
| OUTPUT_STANDARD | ✅ entries schema / WU 落盘 | ✅ authz_configs / entries 更新 schema |
| OFFSET_RATING | ✅ T0+T1 | ✅ T0 全局识别 + T2 路径覆盖 |
| ANTI_HALLUCINATION | ✅ #1/#2/#6/#9 | ✅ #1/#2/#8 + 新增越权约束 |
| TRACKING_SHEET | (Phase 8d 用) - | 同 |
| CHAIN_STANDARD | (Phase 5 用) - | 同 |
| LOOP_POLICY | (Phase 6 局部 loop) - | 同 |
| BASELINE_DIFF | (持续审计 Phase 9) - | 同 |

- [ ] **步骤 2：占位符扫描**

```bash
grep -RH "TBD\|XX\|TODO\|待定" ~/.config/opencode/skills/sourcecpt/skills/entry-recon/SKILL.md ~/.config/opencode/skills/sourcecpt/skills/authz-map/SKILL.md
```
预期 0 处(ok)。"unknown" 在 entries.json schema 中作为合法初值（由 Phase 1a/1b 改写）不算占位符。

- [ ] **步骤 3：类型一致性**

- entries.json 字段 (node_type/entry_type/path/line/signature/param_tree/module/authz_state/authz_ref/validated_summary/audit_state) 与 OUTPUT_STANDARD §3.6 schema 一致？
- authz_configs.json 字段 (node_type/config_type/path/line/content_snippet/covers_pattern/audit_state) 与 OUTPUT_STANDARD §3.6 一致——后发现 OUTPUT_STANDARD §3.6 没明确写 authz_configs 字段, 它是新节点。在子项目 B 实现时同时**修改 OUTPUT_STANDARD 补 authz_configs schema 段**，保证一致性。

```bash
grep -A 10 "authz_configs" ~/.config/opencode/skills/sourcecpt/skills/shared/OUTPUT_STANDARD.md
# 应见 schema 定义, 否则需补充
```

- [ ] **步骤 4：补全 OUTPUT_STANDARD authz_configs schema**

若任务 2 后发现 OUTPUT_STANDARD §3.6 缺 authz_configs 段, 必须补：

在 §3.6 nodes schemas 后追加：

```markdown
#### authz_configs.json 节点

```json
{
  "node_type": "authz_config",
  "config_type": "global_filter|method_secured|interceptor|struts_interceptor|gateway|webfilter",
  "path": "src/main/resources/struts.xml",
  "line": 45,
  "content_snippet": "<interceptor-ref name=\"authc\"/>",
  "covers_pattern": "/admin/* 或 action namespace",
  "covers_action_namespace": null,
  "audit_state": "[ ]"
}
```
```

- [ ] **步骤 5：补全 OUTPUT_STANDARD edges schemas**

若缺 `entry_covered_by`、`entry_in_module`、`entry_calls`、`authz_deviation` 边 schema 同样补齐。

- [ ] **步骤 6：最终 Commit**

```bash
cd /root && git add .config/opencode/skills/sourcecpt/ && \
git commit -m "chore(sourcecpt): self-check subproject B — spec coverage 100% for Phase 1a/1b, OUTPUT_STANDARD schemas supplemented"
```

- [ ] **步骤 7：子项目 B 完成声明**

完成产出：
- Phase 1a entry-recon SKILL.md
- Phase 1b authz-map SKILL.md
- Pipeline 入口 SKILL.md 调度表更新
- Struts 验证产出（Phase 1a 补齐 + Phase 1b 鉴权判定）
- OUTPUT_STANDARD schema 补 authz_configs/edges
- 自检通过

后续子项目 C-H 按设计 §7 调度顺序依次执行。下个子项目 C (Phase 2a vuln-rules) 依赖 entry + authz + sink + module 完整图谱——这正是子项目 A+B 完成后具备的条件。