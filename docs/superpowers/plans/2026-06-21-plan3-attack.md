# Container Pentest Suite V3 — 计划3: 攻击模式库 + 攻击验证Phase

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 创建攻击模式库（7个攻击面25+模式SKILL.md）、条件触发索引、4个攻击验证Phase子技能（attack-pattern/attack-reasoning/chain-builder/chain-verify）、cross-ref和poc-generator

**架构：** 攻击模式SKILL.md严格遵循8段格式+frontmatter（含execution_contexts/max_verification_level/destructive/required_tools/platforms）。条件触发索引在_index.md中。攻击验证通过L0-L3执行上下文模拟攻击者视角。

**设计文档：** `/root/docs/superpowers/specs/2026-06-20-GenCPT-suite-v3-design.md`
**依赖：** 计划1（骨架+共享规则）、计划2（合规规则）

---

## 文件结构

```
GenCPT-suite/
├── attack-patterns/
│   ├── _index.md                   # 完整条件触发读取表
│   ├── escape/                      # AS-1 逃逸
│   │   ├── socket-escape/SKILL.md
│   │   ├── cgroup-escape/SKILL.md
│   │   ├── procfs-escape/SKILL.md
│   │   ├── runc-escape/SKILL.md
│   │   ├── hostpath-mount/SKILL.md
│   │   ├── capability-privesc/SKILL.md
│   │   └── containerd-shim-escape/SKILL.md
│   ├── auth/                        # AS-2 认证授权
│   │   ├── k8s-sa-exploit/SKILL.md
│   │   ├── k8s-rbac-abuse/SKILL.md
│   │   ├── k8s-anonymous-access/SKILL.md
│   │   └── docker-api-auth/SKILL.md
│   ├── network/                     # AS-3 网络
│   │   ├── lateral-move/SKILL.md
│   │   ├── ntfs-alpn/SKILL.md
│   │   ├── cloud-metadata/SKILL.md
│   │   └── dns-exfil/SKILL.md
│   ├── data/                        # AS-4 数据泄露
│   │   ├── secret-exfil/SKILL.md
│   │   ├── env-credential-leak/SKILL.md
│   │   └── image-layer-secret/SKILL.md
│   ├── dos/                         # AS-5 拒绝服务
│   │   ├── resource-abuse/SKILL.md
│   │   └── fork-bomb/SKILL.md
│   ├── supply/                      # AS-6 供应链
│   │   ├── image-tag-mutation/SKILL.md
│   │   └── registry-poison/SKILL.md
│   └── persist/                    # AS-7 持久化
│       ├── webhook-backdoor/SKILL.md
│       ├── cronjob-persist/SKILL.md
│       └── docker-volume-persist/SKILL.md
├── skills/
│   ├── cross-ref/SKILL.md
│   ├── attack-pattern/SKILL.md
│   ├── attack-reasoning/SKILL.md
│   ├── chain-builder/SKILL.md
│   ├── chain-verify/SKILL.md
│   └── poc-generator/SKILL.md
```

---

### 任务 1：攻击模式索引 _index.md

**文件：**
- 修改：`GenCPT-suite/attack-patterns/_index.md`（计划1创建了骨架，现在填充完整条件触发读取表）

- [ ] **步骤 1：编写完整条件触发读取表**

读取设计文档18节（条件触发读取表），写完整的_index.md。内容包含：
- 7大攻击面列表及下属模式
- 条件触发读取表：每行列出"合规发现/侦察发现→需读取的模式"，例如：
  - K8s-5.2.1(privileged=true) → escape/socket-escape, escape/capability-privesc
  - K8s-5.2.3(docker.sock mounted) → escape/socket-escape
  - secret环境变量明文 → data/env-credential-leak
- 平台过滤规则：scope=k8s只加载platforms含k8s的模式
- 每个模式的frontmatter概要（name/platforms/mapped_attack_surfaces/mapped_compliance_families）

设计文档行号参考：18节（行1572-1596）、4节目录结构（行209-259）

- [ ] **步骤 2：验证索引包含全部模式和触发条件**

```bash
grep -c "^- " GenCPT-suite/attack-patterns/_index.md
```

预期：≥25（覆盖所有攻击模式，含ntfs-alpn）

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add attack-patterns/_index.md && git commit -m "feat: add complete attack-patterns index with trigger conditions"
```

---

### 任务 2：AS-1 逃逸攻击面（7个模式）

**文件：**
- 创建7个SKILL.md文件

- [ ] **步骤 1：创建socket-escape/SKILL.md**

严格遵循8段格式+frontmatter。读取设计文档5.11（8段格式规范）和5.16（执行上下文L0-L3 + POC格式）。

frontmatter示例：
```yaml
---
source: manual
confidence: high
platforms: [k8s, docker]
required_tools: []
execution_contexts: [L0, L1, L2]
max_verification_level: L2
destructive: false
mapped_attack_surfaces: [AS-1.1]
mapped_compliance_families: [特权容器, 危险挂载]
---
```

8段内容（每步标注L0/L1/L2/L3）：
1. 前置条件：容器内可见/var/run/docker.sock且有读写权限
2. 探测命令：[L0]kubectl get pod检查privileged, [L1]kubectl exec检查docker.sock
3. 攻击验证：[L2]kubectl exec -- docker run -v /:/host验证逃逸
4. 差分证明：[L0]攻击前后对比 + [L2]容器内读取宿主机文件
5. 绕过策略：AppArmor/Seccomp阻断检测 [L1]kubectl exec检查/proc/self/attr/current
6. 证伪条件：[L1]docker.sock不存在或无读写权限
7. 审批级别：L4（逃逸验证）+ L3（创建临时容器）
8. MITRE ATT&CK：T1611 - Escape to Host

- [ ] **步骤 2：创建剩余6个逃逸模式（cgroup-escape/procfs-escape/runc-escape/hostpath-mount/capability-privesc/containerd-shim-escape）**

每个模式遵循相同8段格式+L层标注。关键差异：
- cgroup-escape：[L1]检查/proc/1/cgroup，[L2]通过cgroup release_agent逃逸，destructive:true时max_verification_level:L3
- procfs-escape：[L1]检查/proc挂载，[L2]通过/proc/sys写操作逃逸
- runc-escape：platforms:[docker,containerd]，[L0]检查runc版本
- hostpath-mount：platforms:[k8s]，[L0]kubectl get pod检查hostPath
- capability-privesc：[L1]检查/proc/self/status的Cap，platforms:[k8s,docker,containerd]
- containerd-shim-escape：platforms:[containerd]，[L1]检查containerd-shim socket

- [ ] **步骤 3：验证7个逃逸模式都包含8段和L层标注**

```bash
for f in GenCPT-suite/attack-patterns/escape/*/SKILL.md; do
  echo "=== $(basename $(dirname $f)) ===" 
  grep -c "^## [1-8]\." "$f"
  grep -c "\[L0\]\|\[L1\]\|\[L2\]\|\[L3\]" "$f"
done
```

预期：每个文件8段≥8，L层标注≥3

- [ ] **步骤 4：Commit**

```bash
cd GenCPT-suite && git add attack-patterns/escape/ && git commit -m "feat: add AS-1 escape attack patterns (7 modes)"
```

---

### 任务 3：AS-2到AS-7攻击面（13个模式）

**文件：**
- 创建13个SKILL.md文件

- [ ] **步骤 1：创建AS-2认证授权（4个模式）**

k8s-sa-exploit：[L0]kubectl get sa，[L1]检查token可读性，[L2]用sa token访问API
k8s-rbac-abuse：[L0]kubectl auth can-i，[L1]检查权限提升路径
k8s-anonymous-access：[L0]kubectl访问匿名端点
docker-api-auth：platforms:[docker]，[L0]docker port检查，[L1]curl访问API

- [ ] **步骤 2：创建AS-3网络（4个模式）**

lateral-move：[L1]kubectl exec curl/nc探测网络，[L2]验证横向访问
ntfs-alpn：[L1]kubectl exec检查ALPN协议协商，[L2]利用NTFS ALPN漏洞
cloud-metadata：[L1]kubectl exec curl访问169.254.169.254
dns-exfil：[L1]kubectl exec验证DNS解析

- [ ] **步骤 3：创建AS-4数据泄露（3个模式）**

secret-exfil：[L0]kubectl get secrets，[L1]kubectl exec检查env挂载
env-credential-leak：platforms:[k8s,docker,containerd]，[L1]kubectl exec检查环境变量
image-layer-secret：platforms:[docker,containerd]，[L0]docker history检查

- [ ] **步骤 4：创建AS-5拒绝服务（2个模式）**

resource-abuse：destructive:true，max_verification_level:L3，[L0]检查资源限制
fork-bomb：destructive:true，max_verification_level:L3，[L0]检查特权+资源限制，POC标⚠️

- [ ] **步骤 5：创建AS-6供应链（2个模式）**

image-tag-mutation：[L0]检查镜像tag策略
registry-poison：[L0]检查registry访问控制

- [ ] **步骤 6：创建AS-7持久化（3个模式）**

webhook-backdoor：platforms:[k8s]，[L0]kubectl get validatingwebhook
cronjob-persist：platforms:[k8s]，[L0]kubectl get cronjobs
docker-volume-persist：platforms:[docker]，[L0]docker volume ls

- [ ] **步骤 7：验证14个模式都包含8段和L层标注**

```bash
for dir in auth network data dos supply persist; do
  for f in GenCPT-suite/attack-patterns/$dir/*/SKILL.md; do
    echo "=== $(basename $(dirname $f)) ==="
    grep -c "^## [1-8]\." "$f"
  done
done
```

预期：每个文件8段≥8

- [ ] **步骤 8：Commit**

```bash
cd GenCPT-suite && git add attack-patterns/ && git commit -m "feat: add AS-2 to AS-7 attack patterns (14 modes)"
```

---

### 任务 4：cross-ref SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/cross-ref/SKILL.md`

- [ ] **步骤 1：创建cross-ref SKILL.md**

读取设计文档11.7节和5.3节，写完整SKILL.md。

核心工作流：
1. 读取Phase 2合规结果+Phase 1侦察数据+三库文件
2. 执行3条核心查询：XREF-001合规→攻击假设前置条件、XREF-002叠加放大（≥3种违规叠加→风险放大）、XREF-003攻击假设→侦察结果比对
3. 标记五态（[x][?][-][!][ ]），每个[x][?]必须生成ATK-CAND
4. 写入知识图谱边（compliance→attack映射）和4个输出文件

MUST输出：knowledge_graph/edges/cross_ref.json、evidence/cross-ref/cross_ref_summary.md、evidence/cross-ref/risk_amplification.md、evidence/cross-ref/prerequisite_signals.md

设计文档行号参考：11.7（行1305-1330）、5.3（行318-351）

- [ ] **步骤 2：验证**

```bash
grep -c "XREF-00[123]\|五态\|交叉关联\|MUST" GenCPT-suite/skills/cross-ref/SKILL.md
```

预期：≥5

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/cross-ref/ && git commit -m "feat: add cross-ref SKILL.md (Phase 3)"
```

---

### 任务 5：attack-pattern + attack-reasoning SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/attack-pattern/SKILL.md`
- 创建：`GenCPT-suite/skills/attack-reasoning/SKILL.md`

- [ ] **步骤 1：创建attack-pattern SKILL.md**

读取设计文档11.8节，写完整SKILL.md。

核心工作流：
1. 扫描触发条件：读Phase 1+2+3结果，对照_index.md条件触发读取表，确定需Read的模式文件，按scope过滤platforms
2. 逐个模式验证：Read模式文件→执行探测命令（按L0/L1，按审批级别）→判断前置条件→满足时L2攻击验证→收集差分证明→生成ATK-CAND
3. 处理未匹配信号：标[?]，写unmatched_signals.md
4. 五态闭环：所有[ ]必须消灭

**关键：不准凭记忆出攻击命令！必须Read对应SKILL.md。命令必须标注L0/L1/L2/L3执行上下文。**

MUST输出：evidence/attack/pattern-hits.md、evidence/attack/unmatched_signals.md、knowledge_graph/edges/attack.json

设计文档行号参考：11.8（行1223-1255）、5.1（行298-317）、5.2（行304-332）、5.16（行965-1120）

- [ ] **步骤 2：创建attack-reasoning SKILL.md**

读取设计文档11.9节，写完整SKILL.md。

核心工作流：
1. 确定推理范围：读[?]候选+Phase 3高关联度+情节记忆（只影响优先级）
2. 逐攻击面推理：读attack-surface-model.md，对未覆盖攻击面逐个推理，探测命令基于当前环境实际状态构造
3. 生成ATK-CAND：source标记llm_reasoning，写入insights.md
4. 情节记忆记录：写入recommendations.md

**关键：不准凭记忆出攻击命令！探测命令必须基于当前环境实际状态构造。**

MUST输出：evidence/attack/reasoning-hits.md、evidence/insights.md、knowledge_graph/edges/attack.json（追加）、episodic_memory/recommendations.md

- [ ] **步骤 3：验证**

```bash
grep -c "不准凭记忆\|L0\|L1\|L2\|ATK-CAND\|MUST" GenCPT-suite/skills/attack-pattern/SKILL.md
grep -c "不准凭记忆\|llm_reasoning\|insights\|情节记忆" GenCPT-suite/skills/attack-reasoning/SKILL.md
```

预期：各≥4

- [ ] **步骤 4：Commit**

```bash
cd GenCPT-suite && git add skills/attack-pattern/ skills/attack-reasoning/ && git commit -m "feat: add attack-pattern and attack-reasoning SKILL.md (Phase 4a/4b)"
```

---

### 任务 6：chain-builder + chain-verify + poc-generator SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/chain-builder/SKILL.md`
- 创建：`GenCPT-suite/skills/chain-verify/SKILL.md`
- 创建：`GenCPT-suite/skills/poc-generator/SKILL.md`

- [ ] **步骤 1：创建chain-builder SKILL.md**

读取设计文档11.10节，写完整SKILL.md。核心：识别链起点→知识图谱查找可达攻击点→判断可行性→评估影响→分配置信度→编号CHAIN-xxx

MUST输出：evidence/chains/chain_builder.md、knowledge_graph/edges/cross_ref.json（追加attack_chain边）

- [ ] **步骤 2：创建chain-verify SKILL.md**

读取设计文档11.11节，写完整SKILL.md。核心：逐链验证每一步→收集差分证明→审批门控（L1-L5）→被阻断链检查绕过策略→差分证明验证（C1/C2/C3可信度）

**关键：审批门控5级+超时降级规则必须明确。差分证明每步标注L0/L1/L2。**

MUST输出：evidence/chains/chain_verification.md

- [ ] **步骤 3：创建poc-generator SKILL.md**

读取设计文档11.12节，写完整SKILL.md。核心：为confirmed和condition_met项生成可执行POC脚本和操作说明。

POC格式严格遵循设计文档5.16.5节：
- 可信度标注（C1/C2/C3）
- 验证层级标注（L0/L1/L2/L3）
- 每步标注执行上下文[L0]/[L1]/[L2]
- L3条件验证的POC标⚠️+不可安全复现原因

MUST输出：evidence/poc/poc_scripts/、evidence/poc/poc_readme.md

- [ ] **步骤 4：验证3个文件**

```bash
for skill in chain-builder chain-verify poc-generator; do
  echo "=== $skill ===" && grep -c "MUST\|L0\|L1\|L2\|CHAIN\|POC\|审批\|可信度" GenCPT-suite/skills/$skill/SKILL.md
done
```

预期：chain-builder≥3, chain-verify≥4, poc-generator≥3

- [ ] **步骤 5：Commit**

```bash
cd GenCPT-suite && git add skills/chain-builder/ skills/chain-verify/ skills/poc-generator/ && git commit -m "feat: add chain-builder, chain-verify, poc-generator SKILL.md (Phase 5/6/7)"
```

---

### 任务 7：hypothesis-libraries内容填充

**文件：**
- 修改：`GenCPT-suite/hypothesis-libraries/compliance-hypotheses.md`（骨架→完整内容）
- 修改：`GenCPT-suite/hypothesis-libraries/attack-hypotheses.md`（骨架→完整内容）
- 修改：`GenCPT-suite/hypothesis-libraries/cross-ref-queries.md`（骨架→完整内容）

- [ ] **步骤 1：填充compliance-hypotheses.md**

从K8s和Docker合规规则中提取关键违规→攻击前置条件的映射。例如：
- K8s-5.2.1(privileged=true) → escape/capability-privesc + escape/socket-escape
- K8s-5.2.3(docker.sock挂载) → escape/socket-escape
- Docker G_5_25(容器特权模式) → escape/capability-privesc

每个假设卡片包含：假设ID、违规族、映射的攻击模式、严重等级、前置条件示例。至少20条映射。

- [ ] **步骤 2：填充attack-hypotheses.md**

为每个攻击模式写前置条件+探测命令+验证步骤+绕过策略+证伪条件+合规映射。与attack-patterns中的SKILL.md对应但更简洁（假设库是摘要索引，不是完整8段格式）。至少20条。

- [ ] **步骤 3：填充cross-ref-queries.md**

写3条核心查询模板和5-10个交叉关联示例：
- XREF-001：合规违规→攻击假设前置条件
- XREF-002：叠加放大（≥3种违规）
- XREF-003：攻击假设→侦察结果比对

- [ ] **步骤 4：验证三库内容**

```bash
wc -l GenCPT-suite/hypothesis-libraries/*.md
```

预期：每个文件≥30行（包含格式说明+示例+实际假设条目）

- [ ] **步骤 5：Commit**

```bash
cd GenCPT-suite && git add hypothesis-libraries/ && git commit -m "feat: fill hypothesis libraries with compliance-attack mappings"
```

---

### 任务 8：端到端验证

- [ ] **步骤 1：验证攻击模式总数**

```bash
find GenCPT-suite/attack-patterns -name "SKILL.md" -not -path "*/_learned/*" | wc -l
```

预期：25个（7+4+4+3+2+2+3，不含_learned目录）

- [ ] **步骤 2：验证所有SKILL.md包含8段格式**

```bash
for f in GenCPT-suite/attack-patterns/*/SKILL.md GenCPT-suite/attack-patterns/*/*/SKILL.md; do
  [ -f "$f" ] && echo "$(basename $(dirname $f)): $(grep -c '^## [1-8]\.' $f) sections"
done
```

预期：每个文件8段

- [ ] **步骤 3：验证6个Phase SKILL.md**

```bash
for skill in cross-ref attack-pattern attack-reasoning chain-builder chain-verify poc-generator; do
  [ -f "GenCPT-suite/skills/$skill/SKILL.md" ] && echo "$skill: OK" || echo "$skill: MISSING"
done
```

预期：6个全部OK

- [ ] **步骤 4：Commit**

```bash
cd GenCPT-suite && git add -A && git status
```

预期：无未提交更改

## 补充实现（V3.1增强）

以下增强功能在 V3 基础上追加，不影响原有功能：

- [x] **建议1：attack-pattern LLM 语义补充扫描** — 步骤1末尾新增第8步，LLM 扫描 _index.md 中所有模式名 + mapped_compliance_families，标记 `[?] 疑似相关`，≤500 tokens。实现位置：`GenCPT-suite/skills/attack-pattern/SKILL.md`。commit: 未提交（工作区变更）
- [x] **建议5：差分证明 L0 强制（poc-generator）** — POC 脚本按攻击类型强制 L0 观测要求，AS-1/AS-5/AS-7 必须三段式，不满足标注可信度降级。实现位置：`GenCPT-suite/skills/poc-generator/SKILL.md`。commit: 未提交（工作区变更）