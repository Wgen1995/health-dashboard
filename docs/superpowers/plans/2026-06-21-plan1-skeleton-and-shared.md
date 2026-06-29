# Container Pentest Suite V3 — 计划1: 骨架与共享规则

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 搭建容器渗透测试套件的目录骨架、共享规则文件、数据契约模板和Pipeline入口SKILL.md

**架构：** 所有SKILL.md是纯Markdown指令文件（给LLM读的），不是代码程序。目录结构严格遵循设计文档4节的树。共享规则文件定义全局标准（输出格式、严重度评级、漏洞分组、SSH命令模板、QA回溯）。Pipeline入口负责参数收集、环境验证、初始化工作目录、启动supervisory-agent。

**技术栈：** 纯Markdown，零代码。设计文档为唯一权威来源。

**设计文档：** `/root/docs/superpowers/specs/2026-06-20-GenCPT-suite-v3-design.md`（1766行）

**相关章节：** 1-6节（概述、架构、技能清单、目录结构、关键机制5.1-5.16、参数设计），11.1（Pipeline入口）

---

## 文件结构

```
GenCPT-suite/
├── SKILL.md                              # Pipeline 入口
├── skills/
│   └── shared/
│       ├── OUTPUT_STANDARD.md            # 输出格式标准
│       ├── SEVERITY_RATING.md            # 严重度评级
│       ├── VULNERABILITY_GROUPING.md     # 漏洞分组
│       ├── SSH_COMMANDS.md               # SSH命令模板+限速+重试
│       └── QA_OVERRIDE_TRACKING.md       # 质检回溯规则
├── compliance-rules/
│   └── _index.md                         # 总索引
├── attack-patterns/
│   └── _index.md                         # 索引+条件触发读取表（骨架）
├── hypothesis-libraries/
│   ├── compliance-hypotheses.md          # 骨架
│   ├── attack-hypotheses.md              # 骨架
│   └── cross-ref-queries.md             # 骨架
├── tools/
│   └── _index.md                         # 工具索引（骨架）
└── references/
    ├── quality_check_templates.md        # 质检校验清单
    ├── compliance.md                      # 安全红线
    ├── workspace-contract.md             # 工作目录规范
    ├── attack-surface-model.md           # 7大攻击面框架
    ├── promotion-criteria.md             # 晋升判定规则
    └── promotion-template.md             # 晋升模板SKILL.md结构
```

---

### 任务 1：创建目录骨架

**文件：**
- 创建：`GenCPT-suite/` 下所有目录

- [ ] **步骤 1：创建完整目录结构**

```bash
mkdir -p GenCPT-suite/skills/shared
mkdir -p GenCPT-suite/skills/recon
mkdir -p GenCPT-suite/skills/recon-source
mkdir -p GenCPT-suite/skills/k8s-compliance
mkdir -p GenCPT-suite/skills/docker-compliance
mkdir -p GenCPT-suite/skills/containerd-compliance
mkdir -p GenCPT-suite/skills/cross-ref
mkdir -p GenCPT-suite/skills/attack-pattern
mkdir -p GenCPT-suite/skills/attack-reasoning
mkdir -p GenCPT-suite/skills/chain-builder
mkdir -p GenCPT-suite/skills/chain-verify
mkdir -p GenCPT-suite/skills/poc-generator
mkdir -p GenCPT-suite/skills/report-compliance
mkdir -p GenCPT-suite/skills/report-attack
mkdir -p GenCPT-suite/skills/report-summary
mkdir -p GenCPT-suite/skills/evolve
mkdir -p GenCPT-suite/compliance-rules/kubernetes
mkdir -p GenCPT-suite/compliance-rules/docker
mkdir -p GenCPT-suite/compliance-rules/containerd
mkdir -p GenCPT-suite/attack-patterns/escape/_learned
mkdir -p GenCPT-suite/attack-patterns/auth/_learned
mkdir -p GenCPT-suite/attack-patterns/network/_learned
mkdir -p GenCPT-suite/attack-patterns/data/_learned
mkdir -p GenCPT-suite/attack-patterns/dos/_learned
mkdir -p GenCPT-suite/attack-patterns/supply/_learned
mkdir -p GenCPT-suite/attack-patterns/persist/_learned
mkdir -p GenCPT-suite/hypothesis-libraries
mkdir -p GenCPT-suite/tools/linux-amd64
mkdir -p GenCPT-suite/tools/linux-arm64
mkdir -p GenCPT-suite/references/report-templates
```

预期：所有目录创建成功，无报错。

- [ ] **步骤 2：验证目录结构**

```bash
find GenCPT-suite -type d | sort
```

预期：输出包含所有上述目录，7个attack-patterns攻击面子目录各有_learned子目录。

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git init && git add -A && git commit -m "feat: create directory skeleton for GenCPT-suite"
```

---

### 任务 2：共享规则 — OUTPUT_STANDARD.md

**文件：**
- 创建：`GenCPT-suite/skills/shared/OUTPUT_STANDARD.md`

- [ ] **步骤 1：创建输出格式标准**

读取设计文档5.7节（数据契约）中MUST输出规范，写OUTPUT_STANDARD.md。内容包含：
- 所有Phase的MUST输出文件列表（从5.7节Phase间MUST输入输出表提取）
- 文件命名规范（snake_case，证据目录按Phase分子目录）
- JSON文件格式要求（知识图谱节点和边的schema）
- Markdown文件格式要求（标题层级、表格格式、五态标记用法）
- WU摘要格式规范（≤500 tokens，结构化：work_unit_id + status + summary + critical_findings + files_written + context_used + issues）

每条规范附设计文档中的章节引用，便于后续SKILL.md实现时追溯。

设计文档行号参考：5.7数据契约（行455-574）

- [ ] **步骤 2：验证文件内容覆盖所有Phase的MUST输出**

```bash
grep -c "Phase" GenCPT-suite/skills/shared/OUTPUT_STANDARD.md
```

预期：≥9（覆盖Phase 1a到Phase 9）

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/shared/OUTPUT_STANDARD.md && git commit -m "feat: add OUTPUT_STANDARD.md with MUST output specs"
```

---

### 任务 3：共享规则 — SEVERITY_RATING.md

**文件：**
- 创建：`GenCPT-suite/skills/shared/SEVERITY_RATING.md`

- [ ] **步骤 1：创建严重度评级**

读取设计文档8节（漏洞验证程度）和5.16.4节（报告可信度分级），写SEVERITY_RATING.md。内容包含：
- 5级验证程度表（confirmed/condition_met/high_risk_clue/不可利用/已阻断）
- 3级可信度（C1/C2/C3）定义和证据要求
- 5级验证程度与C1/C2/C3的映射关系
- 报告标注符号（✅✅/✅/⚠️/➖/🛑）
- 漏洞来源标识（📚/🧠/🔄）
- 执行上下文层级（L0/L1/L2/L3）定义
- 全景报告中每种验证程度的位置（漏洞主体 vs 覆盖矩阵）

设计文档行号参考：8节（行1159-1186）、5.16节（行965-1120）

- [ ] **步骤 2：验证C1/C2/C3和5级映射完整**

```bash
grep -c "C1\|C2\|C3\|confirmed\|condition_met\|high_risk_clue" GenCPT-suite/skills/shared/SEVERITY_RATING.md
```

预期：≥6（覆盖所有级别）

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/shared/SEVERITY_RATING.md && git commit -m "feat: add SEVERITY_RATING.md with C1-C3 and 5-level verification"
```

---

### 任务 4：共享规则 — VULNERABILITY_GROUPING.md

**文件：**
- 创建：`GenCPT-suite/skills/shared/VULNERABILITY_GROUPING.md`

- [ ] **步骤 1：创建漏洞分组定义**

读取设计文档4节目录结构中攻击模式分组和设计文档7节（漏洞来源标识），写VULNERABILITY_GROUPING.md。内容包含：
- 7大攻击面定义（AS-1逃逸/AS-2认证授权/AS-3网络/AS-4数据泄露/AS-5拒绝服务/AS-6供应链/AS-7持久化）
- 每个攻击面的子类型和典型模式
- 漏洞分组与CIS Benchmark分组的映射关系
- ATK-CAND编号规则（设计文档14节）
- COMP-CAND编号规则（设计文档14节）

设计文档行号参考：4节目录结构（行209-259）、7节（行987-998）、14节（行1496-1507）

- [ ] **步骤 2：验证7大攻击面完整**

```bash
grep -c "AS-[0-9]" GenCPT-suite/skills/shared/VULNERABILITY_GROUPING.md
```

预期：≥7（覆盖AS-1到AS-7）

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/shared/VULNERABILITY_GROUPING.md && git commit -m "feat: add VULNERABILITY_GROUPING.md with 7 attack surfaces"
```

---

### 任务 5：共享规则 — SSH_COMMANDS.md

**文件：**
- 创建：`GenCPT-suite/skills/shared/SSH_COMMANDS.md`

- [ ] **步骤 1：创建SSH命令模板**

读取设计文档5.13节（SSH命令限速与重试）和5.16节（执行上下文L0-L3），写SSH_COMMANDS.md。内容包含：
- SSH限速规则（最大并行3、批次间隔2秒、单次超时30秒、降级串行5秒间隔）
- 重试策略表（读3次/探2次/攻1次）
- 失败标记规则（`[!]`环境干扰、ATK-CAND降级）
- L0命令模板（宿主机观察：kubectl get/describe、docker inspect/ps/info、crictl pods/ps/images）
- L1命令模板（容器内观察：kubectl exec -- cat/ls/id/find）
- L2命令模板（容器内攻击验证：kubectl exec -- docker run/nsenter等）
- 各级命令的安全注意事项
- 文件写入路径规范（evidence/recon/raw/、按Phase分子目录）

设计文档行号参考：5.13（行743-762）、5.16（行965-1030）

- [ ] **步骤 2：验证限速和重试策略完整**

```bash
grep -c "重试\|限速\|并行\|超时\|降级" GenCPT-suite/skills/shared/SSH_COMMANDS.md
```

预期：≥4（覆盖限速、重试、超时、降级）

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/shared/SSH_COMMANDS.md && git commit -m "feat: add SSH_COMMANDS.md with rate limiting, retry, and L0-L2 templates"
```

---

### 任务 6：共享规则 — QA_OVERRIDE_TRACKING.md

**文件：**
- 创建：`GenCPT-suite/skills/shared/QA_OVERRIDE_TRACKING.md`

- [ ] **步骤 1：创建质检回溯规则**

读取设计文档5.5节（QA三层机制），写QA_OVERRIDE_TRACKING.md。内容包含：
- 第一层：结构性完整性校验5项检查清单（MUST输出存在、知识图谱边节点对应、ATK-CAND编号连续、五态无`[ ]`、合规覆盖率）
- 第二层：语义抽检流程（5条合规+3条ATK-CAND+2条`[-]`）
- 第三层：回溯验证规则（QA-OVERRIDE记录格式、Phase 8c汇总、置信度评分计算）
- QA-OVERRIDE文件命名规范：`evidence/qa/qa_override_{phase}.md`

设计文档行号参考：5.5（行393-426）

- [ ] **步骤 2：验证三层QA完整**

```bash
grep -c "第一层\|第二层\|第三层\|结构\|语义\|回溯" GenCPT-suite/skills/shared/QA_OVERRIDE_TRACKING.md
```

预期：≥3（覆盖三层）

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/shared/QA_OVERRIDE_TRACKING.md && git commit -m "feat: add QA_OVERRIDE_TRACKING.md with 3-layer QA rules"
```

---

### 任务 7：参考资料 — 6个核心文件

**文件：**
- 创建：`GenCPT-suite/references/quality_check_templates.md`
- 创建：`GenCPT-suite/references/compliance.md`
- 创建：`GenCPT-suite/references/workspace-contract.md`
- 创建：`GenCPT-suite/references/attack-surface-model.md`
- 创建：`GenCPT-suite/references/promotion-criteria.md`
- 创建：`GenCPT-suite/references/promotion-template.md`

- [ ] **步骤 1：创建quality_check_templates.md**

内容包含每个Phase的结构性校验清单模板（从5.5节第一层和17节Phase Checkpoint门控表提取）

设计文档行号参考：5.5（行393-426）、17节（行1555-1570）

- [ ] **步骤 2：创建compliance.md**

内容包含安全红线（从5.8 baseline安全红线提取）和合规规则编写规范（每条规则的格式：编号、描述、检查命令、判定标准、修复建议）

- [ ] **步骤 3：创建workspace-contract.md**

内容包含工作目录结构规范（从5.7数据契约的工作目录结构提取）、文件命名规范、Phase间数据传递规则

- [ ] **步骤 4：创建attack-surface-model.md**

内容包含7大攻击面框架详细定义（从VULNERABILITY_GROUPING.md扩展），每个攻击面包含：攻击面ID、名称、描述、典型攻击路径、与CIS Benchmark的关联

- [ ] **步骤 5：创建promotion-criteria.md**

内容包含攻击模式晋升门槛4项（差分证明充分+探测可复现+无法匹配现有模式+跨会话命中≥2次）、晋升流程、来源标记（manual/curated/learned）、降级归档规则

设计文档行号参考：5.4（行347-396）

- [ ] **步骤 6：创建promotion-template.md**

内容包含晋升新模式时使用的SKILL.md模板，遵循5.11的8段格式规范，含frontmatter模板（source/learned、confidence/medium、platforms/required_tools/execution_contexts/max_verification_level/destructive）

- [ ] **步骤 7：验证6个文件全部创建**

```bash
ls -la GenCPT-suite/references/*.md
```

预期：6个文件全部存在

- [ ] **步骤 8：Commit**

```bash
cd GenCPT-suite && git add references/ && git commit -m "feat: add 6 reference files (quality check, compliance, workspace, attack surface, promotion criteria, promotion template)"
```

---

### 任务 8：假设库骨架和工具索引骨架

**文件：**
- 创建：`GenCPT-suite/hypothesis-libraries/compliance-hypotheses.md`
- 创建：`GenCPT-suite/hypothesis-libraries/attack-hypotheses.md`
- 创建：`GenCPT-suite/hypothesis-libraries/cross-ref-queries.md`
- 创建：`GenCPT-suite/tools/_index.md`
- 创建：`GenCPT-suite/attack-patterns/_index.md`（骨架，条件触发读取表将在计划3填充）

- [ ] **步骤 1：创建三库骨架文件**

compliance-hypotheses.md：骨架包含标题、格式说明（假设卡片结构：假设ID、违规族、映射的攻击模式、严重等级、前置条件示例），3-5个示例卡片

attack-hypotheses.md：骨架包含标题、格式说明（假设ID、攻击面引用、前置条件、探测命令、攻击验证、差分证明、绕过策略、证伪条件、合规映射），3-5个示例卡片

cross-ref-queries.md：骨架包含标题、三条核心查询模板（XREF-001合规→攻击、XREF-002叠加放大、XREF-003攻击→侦察比对），2-3个示例

设计文档行号参考：5.3（行318-351）

- [ ] **步骤 2：创建工具索引骨架**

tools/_index.md：包含标题、通用工具表（jq/yq等，列名/用途/大小/攻击面/平台/上传条件）、攻击验证工具表（cdk/amicontained等），行数约30行

设计文档行号参考：5.15.2（行813-848）

- [ ] **步骤 3：创建攻击模式索引骨架**

attack-patterns/_index.md：包含标题、7大攻击面列表、条件触发读取表占位（将在计划3填充具体内容）、平台过滤规则说明

- [ ] **步骤 4：验证5个文件创建**

```bash
ls -la GenCPT-suite/hypothesis-libraries/ GenCPT-suite/tools/_index.md GenCPT-suite/attack-patterns/_index.md
```

预期：5个文件全部存在

- [ ] **步骤 5：Commit**

```bash
cd GenCPT-suite && git add hypothesis-libraries/ tools/_index.md attack-patterns/_index.md && git commit -m "feat: add hypothesis libraries, tool index, and attack pattern index skeletons"
```

---

### 任务 9：Pipeline入口 SKILL.md

**文件：**
- 创建：`GenCPT-suite/SKILL.md`

- [ ] **步骤 1：创建Pipeline入口SKILL.md**

读取设计文档11.1节（GenCPT Pipeline入口），写SKILL.md。内容包含：

YAML frontmatter：
```yaml
---
name: GenCPT
description: >
  容器与 Kubernetes 渗透测试技能套件（opencode SKILL），不是代码程序。通过 SSH 远程对 K8s/Docker/containerd 环境做合规检测、攻击验证、链式攻击和报告交付。支持攻击模式库自我进化。所有逻辑由 LLM 语义处理，产出物是 Markdown 报告和 JSON 知识图谱。SKILL 全部为中文指令。
  使用场景：对 K8s/Docker/containerd 环境做授权渗透测试。
  不使用场景：未授权攻击、生产破坏性操作、批量扫描非自有目标。
---
```

工作流5步：
1. 参数收集（question工具交互收集server, mode, scope, approval, source-path, baseline）
2. 环境验证（ssh_execute测试连通性+OS/arch/工具检查+记录环境指纹）
3. 初始化工作目录（bash本地创建目录结构，写session_config.json和progress.json）
4. 启动supervisory-agent（1个Task，传入所有参数）
5. 展示摘要（读取报告，输出关键数字和文件路径列表）

禁止事项：不直接调度Phase级agent、不执行检测命令、不读取原始数据、不处理Phase间传递、不做断点续传

参数格式（从设计文档6节）

设计文档行号参考：11.1（行1223-1247）、6节（行1121-1145）

- [ ] **步骤 2：验证SKILL.md包含5步入口流程和禁止事项**

```bash
grep -c "参数收集\|环境验证\|初始化\|启动\|展示摘要\|禁止事项" GenCPT-suite/SKILL.md
```

预期：≥6（覆盖5步+禁止事项）

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add SKILL.md && git commit -m "feat: add Pipeline entry SKILL.md"
```

---

### 任务 10：合规规则总索引

**文件：**
- 创建：`GenCPT-suite/compliance-rules/_index.md`
- 创建：`GenCPT-suite/compliance-rules/kubernetes/_index.md`
- 创建：`GenCPT-suite/compliance-rules/docker/_index.md`
- 创建：`GenCPT-suite/compliance-rules/containerd/_index.md`

- [ ] **步骤 1：创建4个索引文件**

compliance-rules/_index.md：总索引，列出三个平台（Kubernetes 135条/29分组、Docker 64条/7分组、Containerd 28条/5分组），指向各平台子索引

kubernetes/_index.md：K8s索引，列出29个分组文件名和规则数（从设计文档4节目录结构提取G_1_1到G_8_4），总135条

docker/_index.md：Docker索引，列出7个分组文件名和规则数（G_1到G_7），总64条

containerd/_index.md：Containerd索引，列出5个分组文件名和规则数（G_1到G_5），总28条

设计文档行号参考：4节目录结构（行159-207）

- [ ] **步骤 2：验证4个索引文件完整**

```bash
for f in compliance-rules/_index.md compliance-rules/kubernetes/_index.md compliance-rules/docker/_index.md compliance-rules/containerd/_index.md; do echo "=== $f ===" && head -5 GenCPT-suite/$f; done
```

预期：每个文件包含平台名称和规则条数统计

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add compliance-rules/ && git commit -m "feat: add compliance-rules index files for K8s/Docker/Containerd"
```

---

### 任务 11：端到端验证

- [ ] **步骤 1：验证完整目录结构**

```bash
find GenCPT-suite -type f | sort
```

预期：包含SKILL.md、5个shared文件、6个references文件、3个hypothesis-libraries文件、tools/_index.md、attack-patterns/_index.md、4个compliance-rules索引文件

- [ ] **步骤 2：验证SKILL.md frontmatter**

```bash
head -10 GenCPT-suite/SKILL.md
```

预期：包含name、description、使用场景、不使用场景

- [ ] **步骤 3：验证shared目录5个文件**

```bash
ls GenCPT-suite/skills/shared/
```

预期：OUTPUT_STANDARD.md、SEVERITY_RATING.md、VULNERABILITY_GROUPING.md、SSH_COMMANDS.md、QA_OVERRIDE_TRACKING.md

- [ ] **步骤 4：验证references目录6个文件**

```bash
ls GenCPT-suite/references/
```

预期：quality_check_templates.md、compliance.md、workspace-contract.md、attack-surface-model.md、promotion-criteria.md、promotion-template.md

- [ ] **步骤 5：最终Commit（如有未提交的更改）**

```bash
cd GenCPT-suite && git add -A && git status
```

预期：无未提交更改，或仅有.gitignore等辅助文件

## 补充实现（V3.1增强）

以下增强功能在 V3 基础上追加，不影响原有功能：

- [x] **建议3：suite_version 版本兼容** — session_config.json 增加 `suite_version` 字段，baseline 版本不一致时降级为趋势对比。实现位置：`GenCPT-suite/SKILL.md`（session_config.json 结构）。commit: 未提交（工作区变更）
- [x] **建议5：差分证明 L0 强制（SEVERITY_RATING）** — 按 AS-1/AS-5/AS-7 攻击类型强制三段式 L0 观测，不满足 C1 降级 C2；hostpath-mount 例外。实现位置：`GenCPT-suite/skills/shared/SEVERITY_RATING.md`。commit: 未提交（工作区变更）