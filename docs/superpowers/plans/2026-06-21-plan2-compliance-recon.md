# Container Pentest Suite V3 — 计划2: 合规规则 + 侦察与合规Phase

> **面向 AI 代理的工作者：** 必需子技能：使用 superpowers:subagent-driven-development（推荐）或 superpowers:executing-plans 逐任务实现此计划。步骤使用复选框（`- [ ]`）语法来跟踪进度。

**目标：** 创建44个合规规则文件、3个compliance子技能SKILL.md、recon和recon-source的SKILL.md

**架构：** 合规规则按CIS Benchmark分组存储为Markdown，每个文件包含规则编号、描述、检查命令（L0）、判定标准、修复建议。compliance子技能按批次读取规则文件执行检测。recon通过SSH收集环境信息，recon-source扫描本地源码。

**设计文档：** `/root/docs/superpowers/specs/2026-06-20-GenCPT-suite-v3-design.md`
**依赖：** 计划1已完成（目录骨架和共享规则已创建）

---

## 文件结构

```
GenCPT-suite/
├── compliance-rules/
│   ├── kubernetes/          # 29个分组文件 + _index.md（计划1已完成）
│   │   ├── G_1_1_api_server_files.md    # 7条
│   │   ├── G_1_2_api_server_auth.md     # 24条
│   │   ├── ... (27 more files)
│   ├── docker/              # 7个分组文件 + _index.md（计划1已完成）
│   │   ├── G_1_runtime_config.md        # 5条
│   │   ├── ... (6 more files)
│   └── containerd/          # 5个分组文件 + _index.md（计划1已完成）
│       ├── G_1_runtime_config.md        # 3条
│       ├── ... (4 more files)
├── skills/
│   ├── recon/SKILL.md
│   ├── recon-source/SKILL.md
│   ├── k8s-compliance/SKILL.md
│   ├── docker-compliance/SKILL.md
│   └── containerd-compliance/SKILL.md
```

---

### 任务 1：Kubernetes合规规则 — G_1组（API Server）

**文件：**
- 创建：`GenCPT-suite/compliance-rules/kubernetes/G_1_1_api_server_files.md`（7条）
- 创建：`GenCPT-suite/compliance-rules/kubernetes/G_1_2_api_server_auth.md`（24条）
- 创建：`GenCPT-suite/compliance-rules/kubernetes/G_1_3_api_server_dos.md`（1条）
- 创建：`GenCPT-suite/compliance-rules/kubernetes/G_1_4_api_server_leak.md`（3条）
- 创建：`GenCPT-suite/compliance-rules/kubernetes/G_1_5_api_server_log.md`（6条）
- 创建：`container-pest-suite/compliance-rules/kubernetes/G_1_6_api_server_ssl.md`（1条）

每个文件格式严格遵循：

```markdown
# G_1_1 API Server 文件权限 (7条)

## 1.1.1 确保 API Server pod specification 文件权限设为 644 或更严格

**检查命令 [L0]**:
```bash
stat -c %a /etc/kubernetes/manifests/kube-apiserver.yaml
```

**期望值**: 644 或更严格（600）
**判定**: pass=≤644, fail=>644, na=文件不存在且非静态Pod
**修复建议**: chmod 644 /etc/kubernetes/manifests/kube-apiserver.yaml
**CIS映射**: 1.1.1
**攻击面关联**: AS-2 认证授权
```

读取设计文档4节目录结构获取分组列表，对每个分组：
- 查阅CIS Kubernetes Benchmark v1.8（或对应版本）获取每条规则的编号、描述、检查命令
- 每条规则包含：编号、描述、检查命令（标注L0/L1执行上下文）、期望值、判定标准（pass/fail/na）、修复建议、CIS映射、攻击面关联
- `na`判定条件明确写出（如"文件不存在且非静态Pod"）

- [ ] **步骤 1：创建G_1_1到G_1_6共6个文件**

按上述格式为每个文件编写所有规则。G_1_2最大（24条），需仔细编写。

设计文档行号参考：4节目录结构（行159-191）

- [ ] **步骤 2：验证G_1组总规则数为42（7+24+1+3+6+1）**

```bash
grep -c "^\*\*检查命令" GenCPT-suite/compliance-rules/kubernetes/G_1_*.md
```

预期：总计41条规则

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add compliance-rules/kubernetes/ && git commit -m "feat: add K8s G_1 API Server compliance rules (42 checks)"
```

---

### 任务 2：Kubernetes合规规则 — G_2到G_8组

**文件：**
- 创建：G_2_1到G_2_4（4+2+4+1=11条）, G_3_1到G_3_2（4+2=6条）, G_4_1到G_4_3（4+7+1=12条）, G_5_1到G_5_6（8+6+2+5+1+1=23条）, G_6_1到G_6_2（2+1=3条）, G_7_1到G_7_2（15+2=17条）, G_8_1到G_8_4（2+13+2+4=21条）
- 共22个文件

文件名对照（设计文档V3同步更新后）：
- G_2: Etcd（G_2_1_etcd_files, G_2_2_etcd_config, G_2_3_etcd_security, G_2_4_etcd_network）
- G_3: Controller Manager + Scheduler（G_3_1_cm_security, G_3_2_scheduler_security）
- G_4: Kubelet Auth（G_4_1_kubelet_auth, G_4_2_kubelet_authz, G_4_3_kubelet_config）
- G_5: Kubelet Runtime/Streaming/TLS/Leak/DoS/System
- G_6: Network Policies（G_6_1_network_policies, G_6_2_network_policy_leak）
- G_7: Pod安全 + Container Runtime（G_7_1_pod_security, G_7_2_container_runtime）
- G_8: Secret管理/RBAC/Security Context/NetworkPolicies Advanced

- [ ] **步骤 1：创建G_2到G_8共22个文件**

按任务1的格式规范，为每个分组编写合规规则。关键要点：
- G_5（Kubelet）是最大组（23条），Kubelet运行时/Streaming/TLS/防泄露/防DoS/系统配置
- G_7（Pod安全+Container Runtime）涉及privileged容器、hostPath挂载等——这些是攻击验证的关键入口，需在attack-hypotheses中关联
- G_8（Secret管理/RBAC/Security Context/NetworkPolicies Advanced）涉及权限和数据泄露——关联AS-2和AS-4

设计文档行号参考：4节目录结构（行169-191）

- [ ] **步骤 2：验证K8s总规则数为135**

```bash
total=0; for f in compliance-rules/kubernetes/G_*.md; do count=$(grep -c "^\*\*检查命令" "$f" 2>/dev/null || echo 0); total=$((total+count)); done; echo "K8s total rules: $total"
```

预期：135条

- [ ] **步骤 3：更新kubernetes/_index.md使规则数与实际文件一致**

检查_index.md中的条目数是否等于实际规则数，如有差异则修正。

- [ ] **步骤 4：Commit**

```bash
cd GenCPT-suite && git add compliance-rules/kubernetes/ && git commit -m "feat: add K8s G_2-G_8 compliance rules (93 checks, total 135)"
```

---

### 任务 3：Docker合规规则（64条7组）

**文件：**
- 创建：7个分组文件（G_1到G_7）

- [ ] **步骤 1：创建Docker 7个合规规则文件**

按任务1格式规范编写。Docker规则参考CIS Docker Benchmark v1.3.0：
- G_1_runtime_config.md（5条）：运行环境配置
- G_2_daemon_params.md（11条）：守护进程参数（网络配置、日志、TLS等）
- G_3_file_perms.md（10条）：文件权限（docker.sock、证书、配置文件权限）
- G_4_image_build.md（7条）：镜像构建（USER指令、apt-get缓存清理、无sudo）
- G_5_container_runtime.md（26条）：容器运行时（特权模式、网络、capabilities、seccomp等）
- G_6_container_ops.md（2条）：容器运维
- G_7_cluster_config.md（3条）：集群配置

关键要点：
- G_5最大（26条），涉及容器逃逸、capabilities等——直接关联AS-1攻击面
- G_3涉及docker.sock权限——是socket-escape攻击模式的前置条件

设计文档行号参考：4节目录结构（行192-200）

- [ ] **步骤 2：验证Docker总规则数为64**

```bash
total=0; for f in compliance-rules/docker/G_*.md; do count=$(grep -c "^\*\*检查命令" "$f" 2>/dev/null || echo 0); total=$((total+count)); done; echo "Docker total rules: $total"
```

预期：64条

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add compliance-rules/docker/ && git commit -m "feat: add Docker compliance rules (64 checks)"
```

---

### 任务 4：Containerd合规规则（28条5组）

**文件：**
- 创建：5个分组文件（G_1到G_5）

- [ ] **步骤 1：创建Containerd 5个合规规则文件**

- G_1_runtime_config.md（3条）
- G_2_daemon_params.md（3条）
- G_3_file_perms.md（10条）
- G_4_image_build.md（2条）
- G_5_container_runtime.md（10条）

- [ ] **步骤 2：验证Containerd总规则数为28**

```bash
total=0; for f in compliance-rules/containerd/G_*.md; do count=$(grep -c "^\*\*检查命令" "$f" 2>/dev/null || echo 0); total=$((total+count)); done; echo "Containerd total rules: $total"
```

预期：28条

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add compliance-rules/containerd/ && git commit -m "feat: add Containerd compliance rules (28 checks)"
```

---

### 任务 5：recon SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/recon/SKILL.md`

- [ ] **步骤 1：创建recon SKILL.md**

读取设计文档11.2节（recon Phase 1a），写完整SKILL.md。内容包含：

YAML frontmatter：
```yaml
---
name: recon
description: >
  容器环境全光谱侦察。通过SSH远程收集集群结构、Pod规范、安全上下文、服务暴露等信息。
  使用场景：渗透测试第一步，绘制攻击面地图。
  不使用场景：已有完整侦察数据、只做单点合规检查。
---
```

核心工作流4步：
1. 环境指纹识别（分批SSH命令，每批5-7条间隔2秒，含OS/arch收集）
2. 安全上下文提取（从原始数据提取securityContext/volumes/hostNetwork等）
3. 数据写入（按类型分片写入知识图谱节点和边，生成recon_summary.md）
4. 环境指纹生成（env_hash写入session_config.json，含os_type/arch/kernel_version）

MUST输出：knowledge_graph/nodes/（6个JSON）、knowledge_graph/edges/infra.json、evidence/recon/recon_summary.md、evidence/recon/raw/

检查点4项：①主机/容器/Pod/SA清单非空 ②安全上下文提取完成 ③环境指纹已生成 ④QA结构校验通过

分批策略：Pod/容器扫描每批100个间隔2秒，大集群（300+Pod）先获取ns列表再按ns分批

独立运行参数： `--server prod-k8s-01 --scope k8s,docker`

设计文档行号参考：11.2（行1253-1280）、5.6（行415-455）、5.13（行743-762）、5.16（行965-1030）

- [ ] **步骤 2：验证recon SKILL.md包含MUST输入输出和工作流**

```bash
grep -c "MUST 输入\|MUST 输出\|环境指纹\|安全上下文\|检查点\|L0\|L1" GenCPT-suite/skills/recon/SKILL.md
```

预期：≥6

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/recon/ && git commit -m "feat: add recon SKILL.md (Phase 1a)"
```

---

### 任务 6：recon-source SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/recon-source/SKILL.md`

- [ ] **步骤 1：创建recon-source SKILL.md**

读取设计文档11.3节和5.9节，写完整SKILL.md。

YAML frontmatter包含name、description、使用场景/不使用场景。

核心工作流：
1. 扫描范围确定（只扫描容器安全相关文件：Dockerfile、docker-compose、K8s清单、Helm Chart、CI/CD、.env、.dockerignore）
2. 两级扫描策略执行（≤200全量、>200分级）
3. 结构化摘要提取（每个文件5-10行摘要，总大小≤8000 tokens）
4. 数据写入（source_findings.json、source_edges.json、source_analysis.md、source_scan_stats.md）

MUST输出：knowledge_graph/nodes/source_findings.json、knowledge_graph/edges/source_edges.json、evidence/recon/source_analysis.md、evidence/recon/source_scan_stats.md

检查点：①源码文件列表非空或明确无源码 ②JSON文件结构完整 ③QA结构校验通过

设计文档行号参考：11.3（行1281-1300）、5.9（行577-610）

- [ ] **步骤 2：验证recon-source SKILL.md包含扫描策略和token预算**

```bash
grep -c "扫描\|token\|Dockerfile\|K8s清单\|8000\|200" GenCPT-suite/skills/recon-source/SKILL.md
```

预期：≥4

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/recon-source/ && git commit -m "feat: add recon-source SKILL.md (Phase 1b)"
```

---

### 任务 7：k8s-compliance SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/k8s-compliance/SKILL.md`

- [ ] **步骤 1：创建k8s-compliance SKILL.md**

读取设计文档11.4节和9节，写完整SKILL.md。

核心工作流：
1. 读取合规规则（从compliance-rules/kubernetes/按CIS分组分批加载，每批WU约40-50条）
2. 按分组执行SSH命令检测，每条规则用L0或L1上下文
3. 判定每条规则（pass/fail/warn/na），fail必须附SSH输出和判定理由
4. 数据写入（results.json、summary.md、findings.json的k8s部分、compliance.json的k8s部分）

分批策略：WU-1: G_1组(42条) → WU-2: G_2-G_4(29条) → WU-3: G_5-G_6(26条) → WU-4: G_7-G_8(38条)

三重校验：第一重每批完成后检查规则数=预期数、每条有判定、fail有依据；第二重Phase 2完成后结构校验；第三重报告生成前检查点校验

MUST输出：evidence/compliance/k8s/results.json、evidence/compliance/k8s/summary.md、knowledge_graph/nodes/findings.json（k8s部分）、knowledge_graph/edges/compliance.json（k8s部分）

设计文档行号参考：11.4、9节（行1187-1198）、5.10（行593-632）

- [ ] **步骤 2：验证**

```bash
grep -c "分批\|三重校验\|MUST 输入\|MUST 输出\|G_1\|CIS\|135" GenCPT-suite/skills/k8s-compliance/SKILL.md
```

预期：≥4

- [ ] **步骤 3：Commit**

```bash
cd GenCPT-suite && git add skills/k8s-compliance/ && git commit -m "feat: add k8s-compliance SKILL.md (Phase 2a)"
```

---

### 任务 8：docker-compliance + containerd-compliance SKILL.md

**文件：**
- 创建：`GenCPT-suite/skills/docker-compliance/SKILL.md`
- 创建：`GenCPT-suite/skills/containerd-compliance/SKILL.md`

- [ ] **步骤 1：创建docker-compliance SKILL.md**

结构与k8s-compliance类似，调整：
- 分批策略：WU-1: G_1-G_3(26条) → WU-2: G_4-G_7(38条)
- 规则范围改为docker/
- MUST输入中scope含docker

- [ ] **步骤 2：创建containerd-compliance SKILL.md**

结构类似，调整：
- 分批策略：1批完成（28条）
- 规则范围改为containerd/
- MUST输入中scope含containerd

- [ ] **步骤 3：验证**

```bash
grep -c "64\|分批\|7.*分组" GenCPT-suite/skills/docker-compliance/SKILL.md
grep -c "28\|1批\|5.*分组" GenCPT-suite/skills/containerd-compliance/SKILL.md
```

预期：各≥2

- [ ] **步骤 4：Commit**

```bash
cd GenCPT-suite && git add skills/docker-compliance/ skills/containerd-compliance/ && git commit -m "feat: add docker-compliance and containerd-compliance SKILL.md (Phase 2b/2c)"
```

---

### 任务 9：端到端验证

- [ ] **步骤 1：验证合规规则总数**

```bash
echo "K8s:"; grep -c "^\*\*检查命令" GenCPT-suite/compliance-rules/kubernetes/G_*.md | awk -F: '{sum+=$NF} END{print sum}'
echo "Docker:"; grep -c "^\*\*检查命令" GenCPT-suite/compliance-rules/docker/G_*.md | awk -F: '{sum+=$NF} END{print sum}'
echo "Containerd:"; grep -c "^\*\*检查命令" GenCPT-suite/compliance-rules/containerd/G_*.md | awk -F: '{sum+=$NF} END{print sum}'
```

预期：K8s=135, Docker=64, Containerd=28, 总计227

- [ ] **步骤 2：验证5个SKILL.md文件**

```bash
for skill in recon recon-source k8s-compliance docker-compliance containerd-compliance; do
  echo "=== $skill ===" && head -5 GenCPT-suite/skills/$skill/SKILL.md
done
```

预期：每个文件包含name、description YAML frontmatter

- [ ] **步骤 3：最终Commit**

```bash
cd GenCPT-suite && git add -A && git status
```

预期：无未提交更改或仅有辅助文件

## 补充实现（V3.1增强）

以下增强功能在 V3 基础上追加，不影响原有功能：

- [x] **建议2：cross-ref XREF-001 LLM 动态推理** — XREF-001 执行逻辑第4步后新增第5步，LLM 语义模糊匹配 fail/warn 规则与攻击模式前置条件，≤3000 tokens，输出标记 `[?] LLM推理关联`。实现位置：`GenCPT-suite/skills/cross-ref/SKILL.md`。commit: 未提交（工作区变更）