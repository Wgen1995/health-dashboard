# 链标准（CHAIN_STANDARD）

> 适用范围：Phase 5 链构建 + Phase 6 链验证 + Phase 8b 链报告。
> 此文件预备 Phase 5 使用，本子项目 A 写规范不执行验证。
> 约束力：**强制**。违反即触发反幻觉#32（链深超 5 必报截断声明）。

---

## 1. 链节点格式

每条链 CHAIN-xxx 包含节点结构（写入 nodes/chains.json）：

```json
{
  "node_type": "chain",
  "chain_id": "CHAIN-001",
  "steps": [
    {"step": 1, "node_ref": "CAND-001", "verdict": "C2_condition_met", "evidence_ref": "evidence/phase2a/CAND-001.md"},
    {"step": 2, "node_ref": "compliance_violation/G1-01 + ATK-HYP-008", "verdict": "C3_clue", "evidence_ref": "..."},
    {"step": 3, "node_ref": "CAND-015", "verdict": "C3_clue", "evidence_ref": "..."},
    {"step": 4, "node_ref": "CAND-020", "verdict": "C3_llm_reasoning", "evidence_ref": "..."}
  ],
  "total_impact": "未鉴权远程代码执行",
  "reachability": "confirmed_partial",
  "depth": 4,
  "audit_state": "[ ]"
}
```

## 2. 可达性判定规则

| reachability | 含义 | 判定条件 |
|-------------|------|---------|
| `confirmed_full` | 全步骤路径已 Read 验证 | 每步骤 evidence_chain 都有完整 propagation 链（每跳有 Read 证据） |
| `confirmed_partial` | 部分步骤已验证 + 部分 T2 推断 | 主路径已 Read，部分远端节点假设待验证 |
| `theoretical` | 全部 T3 推断 | 无 Read 证据，仅业务逻辑推演 |

## 3. 深度上限

- 默认 `chain-depth = 5`（Phase 5 输入参数，默认 5）
- 硬上限 `chain-depth = 8`（防组合爆炸）
- 超过 5 必须在 CHAIN-*.md 中写"深度截断声明"段
- 超过 8 即停止扩展，标 `[!] depth_exceeded`

## 4. 链步骤边

每条链步骤写入 edges/chain_steps.json：

```json
{
  "edge_type": "chain_steps",
  "chain_id": "CHAIN-001",
  "step_order": 1,
  "node_ref": "CAND-001",
  "verdict": "C2_condition_met"
}
```

## 5. 反幻觉引用

| # | 引用 |
|---|------|
| #32 | 链深超 5 必报"深度截断"声明——本规范 §3 强制 |
| #36 | 挑战者不许读分析方推理链证据链——链步骤 verdict 必引 file:line |

## 6. 边界说明

本规范仅定义链结构标准，不定义如何构建链（那是 Phase 5 子项目 E 的 SKILL.md 职责）。本规范被 Phase 5/6/8b 子技能引用。