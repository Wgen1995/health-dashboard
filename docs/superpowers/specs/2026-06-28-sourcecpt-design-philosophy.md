# SourceCPT 设计哲学

> 将 8 类 32 项核心机制以速查结构陈列,彰显设计体系, readers 可不读全文而掌握全貌。每节独立可查,深度按 §9 详述。

## 速查导航

| 跳转 | 内容 |
|------|------|
| §1 设计哲学总览表 | 8 类 32 项, 一表看全 |
| §2 机制分类矩阵 | 按问题维度分类, 速查某问题对应机制 |
| §3 反幻觉约束全景表 | 59 条按 Phase 分类 |
| §4 Pipeline 9 阶段全景表 | 阶段-输入-产出-预算 |
| §5 子技能清单矩阵 | 17 skill 一查 |
| §6 知识库/规则库一览 | 各库文件用途 |
| §7 交付物清单矩阵 | md/json/sarif/graph 一查 |
| §8 数据流状态机 | Phase 间数据流向 |
| §9 设计哲学详述 | 每机制独立可读 section |
| §10 借鉴与原创标注 | 透明追溯 |
| §11 已知限制与盲区 | 诚实边界声明 |

---

## §1 设计哲学总览表

| id | name | category | problem | principle | origin | phases | file |
|---|------|---------|---------|-----------|--------|--------|------|
| write-to-disk-first | 写盘优先 | Evidence Governance | 子代理会话压缩丢数据 | WU 产立落盘不靠记忆 | GenCPT 继承 | 全局 | SRC_ACCESS.md |
| graph-decouple | 图谱解耦 | Evidence Governance | Phase 间依赖会话传 | 磁盘 knowledge_graph 传 | GenCPT 继承 | 全局 | OUTPUT_STANDARD.md |
| breakpoint-resume | 断点续传 | Evidence Governance | 崩溃重跑成本爆炸 | progress.json 状态机从最后 WU 恢复 | GenCPT 继承 | 全局 | progress.json |
| on-demand-read | 按需读取 | Evidence Governance | 上下文压缩丢数据 | 下游 Read 磁盘非依赖会话记忆 | GenCPT 继承 | 全局 | SRC_ACCESS.md |
| wu-batching | WU 分批 | Harness Engineering | 单子代理跑不动 300w | 按 Phase 切片单 WU 一子任务 | GenCPT 继承 | 2a/2b/4a/5/6 | SRC_ACCESS.md |
| must-input-output | MUST 输入/输出 | Harness Engineering | 上下游契约不清 | 子 SKILL 声明契约, QA 第一层校验 | GenCPT 继承 | 全 Phase | 各 SKILL.md |
| phase-checkpoint | Phase 检查点 | Harness Engineering | 下游乱跑 | 前置缺失跳过并记日志 | GenCPT 继承 | 全 Phase | OUTPUT_STANDARD.md |
| manifest-line-coverage | manifest 行覆盖率门 | Scale Adaptation | 无覆盖率结构 | 全清单 + QA 第一层扫残留 [ ] | SourceCPT 原创 | 全局 | manifest.json |
| intelligent-dispatch | 智能派发 | Scale Adaptation | 300w 无分批策略 | 按体量自适应切批 | 落地材料借鉴+扩 | 0 | dispatch_plan.json |
| modular-graph-temp-expand | 方法级临时展开 | Scale Adaptation | 全方法图 Gb 内存爆 | 模块级持久, sink 触达临时 5 跳方法链用后弃 | SourceCPT 原创 | 4a | SRC_ACCESS.md |
| file-chunking | 文件分块 | Scale Adaptation | 大文件被截断 | 500 行/块, 函数边界, 5 行重叠 | SourceCPT 原创 | 2b | SRC_ACCESS.md |
| increment-mode | 增量模式 | Scale Adaptation | 300w 不能每日全审 | git diff + 调用图邻居 ~5% | loop 借鉴改造 | 全局 | BASELINE_DIFF.md |
| anti-hallucination-system | 59 条反幻觉约束 | Anti-Hallucination | LLM 编造漏洞证据 | 举证责任在 LLM, 证据不完整降级 | GenCPT 10+49 扩展 | 全局 | ANTI_HALLUCINATION.md |
| disproof-mandatory-check | 证伪必检 | Anti-Hallucination | 模式命中不一定是漏洞 | 规则第 6 段证伪必检查 | SourceCPT 原创 | 2a/4a | vuln-rules/* |
| non-vuln-scenarios-filter | 假阳性库前置 | Anti-Hallucination | 误报高 | VULN 判定前对照库命中即 [-] | 落地材料+原创 | 6 | non-vuln-scenarios.md |
| triple-library-association | 三库联动 | Anti-Hallucination | 静态库不覆盖组合 | 三库静态 + LLM 补盲 | GenCPT 借鉴 | 3 | hypothesis-libraries/ |
| sibling-scan-transversal | 三轴 Sibling-Scan | Discovery Amplification | 同类漏报 | 确认触发同 sink/entry/路径三轴兄弟全复查 | SourceCPT 原创 | 5/6 | OUTPUT_STANDARD.md |
| dual-track-index | 双轨索引 | Discovery Amplification | 漏面 | 13 入口 × 28 sink × OWASP × CWE 四维 | SourceCPT 原创 | 0/4a | entry-types/sink-types.md |
| five-state-closure | 五态闭环 | Discovery Amplification | [ ] 被盛盛跳过 | 强制消灭空态, QA 校验 | GenCPT 继承 | 全局 | OUTPUT_STANDARD.md |
| coverage-matrix | 覆盖矩阵 | Discovery Amplification | 漏报不可见 | 矩阵空白即漏报可见 | GenCPT 继承 | 8d | QA_OVERRIDE.md |
| qa-three-layer | QA 三层 | Quality Gate | 各类漏 | 结构/语义/覆盖, 第三层不可 Override | GenCPT 继承 | 8d | QA_OVERRIDE.md |
| evidence-chain-five | 证据链五段强制 | Quality Gate | 漏洞判据不全 | Source/Prop/San/Sink/Disproof 缺一拒收 | SourceCPT 原创+扩展 | 2a/4a/6 | OUTPUT_STANDARD.md |
| c1-c2-c3-confidence | 三态可信度 | Quality Gate | 主观分级 | 由证据决定不由位置 | GenCPT 继承 | 6 | OFFSET_RATING.md |
| six-gates-verification | 六项验收门槛 | Quality Gate | C1 无客观标准 | 可达/可控/可传播/可利用/可复现/影响 | v0.6.1 借鉴 | 6 | OFFSET_RATING.md |
| challenger-independent | 挑战者独立复审 | Quality Gate | 单一 LLM 自合理化 | 不看推理只看结论+源码 | v0.6.1+GenCPT | 6 | chain-verify/SKILL.md |
| adversarial-loop | 对抗 loop 仲裁 | Quality Gate | 争议未决 | ≤3 轮第三方仲裁, 每轮 self-contained | loop 借鉴改造 | 6 | LOOP_POLICY.md |
| triple-track-delivery | md+json+SARIF 三轨交付 | Quality Gate | 不同消费者 | 人读+程序消费+CI 集成 | 工业标准借鉴 | 8 | TRACKING_SHEET.md |
| pattern-evolution-loop | 模式进化闭环 | Evolution & Continuity | 静态库滞后 | 4 门槛 + _learned/ | GenCPT 增强跨日 | 9 | pattern-evolve/SKILL.md |
| cross-day-insights | 跨日 insights 累积 | Evolution & Continuity | 单 Session 学不到 | 跨日 hit_count, 晋升仍走门槛 | SourceCPT 改造 | 9 | last_insights.md |
| baseline-no-override | Baseline 不替代 | Evolution & Continuity | 增量漏报 | baseline 仅 diff 对比, 当日权威 | GenCPT 继承 | 增量审计 | BASELINE_DIFF.md |
| continuous-audit | 持续审计调度 | Evolution & Continuity | 300w 不可每日全审 | cron/CI mode=increment | loop 借鉴改造 | 全局 | 入口 SKILL.md |
| token-hard-stop | Token 硬停止 | Safety Control | LLM 不限跑爆 | 50 亿总量, 触阻塞报告 | loop 借鉴 | 全局 | LOOP_POLICY.md |
| sibling-scan-limit | Sibling-Scan 防爆 | Safety Control | 横扫瓶颈 | 深度≤2/200/1亿 | SourceCPT 原创 | 5/6 | chain-builder/SKILL.md |
| doc-confidence-rating | 文档可信度分级 | Safety Control | 文档撒谎误导 | high/medium/low, 不许单判 | 用户提修正 | 1c | product-context/SKILL.md |

---

## §2 机制分类矩阵(按问题查)

| 问题大类 | 机制 | 何时启用 |
|---------|------|---------|
| **防漏报** | five-state-closure / sibling-scan-transversal / pattern-evolution-loop / modular-graph-temp-expand / dual-track-index | 全局 / 5&6 确认时 / 9 跨日累积 / 4a 每次污点追踪 / 0 识别 |
| **防误报** | anti-hallucination-system / c1-c2-c3-confidence / challenger-independent / non-vuln-scenarios-filter / doc-confidence-rating | 全局 / 6 / 6 / 6 / 1c |
| **防成本爆炸** | breakpoint-resume / token-hard-stop / sibling-scan-limit / increment-mode | 全局 / 全局 / 5&6 / 持续审计模式 |
| **防会话压缩丢数据** | write-to-disk-first / graph-decouple / on-demand-read / phase-checkpoint | 全局 |
| **防模式库滞后** | pattern-evolution-loop / cross-day-insights | 9 |
| **防责任真空** | token-hard-stop / challenger-independent | 全局 / 6 |

---

## §3 反幻觉约束全景表(59条)

### 全局继承(10条, GenCPT基础)

| # | 约束 |
|---|------|
| 1 | 不准凭记忆出攻击命令 |
| 2 | 不准伪造代码行号/输出 |
| 3 | 无证据不写确认态 |
| 4 | 超审批立即停(白盒已简化无5级) |
| 5 | 省略词零容忍 |
| 6 | 占位符必替换 |
| 7 | baseline 永不替代当前 |
| 8-10 | 保留 GenCPT 同前义对应白盒 |

### SourceCPT 扩展(Phase 专属)

| Phase | 编号 | 约束 |
|-------|------|------|
| 5/6 | 11 | 不许跳过 Sibling-Scan |
| 2b | 15-17 | 不许只扫字面量 / 每函数必标审计结果 / 跨块切断不许直接判 false |
| 2a | 18-21 | 不许凭 sink 直判 / 证伪必检 / propagation 必 Read / source 不可控不许硬判 |
| 3 | 22-23 | 三库新假设必引原文 / 静态库未命中不许跳过 LLM 推理 |
| 4a | 24-26 | 不许模式命中直判 C1 / 证伪必检 / 模式不命中不许跳过推理 |
| 4b | 27-29 | ReAct 链必完整 / 新候选标 llm_reasoning / insights 不直接晋升 |
| 全局 | 30-31 | 不许循环重跑已完成 Phase / 增量审计不许降级 baseline |
| 5 | 32-35 | 链深超 5 必报截断 / sibling-scan 禁跳兄弟 / 已扫兄弟必留 audit_state / 推翻原候选不许隐藏 |
| 6 | 36-41 | 挑战者不许读分析方推理 / 六项逐项必输出 verdict / 对抗 loop 每轮必新 Read 证据 / 推翻结论不许隐藏 / C1 必须六项全 pass / SUSPENDED 不许擅升 VULN |
| 1c | 42 | 文档约束不许单独判 VULN(必配代码证据) |
| 全局 | 43 | 规则库/知识库/图谱必须按需加载 |
| 8a | 44-45 | 每发现必含五段证据链 / 每产出必并出 md+json |
| 7 | 46-49 | C3/NOVULN/SUSPENDED 不许生成 POC / POC 不许填真实凭证 / payload 必沿 evidence_chain 验证 / 非 Web 入口不许伪 HTTP 包 |
| 8d | 50-53 | 占位符必替换 / 覆盖矩阵不可 Override / 盲区必须显式列 / 修复建议必基于实际代码 |
| 9 | 54-57 | Insights 不许跳过 4 门槛 / 拒绝后不许再晋升 / 自净剔除不许隐藏 / _learned 模式 8 段必完整 |
| 8d | 58-59 | 跟踪表必含完整字段状态初值"未修复"不许留空 / 类型必含三列(编号+名称+子类型)名称必从映射表查填 |

---

## §4 Pipeline 9 阶段全景表

| Phase | 名称 | 输入 | 产出 | WU数估 | Token估 | max3耗时 |
|-------|------|------|------|-------|-------|---------|
| 0 | 工程地图 | project-path | manifest/dispatch/入口 sink 索引 | bash 脚本 | <1M | min 级 |
| 1a | 入口面 | Phase0+源码 | entries+param 树 | 50-500 | 500k-2M | 10-30min |
| 1b | 鉴权面 | Phase1a | authz_state/authz_configs | 20-100 | 500k-1M | 10-30min |
| 1c | 产品上下文(可选) | context-path | constraints | 5-30 | 100k-500k | 5-20min |
| 2a | 漏洞规则 | 图谱+源码 | vuln_candidates(C3) | 200-1500 | 2-15M | 3-8h |
| 2b | 编码合规(可选) | manifest | compliance_violations | 100 批 × 50 | ~30亿(语义版) | 几天 |
| 3 | 交叉关联 | 全图谱 | 联动+新候选 | 5-30 | 50-200k | 10-40min |
| 4a | 模式匹配 | candidate+模式库 | pattern_match(C2) | 100-500 | 1-6M | 2-6h |
| 4b | LLM 推理 | 未结案+盲区 | 新候选+insights | 50-300 | 1-6M | 2-5h |
| 5 | 漏洞链+SS | candidate | chains+sibling | 20-100 | 1-3M | 1-3h |
| 6 | 挑战+验收 | 候选+源码 | VULN/CLUE/NOVULN/SUSPENDED | 200-1000 | 2-15M | 2-10h |
| 7 | POC | C1/C2 | Burp+SARIF 片段 | 10-100 | 500k-2M | 30-90min |
| 8a | 漏洞报告 | Phase6 | report.md+.json | 1-5 | 200k | min 级 |
| 8b | 漏洞链报告 | Phase5+6 | report.md+.json | 1-5 | 200k | min 级 |
| 8c | 合规报告(可选) | Phase2b | compliance_report | 1-5 | 200k | min 级 |
| 8d | 全景+QA+修复+跟踪表 | 全产出 | pentest_report+qa+patch+xlsx | 1-3 | 500k | 10-30min |
| 9 | 模式进化 | insights+last | _learned/+stale 剔除 | 1-10 | 100k | 10-30min |

**总预算(full 300w 单语言)**: ~5亿 Token, ~8-25h
**增量审计**: ~1.5亿 Token, 2-6h (5% 文件)

---

## §5 子技能清单矩阵

### 主线 13 个(默认 Pipeline)

| Skill | Phase | 可独立运行 |
|-------|------|---------|
| source-map | 0 | 否 |
| entry-recon | 1a | 是 |
| authz-map | 1b | 是 |
| product-context | 1c | 是(可选) |
| vuln-rules | 2a | 是 |
| cross-ref | 3 | 否 |
| attack-pattern | 4a | 否 |
| attack-reasoning | 4b | 否 |
| chain-builder | 5 | 否 |
| chain-verify | 6 | 否 |
| poc-generator | 7 | 否 |
| report-finding | 8a | 否 |
| report-chain | 8b | 否 |
| report-summary | 8d | 否 |

### 可选 2 个(--with-compliance)

| Skill | Phase | 可独立使用 |
|-------|------|---------|
| compliance-audit | 2b | 是(完全独立) |
| compliance-report | 8c | 否 |

### 进化 1 个

| Skill | Phase | 可独立运行 |
|-------|------|---------|
| pattern-evolve | 9 | 是(跨会话累积) |

### 共享规范 7 个(非skill,被引用)

| 文件 | 用途 |
|------|------|
| SRC_ACCESS.md | 文件读取/分块/按需加载/临时方法级图谱 |
| OUTPUT_STANDARD.md | 输出格式/五态/图谱 JSON |
| OFFSET_RATING.md | T0-T3 审计深度分层 |
| CHAIN_STANDARD.md | 链节点格式/可达性 |
| LOOP_POLICY.md | 局部 loop 策略/硬停止 |
| TRACKING_SHEET.md | Excel 跟踪表字段+类型映射表 |
| BASELINE_DIFF.md | 基线对比协议 |

---

## §6 知识库/规则库一览

| 路径 | 内容 | 主使用 |
|------|------|---------|
| entry-types.md | 13 入口面 grep 模式 | Phase 0/1a |
| sink-types.md | 28 sink 类 grep 模式 | Phase 0/2a |
| vuln-rules/A01..A10/ | 28 漏洞规则 8 段格式 | Phase 2a |
| attack-patterns/_index.md | 模式索引+条件触发表 | Phase 4a |
| attack-patterns/*/_learned/ | 学习模式 | Phase 4a |
| hypothesis-libraries/compliance-hypotheses.md | 35 CHK-CAND | Phase 3 |
| hypothesis-libraries/attack-hypotheses.md | 25 ATK-HYP | Phase 3 |
| hypothesis-libraries/cross-ref-queries.md | 3 XREF | Phase 3 |
| compliance-rules/G1..G7/ | 7 分组 ~100 规则(可选) | Phase 2b |
| compliance-rules/enterprise/ | 企业自定义 | Phase 2b |
| non-vuln-scenarios.md | 假阳性场景库 | Phase 6 |
| patch-patterns/A01..A10/ | 修复模式库 | Phase 8d |

---

## §7 交付物清单

| 路径 | 格式 | 用途 |
|------|------|------|
| session_config.json | json | 会话配置/预算/指纹 |
| manifest.json | json | 全文件清单+state |
| dispatch_plan.json | json | 分批方案 |
| progress.json | json | Phase 进度状态机 |
| audit_log.json | json | 每 WU 落盘 |
| chunk_manifest.json | json | Phase 2b 分块状态 |
| sibling_scan_log.json | json | 横向排查落盘 |
| evidence/phase*/CAND-*.* | md+json | 候选证据 |
| evidence/phase5/CHAIN-*.* | md+json | 链证据 |
| evidence/phase6/VULN-*, CLUE-*, NOVULN-*, SUSPENDED-* | md+json | 终态 |
| evidence/phase7/POC-*.* | md+http+json+sarif | POC 包 |
| reports/pentest_report.* | md+json+sarif | 全景报告 |
| reports/patch_suggestion.* | md+json | 修复建议 |
| reports/qa_report.* | md+json | QA 校验 |
| reports/issue_tracking.* | csv+xlsx | Excel 跟踪表 |
| reports/budget_exceeded.md | md | 硬上限触发 |
| baseline/{date}/ | 复制 | 持续审计基线 |
| baseline/last_insights.md | md | 跨日累积 |
| knowledge_graph/nodes/*.json | json | 图谱持久层 |
| knowledge_graph/edges/*.json | json | 图谱关系 |

---

## §8 数据流状态机

### 流转图(文字版速查)

```
project → Phase0 → manifest
                  ↓
           Phase1a/1b/1c → entries/authz/constraints
                  ↓
           Phase2a → CAND (C3 起步)
           Phase2b (可选) → G1-01..G7 fail
                  ↓
           Phase3 联动 → 升级/新CAND
                  ↓
           Phase4a 模式 → CAND升级C2
           Phase4b LLM → 新CAND/insights
                  ↓
           Phase5 链/SS → CHAIN/CAND新增
                  ↓
           Phase6 挑战+6项验收 → VULN.C1/CLUE.C3/NOVULN/SUSPENDED
                  ↓
           Phase7 POC → Burp+SARIF片段
                  ↓
           Phase8a/8b/8c/8d → 报告+QA+Excel 跟踪
                  ↓
           Phase9 → _learned/ + baseline/last_insights.md
```

### Phase 间数据传递介质

| 来源 | 介质 | 使用方 |
|------|------|--------|
| Phase0→1a | knowledge_graph/nodes/entries.json | 1a 补齐 |
| Phase0→1b | knowledge_graph/nodes/entries.json | 1b 标 authz_state |
| Phase1a→2a | entries.json param_tree | 2a 污点追踪 source |
| Phase1b→2a | entries.authz_state | 2a 路径可达性 |
| Phase2a→3 | nodes/vuln_candidates.json | 3 联动 |
| Phase2b→3 | nodes/compliance_violations.json | 3 联动(可选) |
| Phase3→4a | 升级后 vuln_candidates.json | 4a 模式匹配 |
| Phase4a→4b | 未结案 candidate/盲区 sink | 4b 推理 |
| Phase4b→5 | 新 CAND+insights | 5 链构造 |
| Phase5→6 | nodes/chains.json + 增量 CAND | 6 挑战 |
| Phase6→7 | verified_vulns.json C1/C2 | 7 POC |
| Phase6→8 | 全产出 | 8 报告 |
| Phase9→下次 | baseline/last_insights.md | 持续审计 |

---

## §9 设计哲学详述

### §9.1 写盘优先 / 摘要上行 / 按需读取 / 图谱解耦 / 断点续传

**解决问题**: LLM 子代理会话被压缩时上下文丢失, 长流程无法用单会话跑完。

**核心原理**:
- 任何 WU 产出立即落盘 audit_log.json / sibling_scan_log.json
- Phase 间不直接传数据, 全通过 knowledge_graph 目录传
- 下游 WU 输入明确指定文件路径, 不依赖上轮记忆
- 失败时从最后成功 WU 恢复(progress.json), 不重跑已成功 Phase

**与 loop 区别**: loop 重跑不合格, 断点续传保留已成功。安全审计单次几亿 Token 重跑成本爆炸。

**为何适用**: 300w 行项目必然跨多子代理会话, 会话压缩是必然风险, "内存不可信, 磁盘可信" 是兜底原则。

**关联**: SRC_ACCESS.md, OUTPUT_STANDARD.md, progress.json

### §9.2 manifest 强制行覆盖率

**解决问题**: 无文件清单的"已审" 实际可能漏文件, LLM 不会主动承认未审。

**核心原理**: Phase 0 列全文件为 manifest.json, 每文件 state 初值 `[ ]`, WU 完成立即改 state。QA 第一层校验残留 `[ ]` 必拦截。

**为何适用**: 300w 行规模化审计的行覆盖率必须可度量, 否则漏报不可量化。

### §9.3 智能派发 + 文件分块 + 临时方法级图谱

**解决问题**: 300w 行单子代理跑不动, 大文件被截断, 全方法级图谱内存爆。

**核心原理**:
- 按体量自适应切批

### §9.4 三库联动 + 三态可信度 + 六项验收

**解决问题**: LLM 编造看似合理的漏洞证据；传统高中低危分级主观不防 LLM 自抬高危；C1 无客观标准。

**核心原理**:
- 三库(合规假设 / 漏洞假设 / XREF 查询)静态映射 + LLM 补盲
- C1/C2/C3 由证据决定不由位置, 六项门槛是 C1 硬条件

**为何适用**: 三态调和防漏报(全报)与防误报(严判)矛盾——全报但分池让甲方自选。

### §9.5 五态闭环 + 覆盖矩阵 + QA 三层校验

**解决问题**: "未检查" `[ ]` 被默默跳过致漏报；漏报不可见；各类校验被 Override。

**核心原理**:
- `[ ]` 强制消灭, QA 第一层扫残留
- 覆盖矩阵 7 OWASP × 13 入口 × 28 sink, 空白即漏报可见
- QA 第三层不可 Override, 第二层可 Override 但记原因

### §9.6 Sibling-Scan 横向排查

**解决问题**: 发现一处漏洞只报一处, 同类型 sink 在别处可能漏。

**核心原理**: 每确认 C1/C2 强制触发同 sink_type / entry_type / 路径模式三轴兄弟全量复查, "点到面"放大。

**控制**: 深度 ≤ 2, 200 兄弟上限, 1 亿 Token 上上限, 触及写阻塞报告。

### §9.7 挑战者独立 + 对抗 loop 仲裁

**解决问题**: 同一 LLM 自我合理化, Phase 4-5 的分析师既是发现者又是确认者。

**核心原理**: 独立挑战者子代理不看推理上下文只看结论+源码, 质疑姿态而非复核。争议触发 3 轮对抗 loop, 第三方仲裁。

**为何适用**: 反 loop 工程"拆卷子与判卷师分开" 原则在安全审计的正确落地。

### §9.8 模式进化闭环 + 跨日累积 + Baseline 不替代

**解决问题**: 静态模式库跟不上新漏洞, 单 Session 学不到, 增量漏报累积。

**核心原理**: Phase 4b insights → 4 项晋升门槛 → `_learned/` 永久模式；跨日 hit_count 累积但晋升仍走门槛(防时间替代证据)；baseline 仅 diff 对比, 当日审计始终权威。

### §9.9 局部 loop + Token 硬停止

**解决问题**: 全 Pipeline loop 重跑成本爆炸且责任真空; 但个别子流程可循环至收敛。

**核心原理**: 仅 3 局部(loop ≤3 轮)——1b 鉴权遗漏反查 / 2b uncertain 扩读 / 6 对抗仲裁。50 亿 Token 全局硬上限, 触阻塞报告。

**与 loop 区别**: loop 适合"对错清晰可验证"的活(lint 修复), 安全审计的鉴权和支付逻辑不该让 loop 碰, 故主体不 loop。

### §9.10 持续审计调度 + 增量模式

**解决问题**: 300w 行不能每日全审, 但每天有新 PR 要审。

**核心原理**: cron/CI 定时触发 `mode=increment`, 仅审 git diff + 调用图邻居 ~5%, 总成本从 5 亿降到 1.5 亿 Token。Phase 9 跨日累积进化模式库。

**为何适用**: 把 "loop" 从 Session 内移至时间维度——每天比前一天多懂, 不是当日反复跑同一段。

### §9.11 文档可信度分级(Phase 1c)

**解决问题**: 产品文档人为写, 不一定可信, 可能与代码矛盾。

**核心原理**: 每条业务约束加 confidence 字段(high / medium / low / unverified)。文档约束不许单独判 VULN, 必配代码证据。

### §9.12 md+json+SARIF 三轨交付 + Excel 跟踪表 + 类型三列

**解决问题**: 不同消费者(人读/程序调用/CI 集成/运维跟踪)需要不同格式。

**核心原理**: 每产出并出 markdown + json + SARIF；跟踪表 csv(必产)+xlsx(可选含 3 sheet)；类型必含编号+中文名+子类型三列, 名称必从映射表查填。

---

## §10 借鉴与原创标注

| 机制/约束 | 起源 | 性质 |
|---------|------|------|
| Pipeline 编排+知识图谱 | GenCPT | 借鉴 |
| 反幻觉 1-10 全局 | GenCPT | 借鉴 |
| 五态标记 | GenCPT | 借鉴 |
| C1/C2/C3 分级 | GenCPT | 借鉴 |
| 模式库 8 段+条件触发表 | GenCPT | 借鉴 |
| 模式进化 4 门槛+自净 | GenCPT 增强(跨日 hit_count) | 借鉴+扩展 |
| QA 三层+覆盖矩阵 | GenCPT | 借鉴 |
| 分层子技能架构 | v0.6.0 RuoJi6 | 借鉴+修正(反单 skill 退化) |
| 六项验收门槛 | v0.6.1 RuoJi6 | 借鉴 |
| 攻击面通道枚举 | 落地材料 | 借鉴+扩到 13 |
| 任务划分规则 | 落地材料 Stage3 | 借鉴 |
| 假阳性验证原则 | 落地材料 Stage5 | 借鉴+合入 non-vuln-scenarios |
| Sibling-Scan 横向排查 | SourceCPT 原创(用户提设计) | 原创 |
| 13 入口+28 sink 双轨索引 | SourceCPT 原创 | 原创 |
| 局部 loop+硬停止 | loop engineering 部分采纳 | 借鉴改造 |
| 方法级图谱临时展开 | SourceCPT 原创 | 原创 |
| 持续审计跨日调度 | loop engineering 时间维度变体 | 原创改造 |
| 合规降为可选插件 | 用户质疑+反思 | 修正 |
| md+json+SARIF 三轨交付 | 用户提出 | 修正 |
| Excel 跟踪表类型三列 | 用户提出 | 修正 |
| POC 生成无门禁 | 用户提(白盒 POC 非武器) | 修正 |
| 文档约束可信度分级 | 用户提"文档不可信" | 修正 |
| 证据链五段 | v0.6.1 六项门槛+扩展 | 借鉴+扩展 |
| patch-patterns 修复建议 | 借鉴材料 | 借鉴 |
| 假阳性场景库 | 落地材料+v0.6.1 合并 | 借鉴+原创 |
| Token 硬停止预算 | loop engineering 借鉴 | 借鉴改造 |
| SARIF 输出 | 工业标准借鉴 | 借鉴 |

---

## §11 已知限制与盲区(诚实声明)

| 盲区 | 原因 | 缓解 |
|------|------|------|
| 业务逻辑漏洞 | 无静态 sink | Phase 1c+4b 推理+ Top3 盲区显式列 |
| 跨仓库调用链 | Phase 0 单仓图 | 报告标"跨仓未覆盖"Top3 |
| 资源竞争/时序 | 静态分析范畴外 | 显式声明静态范畴 |
| 全量 100% 漏报 | 软件安全客观上限 | 覆盖率可度量+ Top3 盲区(不承诺 100%) |
| 零误报 | LLM 推理编造 | Phase 6 挑战者+假阳性库(不承诺零) |
| 文档与代码不一致 | 文档不可信 | confidence 标注+不许文档单独判 VULN(#42) |
| 全量语义审成本 | 300w 行几亿 Token | 增量模式(默认)+首次 full(月 1 次) |
| 持续审计误漏 baseline 未发现 | 假设历史对 | 月度 full 重审, baseline 永不替代当前(#31) |

**框架承诺**: 已识别攻击面 100% 有结论, 行覆盖率 100% 可度量, 覆盖矩阵可视化, 未知攻击面盲区 Top3 显式列。

**框架不承诺**: 100% 漏洞找全(客观上限), 零误报(LLM 推理性质)。