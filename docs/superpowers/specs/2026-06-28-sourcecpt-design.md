# SourceCPT — 白盒源码安全审计技能套件设计规格

> 基于 LLM 语义驱动的白盒源码安全审计框架。多语言通用、本地源码直读、漏洞链 + 模式进化闭环。
> 参照 GenCPT 容器渗透测试套件的心智模型,按白盒语义重构所有机制。

---

## 设计前提(经头脑风暴确认)

| 维度 | 选择 | 说明 |
|------|------|------|
| 技术栈范围 | 多语言通用(Java/Go/Python/JS/TS) | 每语言一套规则库 + LLM 跨语言语义抽象 |
| 方法论 | LLM 语义为主 + 污点追踪 | 审计深度分层 T0/T1/T2/T3,不依赖 AST 扫器 |
| 源码获取 | 本地路径直接读 | 零外部 MCP,opencode 原生 Read/Glob/Grep |
| 高阶能力 | 全套闭环(漏洞链 + 模式进化) | 9 Phase 完整 Pipeline |
| 规则标准 | OWASP Top10 + CWE 双轨 | 主分组按 OWASP,每条规则映射 CWE 编号 |
| 可信度 | C1/C2/C3 三态 | C1=六项验收门槛全过, C2=条件成立, C3=风险线索 |
| 规模适配 | 300w+ 行代码 | manifest 强制行覆盖率 + 智能派发 + 增量模式 |

---

## 架构方案: A' 自适应镜像

不完全镜像 GenCPT(黑盒与白盒内核有实质差异),而是**保留 11 项可复用机制 + 裁掉 3 项黑盒专用 + 新增 4 项白盒专有**:

### 保留的 11 项 GenCPT 机制

- Pipeline 入口 + Phase 顺序调度 + Task(general) 编排
- 知识图谱 Phase 间解耦
- 断点续传(progress.json 状态机)
- 三库联动(规则库 + 漏洞假设库 + 交叉关联查询)
- 反幻觉约束体系
- 五态标记闭环
- 三态可信度 C1/C2/C3
- 全景覆盖报告 + QA 三层校验
- 条件触发表按需加载
- 分批 WU + 三重校验
- 模式进化闭环 Phase 9

### 裁掉的 3 项黑盒专用

- 安全熔断(无破坏性命令,白盒只读)
- SSH 限速重试(本地无远程命令)
- 方法 C 工具库(无外部工具上传)

### 新增的 4 项白盒专有

- 审计深度分层 T0/T1/T2/T3(替代 L0-L3)
- 污点追踪引擎(source → propagation → sink 证据链)
- 语言感知器(每语言定义 Source/Sink 清单)
- 数据流图谱(source → propagation → sink 边)

---

## 第二部分: 防漏报 + 防误报体系

### A. 防漏报(确保每行代码都分析过)

**100% 找全不可承诺**,但可让覆盖率可度量、可追责、可证明。

| # | 机制 | 实现 |
|---|------|------|
| 1 | Phase 0 manifest 全文件清单 | Glob/bash 列所有源码 → manifest.json(路径+行数+SHA256+state) |
| 2 | WU 逐文件扣账 | 每批 WU 是 manifest 子集,完成时写 audit_log.json `{文件: audited}` |
| 3 | 五态闭环禁 [ ] 残留 | manifest 每文件最终状态必须 `x/[-]/[!]`,QA 第一层拦截残留 [ ] |
| 4 | 覆盖矩阵含行级覆盖 | 全景报告含已读文件/已读行数/总行数 + 未覆盖清单 |
| 5 | 300w 适配分批策略 | <10w 单批 / 10-50w 按模块分批 / >50w 子目录派发 |

### B. 提升漏洞发现深度(降低漏报)

| 层级 | 机制 | 作用 |
|------|------|------|
| L1 | 漏洞模式库(25+ 模式, 8 段格式, OWASP+CWE) | 覆盖已知模式保证基线 |
| L2 | 漏洞假设库 ATK-HYP + 三库联动 | 自动二次推理 |
| L3 | 假阳性场景库 non-vuln-scenarios.md | 反向激活盲区扫描 |
| L4 | Phase 4b LLM 动态推理 | 三静态库未覆盖项最后防线 |
| L5 | Phase 6 双视角对抗 | 第二独立视角专找漏 |

### C. Sibling-Scan 横向排查

发现一处 C1/C2 确证漏洞后,强制同类型兄弟节点全量复查:

**三轴扩展**:
- 同 sink_type(同类型敏感操作)
- 同 entry_type(同类型入口)
- 同 entry-to-sink 路径模式(同数据流形态)

**防爆炸控制**: 深度 ≤ 2, 单次 ≤ 200 兄弟, 并行 max 3, 总 Token 1 亿上上限, 触及写阻塞报告。

**产出**: sibling_scan_log.json 每扫一个兄弟立即落盘,QA 第一层校验完整性。

### D. 防误报

| # | 机制 | 防误报作用 |
|---|------|-----------|
| 1 | 三态可信度强制分级 | 必按门槛逐项举证,非主观判断 |
| 2 | evidence_chain 五段强制 | Source/Propagation/Sanitizer/Sink/Disproof 缺一不可 |
| 3 | 假阳性场景库前置过滤 | VULN 判定前必对照, 命中即 [-] |
| 4 | 证伪条件强制检查 | 规则第 6 段证伪必检, 命中即降级 |
| 5 | 挑战者姿态独立 Phase 6 | 第二独立子代理不看推理只看结论+源码 |
| 6 | 争议升级 + 显式残留 | 双视角争议标 SUSPENDED 显式待人工 |

### E. 防漏报 vs 防误报调和

**核心矛盾**: 防漏报要"宁可多报", 防误报要"宁可少报", 两者相反。

**调和方案**: 三态可信度全报但分池 — C1 必有 6 项证据, C2/C3 标可信度让甲方自选。

---

## 第三部分: 入口面与 sink 索引

### 13 入口面通道

| # | 通道 | grep 模式示例 |
|---|------|--------------|
| 1 | REST | `@RequestMapping @Get @Post flask.route` |
| 2 | RPC | `@DubboService grpc.Service .proto Service` |
| 3 | MQ | `@RabbitListener @KafkaListener @StreamListener` |
| 4 | WebSocket/SSE | `@ServerEndpoint WebSocketHandler socket.on` |
| 5 | GraphQL | `@QueryMapping @MutationMapping type Query` |
| 6 | 定时任务 | `@Scheduled @Cron celery.schedule apscheduler` |
| 7 | 命令行 | `main( cobra.Command @CommandLineRunner argparse` |
| 8 | 脚本入口 | `*.sh *.py 入口文件` |
| 9 | 反序列化 | `@JsonView @XmlElement ObjectInputStream readObject XMLDecoder` |
| 10 | 文件类 | `MultipartFile Part Commons FileUpload + 用户控制路径的 read` |
| 11 | WebService/SOAP | `@WebService @SOAPBinding JAX-WS @WebMethod` |
| 12 | 自定义协议 | `ChannelInitializer Netty Mina IoHandler Decoder` |
| 13 | 事件监听 | `@EventListener ApplicationListener @Subscribe EventListener` |

### 28 sink 类(按 OWASP Top10 × CWE 双轨)

| OWASP | sink 类 | 示例 grep |
|-------|---------|----------|
| A01 | 路径遍历文件读 | `new File( Files.read ioutil.ReadFile OpenFile` |
| A01 | 反射类加载(用户控制类名) | `Class.forName( loadClass( URLClassLoader` |
| A02 | 弱哈希 | `MessageDigest.getInstance("MD5\|SHA1")` |
| A02 | 弱加密 | `Cipher.getInstance("DES\|RC4")` |
| A02 | 硬编码密钥 | `password= secret= apiKey= 字面量` |
| A02 | 弱随机 | `new Random( java.util.Random Math.random` |
| A03 | SQL 拼接 | `${} +.query sql % cursor.execute(%` |
| A03 | 命令执行 | `Runtime.exec ProcessBuilder exec.Command subprocess` |
| A03 | LDAP 注入 | `LdapTemplate DirContext.search query(` |
| A03 | NoSQL 注入 | `MongoCollection.find( redis-cli eval(` |
| A03 | XPath 注入 | `XPath.evaluate xpath.compile` |
| A03 | 表达式注入(SpEL/OGNL/EL) | `SpelExpressionParser Ognl.getValue` |
| A03 | 模板注入 SSTI | `freemarker.template velocity Context Thymeleaf` |
| A03 | XSLT 引擎 | `TransformerFactory.newTransformer XSLTC` |
| A03 | HTTP 头注入 CRLF | `response.setHeader( addHeader(` |
| A03 | 邮件头部注入 | `MimeMessage setHeader InternetAddress` |
| A03 | 日志注入(Log4Shell) | `logger.info( logger.error( + log4j` |
| A05 | 调试端点暴露 | `management.endpoints.actuator` |
| A05 | 详细错误回显 | `e.printStackTrace response.write(e` |
| A06 | 组件 CVE | Phase 2 SBOM 子模块 |
| A07 | 鉴权缺失 | Phase 1b 梳理 + Phase 4 校验 |
| A07 | 越权 IDOR | Phase 4 LLM 逻辑分析 |
| A08 | Java 原生反序列化 | `ObjectInputStream readObject XMLDecoder` |
| A08 | fastjson/XStream | `JSON.parseObject JSONObject.parse XStream.fromXML` |
| A08 | JNDI 注入 | `Context.lookup InitialContext.doLookup` |
| A10 | HTTP/Socket 外呼 SSRF | `HttpClient RestTemplate http.Get requests.post urllib` |
| A10 | DNS 解析 | `InetAddress.getByName dns.Client lookup` |
| 辅助 | XXE 解析 | `DocumentBuilderFactory SAXParserFactory XMLReader JAXBContext` |
| 辅助 | 文件上传 | `MultipartFile.transferTo Part.write Files.copy` |
| 辅助 | 开放重定向 | `response.sendRedirect redirect:` |
| 辅助 | XSS 输出 | `response.getWriter.write println .html(` |

### 保障覆盖全面的 4 项机制

1. **双轨索引 OWASP Top10 × CWE**: 每 sink 必挂 1 OWASP + 1 CWE, 两轴都出矩阵
2. **Phase 4b LLM 补盲区**: 未列举的 sink 动态 CoT 推理 + ReAct 读文件
3. **Phase 9 模式进化闭环**: 新 sink 模式晋升写 `_learned/` -> 下次入清单
4. **Sibling 反向激活**: 确认漏洞后全图谱同 sink_type 兄弟强制复查

---

## 第四部分: Pipeline 详设计

### Phase 0 — 工程地图 (`source-map/SKILL.md`)

**设计意图**: 建立"可审计工作集"——全文件清单 manifest、语言指纹、入口面索引、sink 索引、调用图骨架、智能派发策略。300w 行项目无 Phase 0 则无结构化覆盖率可言。

**MUST 输入**:

| 名称 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `project-path` | string | **是** | 本地源码根目录绝对路径 |
| `scope` | enum | 否 | `java/go/python/js/ts/all`(逗号多选), 默认 `all` |
| `mode` | enum | 否 | `fast/full/increment` |
| `changed-since` | string | 否 | increment 模式 git ref |

**核心工作流**:

1. **语言指纹识别 [T0]**: 统计每语言文件数+行数, 决定 effective_scope
2. **manifest 全文件清单 [T0]**: Glob 列源码 → manifest.json(每文件 path/lang/loc/sha256/state/category)
   - 跳过规则: `*/test/*` `*/node_modules/*` `*/vendor/*` 等, 每条必写 skip_reason
3. **入口面索引 [T0]**: 按 13 通道 grep → 知识图谱 entry 节点
4. **sink 索引 [T0]**: 按 28 类 grep → 知识图谱 sink 节点
5. **调用图骨架 [T1]**: 仅模块级 + 模块间引用关系, 不建方法级全图
6. **智能派发策略 [T0]**: 按体量自适应(小<10w 单批/中10-50w 模块分批/大>50w 子目录派发)
7. **写盘与图谱初始化**

**产出物**: session_config.json / manifest.json / dispatch_plan.json / audit_log.json / knowledge_graph/nodes/entries+Sinks+modules / edges/calls_module

**反幻觉约束**: 不许跳过 manifest 文件(必 skip_reason) / 不许凭记忆判入口 sink(必有 grep 输出) / 占位符必替换

### Phase 1a — 入口面侦察 (`entry-recon/SKILL.md`)

**设计意图**: 把 Phase 0 的"grep 命中行"升级为"结构化入口面清单"——每入口补齐 HTTP 方法、路径模板、参数树、所属模块、鉴权状态(unknown 待 1b)。

**核心工作流**:
1. 加载 Phase 0 入口索引
2. 逐入口补齐结构 [T0-T1]——按入口类型补齐签名/参数树
3. 参数树标注 [T1] —— validated=confirmed/missing/unknown(决定 Phase 4 是否需追校验链)
4. 入口跨模块归属
5. 写盘与图谱更新

**产出物**: knowledge_graph/nodes/entries.json(含完整签名+参数树) / edges/entry_in_module + entry_calls

### Phase 1b — 鉴权面梳理 (`authz-map/SKILL.md`)

**设计意图**: 为每入口补齐 authz_state, 决定该入口"谁能调、需什么角色、有没有全局过滤兜底"。白盒鉴权要素三层: 方法级注解鉴权 / 类级过滤 / 全局框架过滤。

**核心工作流**:
1. 全局鉴权配置识别 [T0] —— SecurityFilterChain/Interceptor/@EnableMethodSecurity
2. 逐入口鉴权状态判定 [T0-T1] —— 三层依次判定, final authz_state:
   - `secured`: 三层任一有效鉴权
   - `unsecured`: 三层全无(鉴权缺失候选)
   - `partial`: 全局覆盖但方法特殊声明(可能冲突/绕过)
   - `unknown`: 全局规则无法判断是否覆盖
3. 越权面识别 [T1-T2] —— 水平越权(IDOR)/垂直越权/鉴权绕过(框架已知模式)
4. 写盘与图谱更新

**Sibling-Scan 协同**: 发现 `unsecured` 入口立即触发同 Controller 其他方法、同模块同 entry_type 复查、同 covers_pattern 全局规则漏洞检查

**反幻觉约束**: 必须先全局后方法 / 不许凭记忆判路径覆盖 / unsecured 必须三层层层排查后才能下 / 越权无业务约束不下结论标 [?]

### Phase 1c — 产品上下文知识图谱 (`product-context/SKILL.md`)[可选]

**设计意图**: 通过读取产品文档建立"业务约束图谱",让 Phase 4b 能对比"代码做了什么 vs 业务该做什么",发现不一致即漏洞。把审计从"找已知漏洞模式"提升到"找代码偏离业务约束"。

**MUST 输入**: `context-path` 产品文档根目录

**可读文档类型(8类)**: API 文档 / PRD/需求 / HLD/LLD / 安全清单 / 通信矩阵 / 数据字典 / 状态机定义 / 运维部署

**核心工作流**:
1. 文档清单与可读性核验 [T0]
2. 业务约束提取 [T1] —— 每类文档按模板提取结构化约束
3. 业务流程边构建 [T1] —— 跨系统流程
4. 文档与代码符号关联 [T1] —— 业务约束锚定到具体入口
5. 写盘与图谱初始化

**后续 Phase 使用**: Phase 4a 命中 sink 时查该入口 constraint; Phase 4b 对 `code_match=unknown` 约束判定 matched/deviated/missing(deviated 即业务逻辑漏洞候选); Phase 5/6 链与挑战; Phase 8c QA 覆盖矩阵含"业务约束覆盖率"

**Sibling 协同**: 发现 `deviated` 后横向排查同 constraint_type 兄弟

**反幻觉约束**: 不许凭记忆判业务约束(必引文档原文+行号) / 不许把"文档未提"直接判漏洞(标 code_match=missing 作线索) / 不许跳过 code_refs 关联

### Phase 2a — 漏洞规则匹配 (`vuln-rules/SKILL.md`)

**设计意图**: 基于 Phase 0 sink 索引 + Phase 1a 入口面, 对每 sink 做 source-to-sink 污点追踪, 产出带证据链的漏洞候选(C3 起步)。需要 propagation 链才成立, 非模式命中即 fail。

**核心工作流**:
1. 加载 sink 索引, 按 sink_type 分组成审计队列(典型 200-500 节点, 300w 项目可达 1000+)
2. 逐 sink 污点追踪 WU [T1-T2]:
   2.1 输入: sink 信息 + 规则文件(按需)
   2.2 沿调用链反查 source, 每跳记录 `{from, to, file, line, transform_type}`
   2.3 检查 source 用户可控(entry 的 param_tree)
   2.4 检查 Sanitizer(参数化/白名单/转义, 判有效/无效)
   2.5 检查证伪条件(规则第 6 段必读必检)
   2.6 输出 evidence_chain JSON
3. 可信度初判: C3 起步, C2 需部分假设, [-] 命中证伪, [?] 鉴权后可达
4. 知识图谱写盘: nodes/vuln_candidate.json + edges/data_flow.json

**300w 适配**: sink 数 1000 / max 并行 3 / 每 WU 6-15k token / 总耗约 8h

**反幻觉约束**: 不许凭 sink 代码片段直判 VULN(必有 propagation 链) / 不许跳过证伪条件 / propagation 每跳必须有 Read 证据(不凭记忆推断) / source 不可控不许硬判(标 C3 待 Phase 4b)

### Phase 2b — 编码规范合规检测 (`compliance-audit/SKILL.md`)[可选, --with-compliance 启用]

**设计意图**: 与漏洞规则不同——模式命中即 fail, 不需证据链。覆盖 OWASP ASVS + CWE Top25 + 企业自定义规范(用户提供)。独立可选使用, 默认 Pipeline 不跑。

**7 分组各独立子skill**(对照 GenCPT K8s/Docker/Containerd 三 compliance 分立):

| ID | 名称 | 典型规则 |
|---|------|---------|
| G1 | 凭证管理 | 硬编码口令/APIKey/Token/连接串, 密钥库未加密 |
| G2 | 加密算法 | MD5/SHA1/DES/RC4/3DES, ECB 模式, TLS1.0/SSLv3 |
| G3 | 随机数 | `new Random()` 用于安全场景, 未设种子 |
| G4 | 密码策略 | 口令复杂度不足, 无强度校验 |
| G5 | 数据保护 | 敏感字段明文存储, 日志打印请求体, `e.printStackTrace()` 回显 |
| G6 | 信息泄露 | 调试端点暴露, 版本信息泄露 |
| G7 | 会话管理 | 会话 Token 弱随机, 固定 Token, Cookie 无 Secure/HttpOnly |

加 `enterprise/` 目录支持用户自定义规范。

**分层处理(三源融合, 适配 300w 行)**:

- **第 0 层 文件分块 [T0, 零 LLM]**: 按 500 行/块切, 函数边界对齐, 5 行重叠防跨块漏
- **第 1 层 逐块语义合规审计 [T1, LLM 主介入]**: 每块单 WU, LLM 逐函数审计 7 分组所有规则, 不许只扫字面量关键字
- **第 2 层 跨块聚合 [轻量 LLM]**: 同文件多块命中合并, uncertain 项扩读 ±100 行
- **第 3 层 假设库联动 [T2]**: 仅对真 fail 项做三库联动, 升级为 C3 漏洞候选

**与 Phase 2a 的本质区别**:

| 维度 | Phase 2a 漏洞规则 | Phase 2b 编码合规 |
|------|-------------------|------------------|
| 判定门槛 | source-sink 证据链完整 | 模式命中即 fail |
| 证据要求 | 必有 propagation 链 | 仅命中行+上下文 |
| 可信度 | C1/C2/C3 三态 | pass/fail/warn |

**反幻觉约束**: 不许只扫字面量关键字(必逐函数语义判) / 每函数必标审计结果(无命中也标CLEAN防遗漏) / 跨块切断的违规不许直接判 false(标 uncertain 扩读)

### Phase 3 — 交叉关联 (`cross-ref/SKILL.md`)

**设计意图**: 把 Phase 2a/2b 的单点发现 + Phase 1a/1b/1c 的入口/鉴权/约束交叉比对, 发现多因素叠加, 升级已有候选的可信度, 产出新候选。对应 GenCPT 三库联动的白盒版。

**三库**(`knowledge-bases/hypothesis-libraries/`):

| 库 | 文件 | 性质 | 卡片数 |
|---|------|------|-------|
| 合规假设库 | compliance-hypotheses.md | 静态映射: 编码违规 → 漏洞可能 | 35 CHK-CAND |
| 漏洞假设库 | attack-hypotheses.md | 静态映射: 代码模式 → 攻击可能 | 25 ATK-HYP |
| 交叉查询库 | cross-ref-queries.md | LLM 动态查询模板 | 3 XREF |

**核心工作流**:
1. 加载全图谱节点 [T0]
2. 静态三库联动 [T1]: LLM 逐 CHK-CAND/ATK-HYP 条件匹配, 命中即升级或新建候选
3. LLM 动态补充推理(XREF-001) [T2]: 主动推理未列举组合, 标 `[?] llm_reasoning`
4. 写盘与图谱更新(新增 edges/cross_ref + upgrade)

**反幻觉约束**: 三库新假设必须引原文证据 / 静态库未命中不许跳过 LLM 推理

### Phase 4a — 漏洞模式匹配 (`attack-pattern/SKILL.md`)

**设计意图**: 8 段格式漏洞模式库 + 条件触发表按需加载, 对 Phase 3 输出候选做精确整体攻击路径验证。模式库与 2a 规则不同——2a 按 OWASP/CWE 做 single sink 污点追踪, 4a 按已知完整漏洞模式组织。

**条件触发表按需加载**: 不读全部 25+ 模式, 根据 candidate 的图谱字段(sink_type / entry_type / authz_state / cross_ref)只读命中的模式 SKILL.md(对照 GenCPT 省 Token)。

**单模式 8 段格式**:
1. 前置条件
2. 探测指令(白盒用 Read 读 sink 函数 ±50 行)
3. 攻击验证(构造 payload 不运行)
4. 差分证明(白盒版: [T1] 基线 + [T2] payload 沿路径走通 + [T1] 对比 sanitizer 覆盖)
5. 绕过策略
6. 证伪条件(命中即 [-])
7. CWE / OWASP 映射
8. 修复建议引用

**核心工作流**:
1. 加载候选清单 [T0]
2. 条件触发表匹配 [T0] —— 仅读命中的模式
3. 逐候选模式验证 [T1-T2] —— 沿模式探测/验证/差分/证伪/绕过逻辑执行
4. 输出 pattern_match 节点, candidate 升级 C3→C2

**反幻觉约束**: 不许凭模式命中直判 C1 / 证伪必检 / 模式不命中不许跳过推理(进 Phase 4b)

### Phase 4b — LLM 推理补盲区 (`attack-reasoning/SKILL.md`)

**设计意图**: Phase 4a 模式库覆盖已知模式, 新漏洞组合漏网。Phase 4b 兜底 — 对所有未结案候选 + 模式库未匹配 sink/entry 做 CoT+ReAct 从零推理。是漏报最后防线。

**触发条件**: 候选无模式匹配强制推理 / 候选 [?] 状态 / candidate 证据链残缺 / Phase 1c code_match=deviated 强制推理业务漏洞 / Phase 0 sink 节点未 Phase 2a 处理抽样推理

**核心工作流**:
1. 识别盲区候选 [T0]
2. CoT 推理 WU [T1-T2] —— ReAct 循环: Thought→Action(Read 文件)→Observation→Thought
3. 业务逻辑漏洞推理 [T3] —— 对 Phase 1c deviated constraint 做白盒强项业务漏洞推理
4. insights 产出与进化喂养 —— 写 evidence/insights.md 供 Phase 9

**反幻觉约束**: 推理发现必附完整 ReAct 链 / 推理新候选标 llm_reasoning / insights 不许直接晋升模式(必 Phase 9 4 门槛)

### Phase 5 — 漏洞链 + Sibling-Scan (`chain-builder/SKILL.md`)

**设计意图**: 把单点候选组成利用链 CHAIN-xxx, 评估可达性与总影响。每确认 C2 触发 Sibling-Scan 同族三轴扩展全量复查。

**核心工作流**:
1. 加载所有候选与节点 [T0]
2. 链可达性分析 [T1-T2] —— 每候选作链起点, 沿图谱边找前置/后置组成链
3. Sibling-Scan 三轴扩展 [T1-T2]: 触发后批量 50 兄弟/WU, max 并行 3, 递归深度 2 防爆炸
4. 写盘: evidence/phase5/CHAIN-*.md+.json + sibling_scans/

**反幻觉约束**: 链深超 5 必报截断声明 / sibling-scan 禁跳任何兄弟(必从图谱查清单) / 已扫兄弟必留 audit_state / 推翻原候选不许隐藏

### Phase 6 — 挑战者对抗 + 六项验收 (`chain-verify/SKILL.md`)

**设计意图**: 独立挑战者子代理不看 Phase 4-5 推理上下文, 只接收结论+evidence_chain+源码, 从零反向论证"这个漏洞成立吗?"。六项验收门槛全过升 C1。

**六项验收门槛(吸收 v0.6.1 标准)**:
1. 可达: 代码路径 entry→sink 无断点
2. 可控: source 用户输入(path/query/body), 非 db/config/const
3. 可传播: propagation 链每跳有 read 证据
4. 可利用: sink 语义可造成真实安全影响, 防护无效/缺失
5. 可复现: 可构造 payload 沿路径可达(白盒不运行)
6. 影响成立: 能说明具体安全影响

**判定**:
- 全 6 项 pass + 挑战者未推翻 → **C1 confirmed** (VULN-{N}.md)
- 4-5 项 pass + 1-2 assume → **C2 condition_met** (VULN-{N}.md 标 condition_met)
- 缺 3+ / 线索不完整 → **C3 high_risk_clue** (CLUE-{N}.md)
- 挑战者推翻 + 仲裁支持 → **[-] disproved** (NOVULN-{N}.md)
- 对抗 loop 争议未决 ≤3 轮 → **[?] suspended** (SUSPENDED-{N}.md)

**核心工作流**:
1. 挑战者独立复审 [T2] —— WU 输入仅 claim+evidence+源码路径, 不许读 phase4-5 markdown
2. 对抗 loop [T2] —— 争议触发: 轮1 原方回应→轮2 挑战者反驳→轮3 第三方仲裁。≤3 轮, 每轮 self-contained, 必输出新 Read 证据防 LLM 加固幻觉
3. 六项验收判定 [T2]
4. Sibling-Scan 二次扩展深度 2 [T2]
5. 写盘与图谱更新

**反幻觉约束**: 挑战者不许读分析方推理 / 六项逐项必输出 verdict(不许"综合来看") / 对抗 loop 每轮必新 Read 证据 / 推翻结论不许隐藏 / C1 必须六项全 pass / SUSPENDED 不许擅自升 VULN

### Phase 7 — POC 生成 (`poc-generator/SKILL.md`)

**设计意图**: 仅对 C1/C2 漏洞生成可复现材料。C3/NOVULN/SUSPENDED 一律不生成(防菜鸟拿未验证材料重放)。无门禁直接生成(白盒 POC 是证据非武器)。

**核心工作流**:
1. 加载 C1/C2 漏洞 [T0]
2. 按入口类型生成 POC [T1-T2]:
   - **Web 入口**(REST/RPC/WS/GraphQL/WebService): 产 Burp Suite 可粘贴原始 HTTP 请求包(.http 文件)+ payload 变体
   - **非 Web 入口**(CRON/MQ/CLI/反序列化/事件监听): 产"代码路径触发说明"+ 复现步骤+ 反序列化附加 gadget 链说明
3. SARIF 片段生成 [T0] —— 每发现产 SARIF result 片段(含 codeFlows), Phase 8 汇总为 `pentest_report.sarif` 集成 GitHub Code Scanning

**SARIF**: Static Analysis Results Interchange Format, OASIS 标准 JSON 格式。GitHub/GitLab CI 的 Security tab 直接消费, 在 PR 上画红线显示 codeFlow。

**产出物**:
```
evidence/phase7/
├── POC-001-{slug}.md           # 人读: 说明+payload 变体
├── POC-001-{slug}.json         # 程序消费
├── POC-001-{slug}.http         # Burp Suite 原始 HTTP 包(仅 Web 入口)
├── POC-001-{slug}.sarif.json   # SARIF 单条片段
└── poc_index.json              # 全 POC 索引
```

**反幻觉约束**: C3/NOVULN/SUSPENDED 不许生成 POC / POC 不许填真实凭证(占位符 {{test_token}}) / payload 必须沿 evidence_chain 验证可达 / 非 Web 入口不许伪造 HTTP 包

### Phase 8 — 报告交付

#### Phase 8a — 漏洞专项报告 (`report-finding/SKILL.md`)

每 C1/C2 漏洞一独立 md+json 报告,含证据链五段:

```markdown
# VULN-001 SQL注入 — UserController.getUser

## 1. 漏洞摘要
| 字段 | 值 |
| ID | VULN-001 |
| 类型编号 | A01 / CWE-89 |
| 类型名称 | SQL注入 |
| 子类型 | 字符串拼接式 |
| 可信度 | C1 实证复现 |
| 位置 | UserDao.java:102 |
| 入口 | GET /api/users/{id} (未鉴权) |
| 鉴权依赖 | 否 |

## 2. 证据链 (代码事实)
### 2.1 Source (用户可控起点)
### 2.2 Propagation (传播链每跳, Read 证据)
### 2.3 Sanitizer (校验清扫, 有效/无效/缺失)
### 2.4 Sink (敏感操作, 代码原文)
### 2.5 证伪检查(已查证)

## 3. 漏洞原理 (逻辑推论)
## 4. 影响成立 (具体安全影响)
## 5. 挑战者验证记录 (轮次/verdict/六项)
## 6. POC 引用 (evidence/phase7/POC-001-*.md+.http+.json)
## 7. 修复建议 (基于实际代码)
### 紧急 P0: 参数化代码改写
### 加固 P1: 白名单校验 + 鉴权覆盖
### 架构层 P2: SecurityFilterChain 覆盖
## 8. 残余风险
## 9. 关联 (链 + sibling)
```

JSON 版本同结构供程序消费。

#### Phase 8b — 漏洞链报告 (`report-chain/SKILL.md`)

每条 CHAIN 一份 md+json, 含链步骤/总影响/可达性/各步证据引用/链整体修复(在哪一步切断最经济)。

#### Phase 8c — 合规报告 (`compliance-report/SKILL.md`)[可选, --with-compliance 启用]

汇编 Phase 2b 合规违规。含执行摘要(7 分组 fail/pass/warn 矩阵)+ fail 项明细(每项规则原文+代码位置+修复引用)+ 合规-攻击叠加升级(纯合规态升级为链节点项)。

#### Phase 8d — 全景报告 + QA + 修复 + 跟踪表 (`report-summary/SKILL.md`)

**10 章结构**:
1. 执行详情(项目/总行数/语言/模式/审计日期/Token 消耗/Phase 统计)
2. 环境覆盖全景(manifest 入口面/sink/鉴权/产品上下文/SBOM/合规覆盖率)
3. ⚠️ 最大检测盲区提示(Top 3 高优先级盲区, 诚实声明)
4. 漏洞分布
5. 攻击面覆盖矩阵(7 OWASP × 13 入口 × 28 sink, 已证伪也入矩阵, 不可 Override)
6. 漏洞链(CHAIN 列表, 总影响/步数/最大可信度)
7. Sibling-Scan 横向排查收益(初始确认/sibling 新增/复查安全/待人工)
8. Phase 6 挑战者对抗记录(展示防误报, 推翻数/维持/争议仲裁)
9. 漏洞来源标识(📚 已知模式库 / 🧠 LLM 推理 / 🔄 学习模式)
10. 未覆盖事项及盲区原因(高/中/低优先级 + 盲区提示)

加损失评估与 Top 5 风险/修复优先级矩阵(P0/P1/P2/P3+盲区项)。

**QA 三层校验**:
1. **结构校验(不可 Override)**: manifest 行覆盖率 100% / VULN evidence 五段完整 / CHAIN 结构完整 / 五态无 [ ] 残留 / md+json+sarif 齐全 / 反幻觉 56 条逐条扫违规(编号 1-59, 中 12-14 grep 版弃用保留占位)
2. **语义抽检(可 Override 记原因)**: 抽 3 条 VULN 重新 Read 源码核对行号 / 抽 5 条 CAND 验证 disproof_checked / 抽 2 条 NOVULN 验证推翻稳定
3. **覆盖矩阵(不可 Override)**: 7 OWASP × 13 入口 × 28 sink 矩阵无空白 / 已证伪必入矩阵 / 盲区 Top 3 显式列

**三轨交付**:
```
reports/
├── pentest_report.md            # 人读全景
├── pentest_report.json          # 程序消费(全章节结构化)
├── pentest_report.sarif         # CI 集成(GitHub Code Scanning)
├── patch_suggestion.md + .json  # 修复建议集(每漏洞引用)
├── qa_report.md + .json         # QA 三层校验记录
├── issue_tracking.csv           # ★新增 必产 Excel 跟踪表
└── issue_tracking.xlsx          # ★新增 可选含格式
```

**跟踪表字段(column)**:

| 列 | 含义 |
|---|------|
| ID | 发现编号 |
| 类型编号 | OWASP + CWE |
| 类型名称 | 具体漏洞中文 |
| 子类型 | 细分类 |
| 标题 | 简述 |
| 可信度 | C1/C2/C3 |
| 位置 | 文件:行 |
| 入口 | 入口签名 |
| 鉴权依赖 | 是/否 |
| 影响 | 具体影响 |
| 链 ID | 关联链 |
| Sibling | 横向关联 |
| POC 引用 | 文件路径 |
| 修复优先级 | P0/P1/P2 |
| 修复建议引用 | 文件路径 |
| 状态 | 未修复/已修复/已确认/已驳回/待复检(初值"未修复"不许留空) |
| 残余风险 | 描述 |
| 审计日期 | 时间戳 |

xlsx 含 3 sheet: Issues / Chains / Summary。

**类型编号 → 类型名称 → 子类型映射表**(位于 `skills/shared/TRACKING_SHEET.md`, Phase 8d 按查表填, 不许凭记忆):

| 类型编号(CWE) | 类型名称 | 常见子类型 |
|--------------|---------|----------|
| CWE-89 | SQL注入 | 字符串拼接式/ORM原生SQL式/MyBatis ${}占位符式 |
| CWE-78 | 命令注入 | Runtime.exec拼接/ProcessBuilder拼接/Shell -c注入式 |
| CWE-22 | 路径遍历 | tar解压未过滤式/用户输入拼路径式/符号链接绕过式 |
| CWE-502 | 反序列化 | Java原生式/fastjson式/XStream式/Log4Shell JNDI式 |
| CWE-79 | XSS | 反射式/存储式/DOM式 |
| CWE-918 | SSRF | URL拼接式/重定向跟随式/DNS重绑定式 |
| CWE-352 | CSRF | 无Token式/弱Token式/Referer缺失式 |
| CWE-287 | 认证缺陷 | 口令硬编码式/弱口令策略/会话固定式 |
| CWE-862 | 授权缺失 | 路径无鉴权式/水平越权IDOR式/垂直越权式 |
| CWE-798 | 硬编码凭证 | 口令/APIKey/Token/连接串式 |
| CWE-327 | 弱加密算法 | MD5/SHA1/DES/RC4式 |
| CWE-330 | 弱随机数 | new Random/Math.random式 |
| CWE-209 | 信息泄露 | 详细错误回显/调试端点/版本信息式 |
| CWE-532 | 日志泄露 | 日志打印请求体/敏感字段式 |
| CWE-611 | XXE | DocumentBuilder/SAXParser式 |
| CWE-434 | 文件上传 | 无类型白名单/路径穿越式 |
| CWE-94 | 表达式注入 | SpEL/OGNL/EL/MVEL式 |
| CWE-1336 | SSTI | Freemarker/Velocity/Thymeleaf式 |
| CWE-74 | 通用注入 | CRLF/邮件头/XPath/LDAP/NoSQL式 |
| CWE-1004 | 口令技术控制 | 口令复杂度不足/无强度校验式 |

**反幻觉约束**: 报告占位符必替换 / 覆盖矩阵不可 Override / 盲区必须显式列 / 修复建议必基于实际代码 / 跟踪表类型必含三列(编号+名称+子类型), 名称必从映射表查填

### Phase 9 — 模式进化闭环 (`pattern-evolve/SKILL.md`)

**设计意图**: Phase 4b 产出的 insights.md 是"候选新模式", Phase 9 把经验证的晋升为永久模式写 `_learned/`, 同时剔除 stale。**跨日累积**是持续审计关键。

**核心工作流**:
1. 合并跨日 insights [T0] —— 读 baseline/last_insights.md + 本次新产
2. 4 项晋升门槛评估 [T1]:
   - ① 差分证明充分(检 ReAct 链 [T1]基线+[T2]执行+[T1]对比)
   - ② 跨≥2 日期命中(查合并后 hit_count 来自不同日期)
   - ③ 未匹配现有模式(对比 _index.md 条件触发表)
   - ④ 累积 hit ≥ 2
3. 用户审批 [T0] —— 4/4 通过提示(批准/拒绝/暂存)
4. 写 _learned/ + 更新 _index.md + 更新 attack-hypotheses.md + 三库一致性检查
5. 自净机制 [T0] —— hit≥5 升 high / 连续 5 次未命中降 medium / 连续 10 次标 stale / 连续 15 次问用户归档
6. 更新 baseline 供下次 [T0] —— last_insights.md + hit_count.json + stale_patterns.json + suite_version.json

**反幻觉约束**: Insights 不许跳过 4 项门槛 / 用户拒绝后不许再晋升 / 自净剔除不许隐藏 / _learned 模式 8 段格式必完整

---

## 第五部分: 持续审计(跨 Session 调度)

### 设计原则: 不在会话内 loop, 在时间维度展开

### 持续审计双部分

**第一部分 会话内增量审计**(已有 mode=increment):
- `git diff --name-only origin/main..HEAD` 取变更文件
- git diff 调用图邻居深度 2
- manifest 缩减至 ~5%
- 全 Phase 1-9 只审 5%, 总成本从 5 亿降到 1.5 亿 token

**第二部分 跨 Session 调度 loop**(新增配套):
- **基线 diff 协议** `BASELINE_DIFF.md`: 每次 full 完成后复制核心产物到 `baseline/{date}/`, 下次 increment 对比当前 vs baseline 报新增/修复/仍存在未修
- **调度触发**(写在 SKILL.md 入口 + README 模板):
  - A 手动增量: 用户主动调
  - B CI 集成: PR merge 后自动触发 GitHub Action(阻塞 C1)
  - C 定时调度: cron 02:00 / Kubernetes CronJob
- **跨会话进化累积**: Phase 9 读 baseline/last_insights.md 合并本日 insight 累加 hit_count, 4 项晋升门槛重新评估

###每月强制 full 重审

防 baseline 漂移漏报累积, 定期地毯式重做。

### Token 硬上上限

```
total_token_limit: 50亿
phase2b_semantic(若启用): 30亿上限
loop_rounds_per_loop: 3
loop_token_per_loop: 50k
total_loop_tokens: 5亿
```

触上上限写 budget_exceeded.md 阻塞报告, 强制停止不强行跑完。

### 反幻觉约束: 增量审计不许降级 baseline(基线仅 diff 对比, 当日审计结果始终权威)

---

## 第六部分: harness 工程设计哲学(防会话压缩丢数据)

借鉴 loop engineering + GenCPT 断点续传精神, 但**主体不 loop**(责任链 + 成本 + 句法不适合), 只在局部用 loop。

### 三个局部 loop(≤3 轮)

| 局部 loop | 触发 | 验收方 | 停止 |
|-----------|------|--------|------|
| 1b 鉴权面遗漏反查 | 1b 完成后 | 独立子代理反查是否有遗漏全局过滤 | 新增 ≤0 或 ≤3 轮 |
| 2b uncertain 扩读 | 2b uncertain 命中 | 独立子代理扩读 ±100 行 | verdict 收敛或 ≤3 轮 |
| 6 对抗仲裁 | 6 第一轮挑战争议 | 第三方仲裁子代理 | 共识达成或 ≤3 轮 |

每轮 self-contained, 下轮子代理只读上轮结论+源码不读推理。每轮必输出新 Read 证据防 LLM 持续加固幻觉。

### 禁用 loop 的核心判定

- Phase 4a 证伪条件检查不许循环(单次)
- Phase 4b ReAct 已是循环, 不嵌套 loop
- Phase 5 链构建不许重跑命中重来
- C1/C2/C3 升级必须人审 Echo, 不许 loop 自升

### 防会话压缩丢数据三层兜底

| 层 | 机制 |
|---|------|
| 1 | 任何 trace 立即写盘, 不靠 LLM 上下文维持(audit_log.json / sibling_scan_log.json / chunk_manifest.json 每条立即落盘) |
| 2 | 子代理下个 WU 不依赖上轮记忆, 只重新读磁盘状态 |
| 3 | QA 第一层强制校验: audit_log + manifest + sibling_scan_log 完整性, 无对应条目即拒收报告 |

设计哲学:"内存不可信, 磁盘可信"原则贯穿全 Pipeline。

### 全局按需加载原则

| 维度 | 加载策略 |
|------|---------|
| 规则库(2a/4a) | 仅 sink_type/候选命中规则文件, 不预载全 vuln-rules/ |
| 模式库(4a) | 仅条件触发表命中的模式 8 段 SKILL.md, 不读全部 25+ |
| Phase 2b 分组 | 仅读该组规则(G1-*.md), 不读其他分组 |
| 图谱(全Phase) | 按 sink/entry/candidate 查询过滤, 不全量载 knowledge_graph/ |
| 子代理 WU | 输入明确指定路径, 不依赖会话记忆 |

---

## 第七部分: 子技能清单

### 主线 13 个(默认 Pipeline)

| Skill | Phase | 可独立运行 |
|-------|------|---------|
| source-map | 0 | 否(必在 Pipeline) |
| entry-recon | 1a | 是 |
| authz-map | 1b | 是 |
| product-context | 1c | 是(可选) |
| vuln-rules | 2a | 是 |
| cross-ref | 3 | 否(依赖前阶) |
| attack-pattern | 4a | 否 |
| attack-reasoning | 4b | 否 |
| chain-builder | 5 | 否 |
| chain-verify | 6 | 否 |
| poc-generator | 7 | 否(依赖6) |
| report-finding | 8a | 否 |
| report-chain | 8b | 否 |
| report-summary | 8d | 否 |

### 可选 2 个(--with-compliance)

| Skill | Phase | 可独立使用 |
|-------|------|---------|
| compliance-audit | 2b | 是(完全独立) |
| compliance-report | 8c | 否(依赖2b) |

### 进化 1 个

| Skill | Phase | 可独立运行 |
|-------|------|---------|
| pattern-evolve | 9 | 是(跨会话累积) |

### 共享规范 7 个(非skill,被引用)

| 文件 | 用途 |
|------|------|
| SRC_ACCESS.md | 文件读取/分块/按需加载/临时方法级图谱 |
| OUTPUT_STANDARD.md | 输出格式/五态/图谱JSON |
| OFFSET_RATING.md | T0-T3 审计深度分层 |
| CHAIN_STANDARD.md | 链节点格式/可达性 |
| LOOP_POLICY.md | 局部 loop 策略/硬停止 |
| TRACKING_SHEET.md | Excel 跟踪表字段定义+类型映射表 |
| BASELINE_DIFF.md | 基线对比协议(持续审计) |

---

## 第八部分: 反幻觉约束全景(56 条)

> 全局 10 条继承 GenCPT; SourceCPT 扩展 49 条(原编号 11-59); 其中编号 12-14 为旧 grep 版专属约束语义审改造后由 15-17 取代保留占位以保持编号稳定性不重赋值。

### 全局继承(10 条, GenCPT 基础)

1. 不准凭记忆出攻击命令
2. 不准伪造代码行号/输出
3. 无证据不写确认态
4. 超审批立即停(已简化, 白盒无 5 级)
5. 省略词零容忍
6. 占位符必替换
7. baseline 永不替代当前
8-10. 保留 GenCPT 同前义对应白盒

### SourceCPT 扩展(Phase 专属)

| Phase | 编号 | 约束 |
|-------|------|------|
| 5/6 | 11 | 不许跳过 Sibling-Scan |
| 2b | 15-17 | 不许只扫字面量关键字 / 每函数必标审计结果 / 跨块切断违规不许直接判 false |
| 2a | 18-21 | 不许凭 sink 直判 VULN / 不许跳过证伪条件 / propagation 每跳必 Read / source 不可控不许硬判 |
| 3 | 22-23 | 三库新假设必引原文 / 静态库未命中不许跳过 LLM 推理 |
| 4a | 24-26 | 不许凭模式命中直判 C1 / 证伪必检 / 模式不命中不许跳过推理 |
| 4b | 27-29 | ReAct 链必完整 / 推理新候选标 llm_reasoning / insights 不许直接晋升 |
| 全局 | 30-31 | 不许循环重跑已完成 Phase / 增量审计不许降级 baseline |
| 5 | 32-35 | 链深超 5 必报截断 / sibling-scan 禁跳兄弟 / 已扫兄弟必留 audit_state / 推翻原候选不许隐藏 |
| 6 | 36-41 | 挑战者不许读分析方推理 / 六项逐项必输出 verdict / 对抗 loop 每轮必新 Read 证据 / 推翻结论不许隐藏 / C1 必须六项全 pass / SUSPENDED 不许擅升 VULN |
| 1c | 42 | 文档约束不许单独判 VULN(必配代码证据) |
| 全局 | 43 | 任何 Phase 规则库/知识库/图谱必须按需加载 |
| 8a | 44-45 | 每发现必含五段证据链 / 每产出必并出 md+json |
| 7 | 46-49 | C3/NOVULN/SUSPENDED 不许生成 POC / POC 不许填真实凭证 / payload 必沿 evidence_chain 验证可达 / 非 Web 入口不许伪 HTTP 包 |
| 8d | 50-53 | 占位符必替换 / 覆盖矩阵不可 Override / 盲区必须显式列 / 修复建议必基于实际代码 |
| 9 | 54-57 | Insights 不许跳过 4 项门槛 / 拒绝后不许再晋升 / 自净剔除不许隐藏 / _learned 模式 8 段必完整 |
| 8d | 58-59 | 跟踪表必含完整字段, 状态初值"未修复"不许留空 / 跟踪表类型必含三列(编号+名称+子类型), 名称必从映射表查填 |

---

## 第九部分: 交付物清单

```
<sourcecpt-session>/
├── session_config.json              # 含 env_fingerprint + budget + dispatch_strategy
├── manifest.json                    # 全文件清单 + state
├── dispatch_plan.json              # 分批方案
├── progress.json                    # Phase 进度状态机
├── audit_log.json                   # 每 WU 落盘状态
├── chunk_manifest.json              # Phase 2b 分块状态(若启用)
├── sibling_scan_log.json            # 横向排查落盘(若触发)
├── evidence/
│   ├── phase1a/ ─ phase9/          # 每 Phase 候选/证据
│   │   ├── CAND-*-*.md + .json     # 候选
│   │   ├── DISPROVED-*-*.md+.json  # 证伪项
│   │   ├── CHAIN-*-*.md + .json    # 链
│   │   ├── VULN-*-*.md + .json     # C1 实证
│   │   ├── CLUE-*-*.md + .json     # C3 线索
│   │   ├── NOVULN-*-*.md+.json     # 推翻
│   │   ├── SUSPENDED-*-*.md+.json   # 争议待人工
│   │   ├── POC-*-*.md + .json + .http + .sarif.json
│   │   ├── insights.md             # Phase 4b 产出
│   │   ├── phase6_loop/            # 对抗 loop 状态
│   │   └── sibling_scans/          # 横向排查明细
├── reports/
│   ├── pentest_report.md + .json + .sarif   # 全景报告三轨
│   ├── compliance_report.md+.json            # 可选
│   ├── chain_report.md + .json
│   ├── patch_suggestion.md + .json
│   ├── qa_report.md + .json
│   ├── issue_tracking.csv + .xlsx            # Excel 跟踪表
│   └── budget_exceeded.md                    # 触硬上限时
├── knowledge_graph/
│   ├── nodes/
│   │   ├── entries.json            # 入口面
│   │   ├── sinks.json              # sink 索引
│   │   ├── modules.json            # 模块骨架
│   │   ├── authz_configs.json      # 鉴权配置
│   │   ├── constraints.json        # 业务约束(若启用 1c)
│   │   ├── compliance_violations.json(若启用 2b)
│   │   ├── vuln_candidates.json    # 漏洞候选(C3 起步)
│   │   ├── pattern_matches.json    # 模式匹配结果
│   │   ├── chains.json             # 漏洞链
│   │   └── verified_vulns.json      # Phase 6 判定终态
│   └── edges/
│       ├── calls_module.json
│       ├── entry_in_module.json
│       ├── entry_covered_by.json
│       ├── data_flow.json
│       ├── cross_ref.json
│       ├── upgrade.json
│       ├── chain_steps.json
│       ├── sibling_of.json
│       ├── business_flow.json(若启用 1c)
│       └── constraint_to_code.json(若启用 1c)
└── baseline/
    ├── {date}/                    # 历史基线
    ├── last_insights.md           # 跨日累积 insights
    ├── hit_count.json             # 学习模式命中累积
    ├── stale_patterns.json        # 待归档清单
    └── suite_version.json         # 版本兼容检查
```

---

## 第十部分: 已知限制与盲区(诚实声明)

| 盲区 | 原因 | 缓解 |
|------|------|------|
| 业务逻辑漏洞(状态机/价格篡改) | 无静态 sink | Phase 1c(可选)+ 4b 推理+ Top3 盲区显式列 |
| 跨仓库调用链 | Phase 0 单仓工程图 | 报告标"跨仓未覆盖"Top3 |
| 资源竞争/时序 | 静态分析范畴外 | 显式声明静态范畴 |
| 全量 100% 漏报 | 软件安全客观上限 | 覆盖率可度量 + 盲区 Top3 显式列(不承诺100%) |
| 零误报 | LLM 推理编造 | Phase 6 挑战者+假阳性库(不承诺零) |
| 文档与代码不一致 | 文档不可信 | confidence 标注 + 不许文档单独判 VULN |
| 全量语义审成本 | 300w 行几亿 token | 增量模式(默认)+ 首次 full(月1次) |

**框架承诺**: 已识别攻击面 100% 有结论, 行覆盖率 100% 可度量, 覆盖矩阵可视化, 未知攻击面盲区 Top3 显式列。

**框架不承诺**: 100% 漏洞找全(软件安全客观上限) / 零误报(LLM 推理性质)。

---

## 第十一部分: 调度方式

### 入口 SKILL.md 调度方法参数

| 参数 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `project-path` | string | **是** | - | 源码根目录 |
| `mode` | enum | 否 | `full` | `fast`/`full`/`increment` |
| `scope` | enum | 否 | `all` | `java/go/python/js/ts/all` 或 `module=<path>` 或 `changed` 或 `owasp-a1..a10` |
| `changed-since` | string | 否 | - | increment 模式 git ref |
| `context-path` | string | 否 | - | 启用 Phase 1c 产品上下文 |
| `--with-compliance` | flag | 否 | - | 启用 Phase 2b + 8c 合规 |
| `chain-depth` | int | 否 | 5 | 链最大深度硬上限 8 |
| `changed-since` | string | 否 | - | 增量模式 git ref |

### 调度模式

| mode | 行为 | 规模适配 |
|------|------|---------|
| `fast` | 跳 Phase 3-7, 只 P0+P2a+P8a+P8d 出基线 | 快速摸底 |
| `full` | 全 Phase 0-9 | 单模块 ≤30w 行首次全审 |
| `increment` | 读 P0 索引 + git diff + 调用图邻居 | 300w 日常复检 |

### 持续审计渠道

- A 手动增量: 用户主动调用
- B CI 集成(推荐企业): PR merge 后自动触发 GitHub Action, C1 阻塞 merge
- C 定时调度(推荐 300w 项目): `cron 0 2 * * * opencode run --skill sourcecpt --mode increment --changed-since last-night --project /repo --output /audit/$(date)`

### baseline 模式

- `--baseline <path>`: 历次基线目录路径,用于 diff 对比(suite_version 一致才直接 diff,否则降级趋势对比)

---

## 第十二部分: 借鉴与原创标注(透明追溯)

| 机制/约束 | 起源 | 性质 |
|---------|------|------|
| Pipeline 编排+知识图谱 | GenCPT | 借鉴 |
| 反幻觉1-10 全局 | GenCPT | 借鉴 |
| 五态标记 | GenCPT | 借鉴 |
| C1/C2/C3 分级 | GenCPT | 借鉴 |
| 模式库 8 段+条件触发表 | GenCPT | 借鉴 |
| 模式进化 4 门槛+自净 | GenCPT | 借鉴增强(跨日hit_count) |
| QA 三层+覆盖矩阵 | GenCPT | 借鉴 |
| 分层子技能架构 | v0.6.0 RuoJi6 | 借鉴+修正(反对退回单skill) |
| 六项验收门槛 | v0.6.1 RuoJi6 | 借鉴 |
| 攻击面通道枚举 | 落地材料(RuoJi6 类似项目) | 借鉴+扩到13 |
| 任务划分规则 | 落地材料 Stage3 | 借鉴 |
| 假阳性验证原则 | 落地材料 Stage5 | 借鉴+合入 non-vuln-scenarios |
| Sibling-Scan 横向排查 | SourceCPT 原创(用户提需求设计) | 原创 |
| 13 入口+28 sink 双轨索引 | SourceCPT 原创 | 原创 |
| 局部 loop+硬停止 | loop engineering 部分采纳 | 借鉴改造 |
| 方法级图谱临时展开 | SourceCPT 原创 | 原创 |
| 持续审计跨日调度 | loop engineering 时间维度变体 | 原创改造 |
| 合规降为可选插件 | 用户质疑+反思 | 修正 |
| md+json+SARIF 三轨交付 | 用户提出 | 修正 |
| Excel 跟踪表类型三列 | 用户提出 | 修正 |
| 证据链五段(证据/原理/影响/防护/复现) | v0.6.1 六项门槛+扩展 | 借鉴+扩展 |
| patch-patterns 修复建议 | 借鉴材料 | 借鉴 |
| 假阳性场景库 | 落地材料+v0.6.1 合并 | 借鉴+原创 |
| Token 硬停止预算 | loop engineering 借鉴 | 借鉴改造 |
| POC 生成无门禁 | 用户提(白盒POC非武器) | 修正(去掉门禁) |
| SARIF 输出 | 工业标准借鉴 | 借鉴 |
| 文档约束可信度分级(1c) | 用户质疑"文档不可信" | 修正 |

---

## 附录 A: harness 机制清单(8类32项, SourceCPT 自有分类)

按问题维度归类, 表彰设计本身可追溯:

### A.1 Evidence Governance(证据治理) 4 项

| id | name | problem | principle | origin | phases | file |
|---|------|---------|-----------|--------|--------|------|
| write-to-disk-first | 写盘优先 | 子代理会话压缩丢数据 | WU 产立落盘不靠记忆 | GenCPT继承 | 全局 | SRC_ACCESS.md |
| graph-decouple | 图谱解耦 | Phase 间依赖 | 磁盘 knowledge_graph 传 | GenCPT继承 | 全局 | OUTPUT_STANDARD.md |
| breakpoint-resume | 断点续传 | 崩溃重跑成本爆炸 | progress.json 状态机从最后 WU 恢复 | GenCPT继承 | 全局 | progress.json |
| on-demand-read | 按需读取 | 上下文压缩丢数据 | 下游按需 Read 磁盘非依赖会话 | GenCPT继承 | 全局 | SRC_ACCESS.md |

### A.2 Harness Engineering(外壳工程) 3 项

| id | name | problem | principle | origin | phases |
|---|------|---------|-----------|--------|--------|
| wu-batching | WU 分批 | 单子代理跑不动大项目 | 工作单元按 Phase 切片 | GenCPT继承 | 2a/2b/4a/5/6 |
| must-input-output | MUST 输入/输出契约 | 上下游契约不清 | 子 SKILL 声明契约, QA 第一层校验 | GenCPT继承 | 全 Phase |
| phase-checkpoint | Phase 检查点 | 下游乱跑 | 前置缺失跳过并记日志 | GenCPT继承 | 全 Phase |

### A.3 Scale Adaptation(规模适配, 白盒专属) 5 项

| id | name | problem | principle | origin | phases |
|---|------|---------|-----------|--------|--------|
| manifest-line-coverage | manifest 行覆盖率门 | 无覆盖率结构 | manifest 全清单, QA 第一层扫残留 [ ] | SourceCPT 原创 | 全局 |
| intelligent-dispatch | 智能派发 | 300w 无分批策略崩溃 | 按体量自适应切批 | 落地材料借鉴+扩展 | 0 |
| modular-graph-temp-expand | 方法级图谱临时展开 | 全方法图 Gb 内存爆, 模块图精度差 | 持久模块级, sink 触达时临时建 5 跳方法链用后弃 | SourceCPT 原创 | 4a |
| file-chunking | 文件分块 | 单文件大被截断 | 500 行/块, 函数边界对齐, 5 行重叠 | SourceCPT 原创 | 2b |
| increment-mode | 增量模式 | 300w 不可能每日全审 | git diff + 调用图邻居 ~5% | loop 工程借鉴 | 全局 |

### A.4 Anti-Hallucination(反幻觉体系, 白盒扩展) 4 类

| id | name | problem | principle | origin | phases |
|---|------|---------|-----------|--------|--------|
| anti-hallucination-system | 56 条反幻觉约束 | LLM 编造漏洞证据 | 举证责任在 LLM, 证据链不完整降级 | GenCPT 10+扩展 46(含 3 占位) | 全局 |
| disproof-mandatory-check | 证伪必检 | 模式命中不一定是漏洞 | 规则第 6 段证伪必检查 | SourceCPT 原创 | 2a/4a |
| non-vuln-scenarios-filter | 假阳性场景库前置 | 误报率高 | VULN 判定前对照库, 命中即 [-] | 落地材料+原创 | 6 |
| triple-library-association | 三库联动 | 静态库不覆盖组合 | 三库各自静态+LLM 动态补盲 | GenCPT 借鉴 | 3 |

### A.5 Discovery Amplification(发现放大, 白盒专属) 4 项

| id | name | problem | principle | origin | phases |
|---|------|---------|-----------|--------|--------|
| sibling-scan-transversal | 三轴 Sibling-Scan | 同类漏报 | 确认漏洞触发同 sink/entry/路径三轴兄弟全复查 | SourceCPT 原创 | 5/6 |
| dual-track-index | 双轨索引 | 漏面 | 13 入口 × 28 sink × OWASP × CWE 四维 | SourceCPT 原创 | 0/4a |
| five-state-closure | 五态闭环 | [ ] 被默默跳过 | 强制消灭空态, QA 校验 | GenCPT 继承 | 全局 |
| coverage-matrix | 覆盖矩阵 | 漏报不可见 | 矩阵空白即漏报可见 | GenCPT 继承 | 8d |

### A.6 Quality Gate(质量门禁) 5 项

| id | name | problem | principle | origin | phases |
|---|------|---------|-----------|--------|--------|
| qa-three-layer | QA 三层 | 各类_in漏 | 结构/语义/覆盖, 第三层不可 Override | GenCPT 继承 | 8d |
| evidence-chain-five-sections | 证据链五段强制 | 漏洞判据不全 | Source/San/Prop/Sink/Disproof 缺一拒收 | SourceCPT 原创+扩展 | 2a/4a/6 |
| c1-c2-c3-confidence | 三态可信度 | 主观分级 | 由证据决定不由位置 | GenCPT 继承 | 6 |
| six-gates-verification | 六项验收门槛 | C1 无客观标准 | 可达/可控/可传播/可利用/可复现/影响 | v0.6.1 借鉴 | 6 |
| challenger-independent-review | 挑战者独立复审 | 单一 LLM 自合理化 | 独立子代理不看推理只看结论+源码 | v0.6.1+GenCPT | 6 |

### A.7 Evolution & Continuity(进化持续, 白盒扩展) 4 项

| id | name | problem | principle | origin | phases |
|---|------|---------|-----------|--------|--------|
| pattern-evolution-loop | 模式进化闭环 | 静态库滞后 | 4 项门槛+_learned/ | GenCPT 增强跨日 | 9 |
| cross-day-insights | 跨日 insights 累积 | 单 Session 学不到 | 跨日 hit_count, 晋升仍走门槛 | SourceCPT 改造 | 9 |
| baseline-no-override | Baseline 不替代 | 增量漏报 | baseline 仅作 diff 对比 | GenCPT 继承 | 增量审计 |
| continuous-audit-schedule | 持续审计调度 | 300w 日常不可全审 | cron/CI mode=increment | loop 借鉴改造 | 全局 |

### A.8 Safety Control(安全控制, 白盒简化) 3 项

| id | name | problem | principle | origin | phases |
|---|------|---------|-----------|--------|--------|
| token-hard-stop | Token 硬停止 | LLM 不限跑爆 | 50 亿总量, 触阻塞报告 | loop 借鉴 | 全局 |
| sibling-scan-hard-limit | Sibling-Scan 防爆 | 横扫瓶颈 | 深度≤2/200/1 亿上上限 | SourceCPT 原创 | 5/6 |
| doc-confidence-rating | 文档可信度分级 | 文档撒谎误导 | high/medium/low/unverified, 不许单独判 VULN | 用户提修正 | 1c |

---

## 附录 B: 13 入口 + 28 sink 完整清单见正文第三部分。

---

## 附录 C: 反幻觉 56 条完整列表见正文第八部分。