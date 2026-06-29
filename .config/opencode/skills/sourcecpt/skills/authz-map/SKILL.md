---
name: authz-map
description: >
  白盒源码审计第三步: 鉴权面梳理。为 Phase 1a 每个入口补齐 authz_state:
  方法级注解鉴权 / 类级过滤 / 全局框架过滤（Spring Security/Struts Interceptor/
  Servlet Filter/Shiro/Django middleware/OAuth 等)三层依次判定。识别鉴权缺失、
  鉴权绕过、水平越权 IDOR、垂直越权候选。LLM 易漏全局层——只看方法注解未见就判
  "无鉴权"是误报重灾区, 必须先全局后方法。SKILL 通用适配任意鉴权框架。
  使用场景: Phase 1a 完成后, Phase 2a 之前。鉴权状态决定 sink 可达性。
  不使用场景: 仅审技术漏洞不涉鉴权（罕见, 通常都要做）。
---

# authz-map — Phase 1b 鉴权面梳理

本 SKILL 为 Phase 1a 输出的每个入口补齐 `authz_state`, 决定该入口"谁能调、需什么角色、有没有全局过滤兜底"。白盒鉴权要素三层——**方法级注解鉴权 / 类级过滤 / 全局框架过滤**, 必须依次判定, **先全局后方法**（反幻觉#8）。只看方法注解未见就判 `unsecured` 是误报重灾区, 许多项目有全局 Filter 默认覆盖所有路径。

SKILL **通用适配任意鉴权框架** — Spring Security/Struts 2 Interceptor/Servlet Filter/Shiro/Django middleware/Express middleware/OAuth/自定义中间件。LLM Glob 配置文件后按实际模式识别, 不预设单一框架字段名。

---

## MUST 输入

| 名称 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `session-dir` | string | **是** | 含 Phase 0 + 1a 产出的 entries.json（含 module 字段补齐）|
| `project-path` | string | **是** | 项目路径, 必可访问全局鉴权配置文件 |

---

## 引用共享规范

| 规范文件 | 用途 |
|---------|------|
| `skills/shared/SRC_ACCESS.md` | 按需读, 防 entries 全量入上下文 / 临时方法级图谱 |
| `skills/shared/OUTPUT_STANDARD.md` | authz_configs / entries / authz_deviation schemas |
| `skills/shared/OFFSET_RATING.md` | T0 全局配置识别, T2 路径覆盖判定 |
| `skills/shared/ANTI_HALLUCINATION.md` | #1 不凭记忆 / #2 不伪造 / #8 必先全局后方法 |

---

## 反幻觉约束本 Phase 专属

| # | 约束 | 强制力 |
|---|------|-------|
| #1 | 不凭记忆判全局配置类型——必 Glob/Read 实际配置文件 | WU 输入必含 Read |
| #2 | 全局配置 path/covers_pattern 必引原文+行号 | evidence/phase1b/global_configs.md |
| #8 | 必须先查全局后查方法——禁只看方法注解判鉴权 | WU 执行顺序强制全局优先 |
| 本 Phase 内联#1.1 | unsecured 必须三层层层排查后才能下, 任一层未见证据不算 | evidence 必含三层检查记录 |
| 本 Phase 内联#1.2 | 越权无业务约束不下硬结论标 [?] | IDOR/垂直越权必须 authz_state="partial" + reason |
| 本 Phase 内联#1.3 | 全局规则路径匹配必须显式引原文 | covers_pattern 字段必来自实际配置文件 |

---

## MUST 输入与 MUST 输出

**MUST 输入**:
- `session-dir` 含 Phase 0 + 1a 产出的 entries.json（含 module 补齐）
- `project-path` 必可达全局鉴权配置文件

**MUST 输出**（Phase 1b 完成判定）:
- `knowledge_graph/nodes/authz_configs.json` — 全局鉴权配置节点（每配置文件每条 interceptor/filter/match 声明一个节点）
- `knowledge_graph/nodes/entries.json` 每节点 `authz_state` 更新为 `secured|unsecured|partial|unknown` 之一必非 "unknown"（真无法判改 `[?]` reason, 显式说明）
- `knowledge_graph/edges/entry_covered_by.json` — 入口被全局鉴权配置覆盖的边
- `knowledge_graph/edges/authz_deviation.json` — 鉴权偏离候选（路径暗示角色与实际鉴权不一致）
- `evidence/phase1b/global_configs.md` — 全局鉴权配置总览（含全部已识别配置）
- `evidence/phase1b/{path-sanitized}.md` — 每入口一 markdown 证据摘要（含三层判定记录）
- `evidence/phase1b/sibling_scan/{trigger_entry_id}.json` — Sibling-Scan 结果（若有触发）
- `audit_log.json` 追加每 entry 一条 verdict=authz_determined
- `progress.json` phases.phase1b = "complete"

---

## 核心工作流

### 步骤 1: 全局鉴权配置识别 [T0]

**先全局后方法**（反幻觉#8 强制顺序）。先 Glob 整项目的全局鉴权配置文件, 不预设框架:

#### 1.1 通用配置文件识别模式（跨框架）

| 配置类型 | 通用 识别模式 |
|---------|--------------|
| 注解式全局配置类 | grep `@EnableWebSecurity\|@EnableMethodSecurity\|@EnableGlobalMethodSecurity\|@EnableWebMvcSecurity\|SecurityFilterChain\|WebSecurityConfigurerAdapter\|OAuth.*Configuration\|AuthFilter\|AuthorizationFilter` (含 Spring Security 主流注解) |
| XML/properties 式配置 | 文件名 `struts.xml\|struts-*.xml` (Struts) `web.xml` (Servlet) `security.xml\|security.properties` (通用) |
| 注解 WebFilter | grep `@WebFilter\|@WebListener\|@Filter` |
| Servlet/Filter Java 类 | grep `implements Filter\|extends OncePerRequestFilter\|extends GenericFilterBean\|implements HandlerInterceptor\|extends HandlerInterceptorAdapter\|implements WebMvcConfigurer` |
| Struts 2 interceptor | grep `implements Interceptor\|extends MethodFilterInterceptor\|AbstractInterceptor` + XML 中 `<interceptor-ref name="...">` `<interceptor-stack>` |
| Shiro | grep `ShiroFilterFactoryBean\|@RequiresPermissions\|@RequiresRoles` + shiro.ini / ShiroConfig.java |
| Django middleware | grep `MIDDLEWARE = \[` (settings.py) |
| Express middleware | grep `app.use(\|router.use(` (Express) |
| 其他自定义 | grep `implements Filter\|Filter.*doFilter\|HandlerInterceptor.*preHandle` |

LLM 必按上述通用识别 + Glob 全项目配置文件位置, 不预设单一框架。

#### 1.2 每条命中写 `authz_configs.json` 节点

```json
{
  "node_type": "authz_config",
  "config_type": "global_filter|method_secured|interceptor|gateway|webfilter",
  "framework": "spring_security|struts2_interceptor|servlet_filter|shiro|django_middleware|express_middleware|oauth|other",
  "path": "src/main/resources/struts.xml 或 SecurityConfig.java",
  "line": 45,
  "content_snippet": "(配置原文截 ≤200 字)",
  "covers_pattern": "该配置声称覆盖的 URL/action 命名空间模式, 必引配置文件原文",
  "authentication_required": true,
  "audit_state": "[ ]"
}
```

**通用规则**:
- `config_type` 必为通用枚举 `global_filter|method_secured|interceptor|gateway|webfilter` 之一, 不预设特定框架字段名
- `framework` 字段细分实际使用的鉴权框架（公开枚举, ABC框架未知填 `other`）
- `covers_pattern` 必引配置文件原文（反幻觉#2） — Spring Security 是 antMatchers() 抗, Struts 2 是 `<action name="*">` 命名空间, Django 是 URL_CONF, 用具体值不许凭记忆瞎填
- 若配置含 interceptor-stack 则对每 interceptor 标 covers 是否含 authc/authorizer
- 每条立即写盘 `knowledge_graph/nodes/authz_configs.json`

#### 1.3 写 evidence/phase1b/global_configs.md 总览

```markdown
# 全局鉴权配置总览

## 检索到的全局配置文件数: N
## 各配置节点 ID/类型/覆盖模式表

| config_id | framework | path | covers_pattern | auth_required |
| ...        | spring_security | config/SecurityConfig.java:28 | /api/admin/** | true |
| ...        | struts2_interceptor | struts.xml:45 | /admin/* | true |
| ...        | servlet_filter | filter/AuthFilter.java:18 | /* | true |
```

### 步骤 2: 逐入口鉴权状态判定 [T0-T1]

对每 entry 按**三层依次判定**, **顺序强制**: 全局→类级→方法级。LLM 必先 Review `authz_configs.json` 全清单, 再 Read entry 所在类文件头, 最后 Read 方法所在行附近, 不许跳过任一层。

#### 2.1 判定矩阵

| 判定 | 条件 | authz_state |
|------|------|-------------|
| `secured` | 三层中任一有效鉴权覆盖（全局 covers_pattern 命中 + 鉴权要求; 或类级/方法级有 @PreAuthorize/@Secured/@RolesAllowed/@RequiresPermissions 等注解; 或手写角色检查代码） | secured + authz_ref 引具体 config_id 或 file:line |
| `unsecured` | 三层全无任何鉴权声明或拦截器/Filter 不覆盖该入口路径 | unsecured + 配三层检查记录 |
| `partial` | 全局声明覆盖但方法显式排除（@SkipAuth / noInterceptor 等）→ 冲突/绕过候选 | partial + conflict_reason 引原文 |
| `unknown` | 全局规则 wildcard 但方法路径复杂无法判定（少见） | unknown + `[?] reason` |

#### 2.2 每入口的 evidence 三层记录模板

evidence/phase1b/{entry_id}.md 必含：

```markdown
# Authz Determination: entry:src/controller/UserController.java:28

## 全局层检查 [T0]
- search authz_configs.json for path match
- 命中 authz_configs#5: covers_pattern "/user/*" 声明 authc interceptor
- 或: 全局无任何配置覆盖此路径

## 类级检查 [T1]
- Read {path} class header (lines 10-30)
- 是否有 @PreAuthorize / @Secured / @RequiresPermissions 在类签名上
- 是否在 @RestController 但全局 @EnableMethodSecurity 关闭

## 方法级检查 [T1]
- Read UserController.java line 28 ± 20
- 是否有 @PreAuthorize/@Secured/@RolesAllowed/@RequiresPermissions 在方法签名上
- 是否有手写角色检查代码 if(!user.isAdmin()) throw...

## 综合判定
authz_state = secured|unsecured|partial|unknown
authz_ref = (具体引用: config:5 或 file:src/...:28)
reason = (引实际判据)
```

#### 2.3 边写盘

每 entry 完成立即:
- 更新 `entries.json` 该节点的 `authz_state` 和 `authz_ref`
- 若被全局规则覆盖写 `edges/entry_covered_by.json`:
  ```json
  {"edge_type":"entry_covered_by","from":"entry:src/controller/AdminController.java:35", "to":"authz_config:5","match_basis":"path pattern /admin/* matched by authc interceptor"}
  ```

### 步骤 3: 越权面识别 [T1-T2]

无业务约束来源时（Phase 1c 未启用）, IDOR 与垂直越权仅做浅识别（不许硬判 VULN, 反幻觉本 Phase 内联 #1.2）:

#### 3.1 水平越权 IDOR 浅识别

```
判定: 入口遵循如下条件即候选
- 入参含资源 ID（{id}、userId、orderId、documentId... 等）
- authz_state in {unsecured, partial}
- param_tree 含 path 或 query 类型的 ID 参数（Phase 1a 即已标）
```

满足候选条件 → 在 `edges/authz_deviation.json` 写已知候选边:
```json
{
  "edge_type": "authz_deviation",
  "from": "entry:src/controller/OrderController.java:35",
  "to": null,
  "deviation_type": "idor",
  "rationale": "入参 id(orderId) 无显式归属校验, authz_state=unsecured",
  "phase4b_followup": true
}
```

#### 3.2 垂直越权浅识别

```
判定:
- 路径暗示角色（/admin/, /internal/, /api/admin/*, /sys/**）
- authz_state != "secured"
- 无显式 admin 角色注解或手写角色检查
```

满足候选 → authz_deviation 边, `deviation_type="vertical_privesc"`。

#### 3.3 鉴权绕过候选

```
判定:
- 全局规则按路径前缀匹配 (Spring antMatchers / Struts namespace)
- 但存在已知框架绕过模式（Spring Security 老版本路径变体如 `/api/admin;/x`, `/api/admin/%2f`）
- LLM 识别到这种路径变体即标 bypass_candidate
```

满足 → authz_deviation 边, `deviation_type="authz_bypass"`。

#### 3.4 反幻觉：候选不是判定

所有 authz_deviation 边必标 `phase4b_followup: true`, **不是 VULN 判定**。Phase 4b LLM 推理在 (1c 业务约束对照时) 复核升级; Phase 6 挑战者最终判定。

### 步骤 4: Sibling-Scan 协同 (深度 1, 本 Phase 内触发)

发现 `unsecured` 入口立即触发 Sibling-Scan（同 Controller 其他方法 + 同模块同 entry_type 复查 + 同 covers_pattern 全局规则漏洞）:

#### 4.1 三轴扩展

```
入口 X authz_state=unsecured
  三轴扩展:
  1. 同 Controller 其他方法（同 file 其他 entry）—— Read 类内全方法
  2. 同 module 同 entry_type 的其他 Controller 方法（按 modules.json）—— Glob + Re-evaluate
  3. 全局规则声称覆盖该路径模式但本入口未匹配？- 报潜在 bug
     (entry_covered_by 是否漏了这条？)
```

#### 4.2 写 sibling_scan 状态

写 `evidence/phase1b/sibling_scan/{trigger_entry_id}.json`:
```json
{
  "triggered_by": "entry:src/controller/AdminController.java:35",
  "trigger_reason": "authz_state=unsecured",
  "scanned_siblings": [
    {"entry_ref": "entry:src/controller/AdminController.java:50", "authz_state": "unsecured", "verdict": "已标"},
    ...
  ],
  "depth": 1,
  "audit_state": "[x]"
}
```

不递归深度 2, 留 Phase 5/6 处理（避免本 Phase 超预算）。

### 步骤 5: 写盘与图谱更新

每 entry WU 完成立即（防会话压缩丢数据, SRC_ACCESS §5）:
- 更新 `knowledge_graph/nodes/entries.json` 该节点的 `authz_state` `authz_ref`
- 若被全局规则覆盖, 写 `knowledge_graph/edges/entry_covered_by.json` 一边
- 若越权候选, 写 `knowledge_graph/edges/authz_deviation.json` 一边
- 追加 `audit_log.json` 一条 `{item_id: entry:{path}:{line}, phase: "1b", verdict: "authz_determined", timestamp, wu_id}`
- 写 `evidence/phase1b/{path-sanitized}.md` 证据（含三层判定记录）

全局 configs 写盘集中在步骤 1 完成后一次写完整 `authz_configs.json`（避免并发写冲突）。

Updates `progress.json` phases.phase1b -> "complete", current_phase 改为下个 Phase。

---

## 产出物完整清单

```
{session_dir}/
├── knowledge_graph/
│   ├── nodes/
│   │   ├── authz_configs.json                # 新增 全局鉴权配置节点
│   │   └── entries.json                     # 更新 authz_state
│   └── edges/
│       ├── entry_covered_by.json            # 新增 入口被全局规则覆盖
│       └── authz_deviation.json            # 新增 越权/绕过候选
├── evidence/phase1b/
│   ├── global_configs.md                    # 全局鉴权总览
│   ├── {path-sanitized}.md                 # 每入口三层判定记录
│   └── sibling_scan/
│       └── {trigger_entry_id}.json          # 兄弟排查状态
└── audit_log.json                           # 追加每 entry 一条 phase=1b
```

---

## 300w 适配

authz_configs 通常 ≤30 全局文件, 单子代理 30 分钟识别（即使 300w 项目全局配置数变化不大）。逐 entry 判定按 dispatch_plan max 并行 3 子代理。

| 项 | 值 |
|----|----|
| 单 entry WU token | 2-5k（读全局清单 + entry 文件 ± 50行） |
| 典型 entry 数 | 100-1000 |
| 总 token | 500k-3M |
| max 3 并行耗时 | 15-60 分钟 |

---

## 边界声明

Phase 1b 标的 `authz_state` 仅是"该入口是否有鉴权声明"。**不代表漏洞**:
- `unsecured` 在内部接口场景可能不构成漏洞（被网关/反向代理挡, Phase 1c 业务约束对照时复核）
- `partial` 需 Phase 6 挑战者确认是否真绕过
- 越权候选（`authz_deviation`）必标 `phase4b_followup=true`, **不许升 C1/C2**——必须经 Phase 4b LLM 推理 + Phase 6 挑战者最终判定

最终可信度由 Phase 6 综合判定。本 Phase 不产 VULN/CLUE 终态, 仅产候选留待下游。

**SKILL 通用适配**: Spring Security/Struts 2 Interceptor/Servlet Filter/Shiro/Django middleware/Express middleware/OAuth/自定义中间件——LLM Glob 配置文件 + 三层判定逻辑, 不预设单一框架字段名, 任何主流鉴权框架都通过通用规则识别。