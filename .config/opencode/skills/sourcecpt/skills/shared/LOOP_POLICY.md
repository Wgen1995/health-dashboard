# Loop 使用策略（LOOP_POLICY）

> 适用范围：所有 Phase 子流程中是否启用循环。
> 借鉴：loop engineering（Addy Osmani、Boris Cherny）部分采纳——主体不 loop，局部≤3 轮。
> 核心原则：loop 适合"对错清晰可验证"的活；安全审计的鉴权与支付逻辑不该让 loop 碰，故主体不 loop。
> 约束力：**强制**。

---

## 1. 三个适用 loop 的子流程（局部，≤3 轮，Token 硬上限）

| 子流程 | loop 触发条件 | 验收方 | 停止条件 |
|--------|-------------|--------|---------|
| 1b 鉴权遗漏反查 | Phase 1b 全局识别完后 | 独立子代理反查是否有遗漏全局过滤 | 新增项 ≤0 或 ≤3 轮 |
| 2b uncertain 扩读 | Phase 2b uncertain 命中项 | 子代理扩读 ±100 行 | verdict 收敛或 ≤3 轮 |
| 6 对抗仲裁 | Phase 6 第一轮挑战后争议 | 第三方仲裁子代理 | 共识达成或 ≤3 轮 |

### 1.1 每轮 self-contained 原则

下轮子代理只读上轮结论+源码，**不读上轮推理过程**。这避免"自我合理化"持续加固幻觉。

### 1.2 每轮必新 Read 证据

每轮 verdict 必须基于本轮 Read 的源码，不许"上轮已确认 + 本轮小补强"。状态文件要求每轮写新 Read 证据。

### 1.3 状态文件必写盘

每个 loop 子流程的状态文件必写入 `evidence/phase{N}_loop/{item_id}/loop_state.json`：

```json
{
  "round": 1,
  "timestamp": "ISO8601",
  "analyzer_verdict": "...",
  "challenger_verdict": "...",
  "new_read_evidence": ["UserController.java:50±100", "..."],
  "arbitration_verdict": null
}
```

---

## 2. 禁用 loop 的核心判定

| 禁用项 | 理由 |
|--------|------|
| Phase 4a 证伪条件检查 | 单次精确判定，不许循环 |
| Phase 4b ReAct 已是循环 | Thought-Action-Observation 不嵌套外部 loop |
| Phase 5 链构建 | 组合优化不许"不合格重来"，重定义链路成本更高 |
| C1/C2/C3 升级 | 必须人审 Echo，不许 loop 自升 |
| Pipeline 顶层 | 9 Phase 顺序调度，不许 Agent 自动跳 Phase（破坏审计 trail） |
| 已完成 Phase | 已写盘的 Phase 不许 loop 重跑，失败用断点续传从最后成功 WU 恢复 |

---

## 3. Loop 硬停止条件

| 限制 | 值 | 触发动作 |
|------|----|---------|
| 单子流程 loop 轮次 | ≤3 | 超出即停止，写入 loop_state.json 标 "round_limit_reached" |
| 单 loop Token | ≤50000 | 触及写阻塞报告 evidence/phase{N}_loop/{item_id}/budget_limit.md |
| 全 Pipeline loop 累计 Token | ≤5 亿 | 触及写 budget_exceeded.md |
| 全 Pipeline 总 Token | ≤50 亿 | 触及写 budget_exceeded.md + 状态保存 + 人介入 |
| WU 时间硬限 | ≤300000 ms（opencode Task 时长上限） | 超时即 WU 失败，触发断点续传 |
| 单 sink 污点追踪跳数 | ≤5 | 超过即标 `path_too_deep` 待 Phase 4b 推理 |

---

## 4. 反幻觉引用

| # | 引用 |
|---|------|
| #30 | 不许循环重跑已完成 Phase——本规范 §2 禁用项 |
| #38 | 对抗 loop 每轮必新 Read 证据——本规范 §1.2 强制 |
| #40 | SUSPENDED 不许擅升 VULN——对抗 loop 争议未决必须显式 `[?]` |

## 5. 与断点续传的关系

断点续传不是 loop——断点续传保留已成功 WU 不重跑，只跑未完成的；loop 是"不合格重来"。安全审计单次几亿 token 重跑成本爆炸，故**断点续传 > loop**。

参考 SRC_ACCESS §5 与 progress.json 状态机（OUTPUT_STANDARD §3.2）。