# Container Pentest Suite V3 — 计划4: 报告 + 进化 + 剩余SKILL

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 创建3个报告SKILL.md、evolve SKILL.md，完成所有16个SKILL和report-templates参考文件

**架构：** 报告SKILL分3个并行Phase（8a合规+8b攻击+8c综合+全景），每个读独立数据源。evolve独立运行，处理LLM推理晋升。report-templates提供报告格式模板。

**设计文档：** `/root/docs/superpowers/specs/2026-06-20-GenCPT-suite-v3-design.md`
**依赖：** 计划1-3（骨架+合规+攻击模式+攻击验证）

---

## 文件结构

```
GenCPT-suite/
├── skills/
│   ├── report-compliance/SKILL.md
│   ├── report-attack/SKILL.md
│   ├── report-summary/SKILL.md
│   └── evolve/SKILL.md
└── references/
    └── report-templates/
        ├── compliance_report.md
        ├── attack_report.md
        └── pentest_report.md
```

---

### 任务 1：report-templates参考文件

**文件：**
- 创建：`GenCPT-suite/references/report-templates/compliance_report.md`
- 创建：`GenCPT-suite/references/report-templates/attack_report.md`
- 创建：`GenCPT-suite/references/report-templates/pentest_report.md`

- [ ] **步骤 1：创建compliance_report.md模板**

包含完整Markdown模板结构：
- 报告头（检测时间、目标环境、mode/scope/审批模式）
- 合规概览表（总规则数/pass/fail/warn/na统计）
- 按CIS分组的详细结果（每组一个小节，每条规则含编号、描述、判定、依据摘要、修复建议）
- 合规检查点报告（Phase 2后独立输出）
- 高风险项汇总（fail中严重性High/Critical的条目）

读取设计文档5.2（合规检查点报告）和5.10（三重校验）

- [ ] **步骤 2：创建attack_report.md模板**

包含：
- 报告头（同上）
- ATK-CAND汇总表（编号/来源/攻击面/可信度/状态）
- 每个ATK-CAND详细结果（前置条件+L层探测命令+差分证明+审批状态）
- 攻击链分析（CHAIN-xxx）
- POC包引用

读取设计文档8节（漏洞验证程度）、5.16.4-5.16.5（可信度+C1/C2/C3+POC格式）

- [ ] **步骤 3：创建pentest_report.md模板**

包含10大章节的Markdown模板：
1. 执行详情（时间/模式/环境/Phase/WU统计）
2. 环境覆盖全景（侦察/源码/合规覆盖率）
3. 合规分组热力图
4. 攻击面覆盖矩阵（7攻击面×模式库/LLM推理/总覆盖）
5. 已知模式库覆盖全景
6. LLM推理覆盖全景
7. 攻击验证结果分布（confirmed/条件成立/高风险线索/不可利用/已阻断）
8. 漏洞来源标识（📚/🧠/🔄）
9. 未覆盖事项及原因
10. 产品安全质量评估

读取设计文档5.3（全景报告10大章节）、5.4（未覆盖事项格式）

- [ ] **步骤 4：验证3个模板文件**

```bash
ls -la GenCPT-suite/references/report-templates/
```

预期：3个文件

- [ ] **步骤 5：Commit**

```bash
cd GenCPT-suite && git add references/report-templates/ && git commit -m "feat: add 3 report templates (compliance, attack, pentest)"
```

---

### 任务 2：report-compliance SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/report-compliance/SKILL.md`

- [ ] **步骤 1：创建report-compliance SKILL.md**

读取设计文档11.13节，写完整SKILL.md。

YAML frontmatter：
```yaml
---
name: report-compliance
description: >
  合规检测报告生成。读取Phase 2合规数据，生成MD+JSON格式合规报告和合规检查点报告。
  使用场景：Phase 2完成后生成报告。
  不使用场景：攻击验证阶段。
---
```

核心工作流：
1. 读取evidence/compliance/目录下所有数据
2. 按CIS分组汇总（每组pass/fail/warn/na统计）
3. 生成compliance_checkpoint_report.md（每条规则有判定+依据+SSH输出摘要）
4. 生成compliance_report.md（在检查点基础上增加攻击关联+风险等级+修复优先级）
5. 生成compliance_report.json（结构化数据）

三重校验第三重：报告中每条规则有判定、fail/warn有依据、数字合计

MUST输出：reports/compliance_checkpoint_report.md、reports/compliance_checkpoint_report.json、reports/compliance_report.md、reports/compliance_report.json

设计文档行号参考：11.13（行1333-1350）、5.2（行304-340）、5.10（行593-632）

- [ ] **步骤 2：验证**

```bash
grep -c "三重校验\|检查点\|MUST 输出\|CIS\|checkpoint" GenCPT-suite/skills/report-compliance/SKILL.md
```

预期：≥4

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/report-compliance/ && git commit -m "feat: add report-compliance SKILL.md (Phase 8a)"
```

---

### 任务 3：report-attack SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/report-attack/SKILL.md`

- [ ] **步骤 1：创建report-attack SKILL.md**

读取设计文档11.14节，写完整SKILL.md。

核心工作流：
1. 读取evidence/attack/目录下所有数据（pattern-hits.md、reasoning-hits.md、unmatched_signals.md、insights.md）
2. 读取evidence/chains/目录（chain_builder.md、chain_verification.md）
3. 生成attack_report.md，包含：
   - ATK-CAND汇总表（编号/来源📚🧠🔄/攻击面/可信度C1-C3/状态）
   - 每个ATK-CAND详细结果（前置条件+L层探测+差分证明+审批状态+执行上下文标注）
   - 攻击链分析（CHAIN-xxx）
   - POC包引用
4. 生成attack_report.json

**关键：每条ATK-CAND必须标注可信度（C1/C2/C3）和执行上下文层级（L0/L1/L2/L3）。不可利用项也要包含（标注"已证伪+证伪依据"）。已阻断项标注阻断机制名称。**

MUST输出：reports/attack_report.md、reports/attack_report.json、reports/poc_package/

设计文档行号参考：11.14（行1351-1370）

- [ ] **步骤 2：验证**

```bash
grep -c "C1\|C2\|C3\|L0\|L1\|L2\|L3\|ATK-CAND\|不可利用\|已阻断\|POC" GenCPT-suite/skills/report-attack/SKILL.md
```

预期：≥5

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/report-attack/ && git commit -m "feat: add report-attack SKILL.md (Phase 8b)"
```

---

### 任务 4：report-summary SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/report-summary/SKILL.md`

- [ ] **步骤 1：创建report-summary SKILL.md**

读取设计文档11.15节，写完整SKILL.md。

核心工作流：
1. 读取8a+8b报告+知识图谱完整数据+insights+session_history
2. 生成pentest_report.md（综合报告）
3. 生成coverage_report.md（全景报告，10大章节）
4. QA语义抽检（5条合规+3条ATK-CAND+2条[-]→ssh_execute重新验证）
5. 计算QA置信度评分
6. 更新episodic_memory/session_history.md和_index.md hit_count

全景报告10大章节严格遵循设计文档5.3节：
1. 执行详情
2. 环境覆盖全景
3. 合规分组热力图
4. 攻击面覆盖矩阵
5. 已知模式库覆盖全景
6. LLM推理覆盖全景
7. 攻击验证结果分布（含C1/C2/C3/不可利用/已阻断分类统计）
8. 漏洞来源标识（📚/🧠/🔄）
9. 未覆盖事项及原因（模式库缺失+LLM未覆盖+环境限制+审批限制）
10. 产品安全质量评估

**关键：不可利用（已证伪）项必须出现在全景报告第7节的覆盖矩阵中，标注"已检查不可利用+证伪依据"。**

MUST输出：reports/pentest_report.md、reports/pentest_report.json、reports/coverage_report.md、evidence/qa/qa_summary_report.md

设计文档行号参考：11.15（行1371-1406）、5.3（行304-340）、5.5（行393-426）

- [ ] **步骤 2：验证**

```bash
grep -c "10\|覆盖全景\|QA\|语义抽检\|不可利用.*覆盖\|C1\|C2\|C3" GenCPT-suite/skills/report-summary/SKILL.md
```

预期：≥6

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/report-summary/ && git commit -m "feat: add report-summary SKILL.md (Phase 8c)"
```

---

### 任务 5：evolve SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/evolve/SKILL.md`

- [ ] **步骤 1：创建evolve SKILL.md**

读取设计文档11.16节，写完整SKILL.md。

YAML frontmatter：
```yaml
---
name: GenCPT-evolve
description: >
  攻击模式进化。分析本次检测的LLM推理发现，评估是否晋升为新攻击模式，执行自净。
  可在Pipeline末尾用 --evolve 参数触发，也可独立运行。
  不使用场景：检测中、不需要进化模式库时。
---
```

核心工作流5步：
1. 收集洞察：Read insights.md + session_history.md + _index.md
2. 逐条评估：与现有模式比对（是变体→更新hit_count；是全新→晋升门槛检查4项：差分证明充分+探测可复现+无法匹配现有模式+跨会话命中≥2次）
3. 用户审批：question工具逐条确认（添加/拒绝/暂存）
4. 通过→Read promotion-template.md→生成SKILL.md（8段结构）→写入_learned/→更新_index.md和attack-hypotheses.md
5. 一致性维护：LLM语义检查三库一致性

独立运行参数：`--session-dir /path/to/session`

**关键：晋升门槛4项全满足才标记可晋升。降级规则：learned命中≥5次跨≥2环境→high；连续5次未命中→medium；连续10次→stale；连续15次→问用户。**

设计文档行号参考：11.16（行1407-1435）、5.4（行347-396）

- [ ] **步骤 2：验证**

```bash
grep -c "晋升门槛\|4项\|hit_count\|stale\|question\|promotion-template\|_learned" GenCPT-suite/skills/evolve/SKILL.md
```

预期：≥5

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/evolve/ && git commit -m "feat: add evolve SKILL.md (Phase 9)"
```

---

### 任务 6：tools/_index.md 完整填充

**文件：**
- 修改：`GenCPT-suite/tools/_index.md`（计划8创建了骨架，现在填充完整内容）

- [ ] **步骤 1：填充完整工具索引**

读取设计文档5.15.2（工具索引格式），写完整_index.md。内容包含：
- 通用工具表（jq/yq/soct等，5-7个工具）
- 攻击验证工具表（cdk/amicontained/checksec/capsh等，6-8个工具）
- 每个工具列出：名称、用途、大小、攻击面、平台、上传条件

设计文档行号参考：5.15.2（行813-848）

- [ ] **步骤 2：验证工具数量**

```bash
grep -c "^|.*|" GenCPT-suite/tools/_index.md
```

预期：≥12（通用5-7 + 攻击验证6-8）

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add tools/_index.md && git commit -m "feat: fill complete tool index with upload conditions"
```

---

### 任务 7：最终端到端验证

- [ ] **步骤 1：验证16个SKILL.md全部存在**

```bash
for skill in recon recon-source k8s-compliance docker-compliance containerd-compliance cross-ref attack-pattern attack-reasoning chain-builder chain-verify poc-generator report-compliance report-attack report-summary evolve; do
  [ -f "GenCPT-suite/skills/$skill/SKILL.md" ] && echo "$skill: OK" || echo "$skill: MISSING"
done
```

预期：14个全部OK（加上Pipeline入口SKILL.md=15个，加上shared目录=16个子技能目录）

- [ ] **步骤 2：验证合规规则总数=227**

```bash
echo "K8s:"; grep -c "^\*\*检查命令" GenCPT-suite/compliance-rules/kubernetes/G_*.md 2>/dev/null | awk -F: '{s+=$NF}END{print s}'
echo "Docker:"; grep -c "^\*\*检查命令" GenCPT-suite/compliance-rules/docker/G_*.md 2>/dev/null | awk -F: '{s+=$NF}END{print s}'
echo "Containerd:"; grep -c "^\*\*检查命令" GenCPT-suite/compliance-rules/containerd/G_*.md 2>/dev/null | awk -F: '{s+=$NF}END{print s}'
```

预期：135+64+28=227

- [ ] **步骤 3：验证攻击模式总数≥20**

```bash
find GenCPT-suite/attack-patterns -name "SKILL.md" -not -path "*/_learned/*" | wc -l
```

预期：≥20（实际应为25，含ntfs-alpn）

- [ ] **步骤 4：验证假设库3个文件内容完整**

```bash
wc -l GenCPT-suite/hypothesis-libraries/*.md
```

预期：每个文件≥30行

- [ ] **步骤 5：验证所有文件总数**

```bash
find GenCPT-suite -type f | wc -l
```

预期：≥90（16个SKILL.md + 5个shared + 44个合规规则 + 25个攻击模式 + 3个假设库 + 工具索引 + 6个参考文件 + 3个报告模板 + 索引文件 + Pipeline入口）

- [ ] **步骤 6：最终Commit**

```bash
cd GenCPT-suite && git add -A && git status
```

预期：无未提交更改

- [ ] **步骤 7：验证设计文档关键决策覆盖**

对照设计文档，确认以下每个决策在文件中有实现：
- 方法C（工具库）→ tools/_index.md
- 攻击者视角分层（L0-L3）→ 攻击模式SKILL.md中的[L0]/[L1]/[L2]/[L3]标注
- 可信度分级（C1-C3）→ SEVERITY_RATING.md + report-attack SKILL.md
- 五态标记 → attack-pattern SKILL.md的penta-state规则
- 三重校验 → QA_OVERRIDE_TRACKING.md + compliance SKILL.md
- 合规按CIS分组 → compliance-rules/目录结构
- 执行上下文+POC格式 → poc-generator SKILL.md + 攻击模式SKILL.md

```bash
echo "方法C:"; [ -f "GenCPT-suite/tools/_index.md" ] && echo "OK" || echo "MISSING"
echo "攻击者视角:"; grep -c "\[L0\]\|\[L1\]\|\[L2\]\|\[L3\]" GenCPT-suite/attack-patterns/escape/socket-escape/SKILL.md
echo "可信度:"; grep -c "C1\|C2\|C3" GenCPT-suite/skills/shared/SEVERITY_RATING.md
echo "五态:"; grep -c "\[x\]\|\[?\]\|\[-\]\|\[!\]\|\[ \]" GenCPT-suite/skills/attack-pattern/SKILL.md
echo "三重校验:"; grep -c "第一重\|第二重\|第三重" GenCPT-suite/skills/shared/QA_OVERRIDE_TRACKING.md
```

预期：每项都有实现

## 补充实现（V3.1增强）

以下增强功能在 V3 基础上追加，不影响原有功能：

- [x] **建议3：baseline 版本检查** — report-summary 读取 baseline 报告时检查 suite_version，版本不一致降级为趋势对比并标注警告。实现位置：`GenCPT-suite/skills/report-summary/SKILL.md`。commit: 未提交（工作区变更）
- [x] **建议4：审批熔断机制** — 10 分钟内 ≥5 次 L3/L4 自动通过时暂停，强制 manual approval；session_config.json 维护 auto_high_risk_exec_count 计数器。实现位置：`GenCPT-suite/skills/chain-verify/SKILL.md` + `GenCPT-suite/SKILL.md`。commit: 未提交（工作区变更）