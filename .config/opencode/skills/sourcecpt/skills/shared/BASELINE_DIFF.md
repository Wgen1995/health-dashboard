# Baseline 对比协议（BASELINE_DIFF）

> 适用范围：持续审计模式（mode=increment）。
> 此文件预备子项目 H（持续审计 + Phase 1c）使用，本子项目 A 写规范不执行增量审计。
> 约束力：**强制**。违反即触发反幻觉#7、#31。

---

## 1. 基线建立

每次 full 审计完成后，复制核心产物到 `baseline/{date}/`：

| 复制项 | 原路径 | 目标路径 |
|--------|--------|---------|
| audit_log.json | `{session_dir}/audit_log.json` | `baseline/{date}/audit_log.json` |
| vuln_candidates.json | `{session_dir}/knowledge_graph/nodes/vuln_candidates.json` | `baseline/{date}/vuln_candidates.json` |
| compliance_violations.json（若启用） | `{session_dir}/knowledge_graph/nodes/compliance_violations.json` | `baseline/{date}/compliance_violations.json` |
| pattern_matches.json | `{session_dir}/knowledge_graph/nodes/pattern_matches.json` | `baseline/{date}/pattern_matches.json` |
| verified_vulns.json | `{session_dir}/knowledge_graph/nodes/verified_vulns.json` | `baseline/{date}/verified_vulns.json` |
| suite_version | （直接写） | `baseline/suite_version.json` |

```json
// baseline/suite_version.json
{
  "suite_version": "V1.0",
  "established_at": "ISO8601",
  "established_by_session": "uuid"
}
```

---

## 2. 增量对比

下次 increment 审计时，Phase 0 需对比当前状态 vs `baseline/{last_date}/`：

### 2.1 git diff 文件清单

```bash
git -C {project-path} diff --name-only {changed-since}..HEAD
```

只审 git diff 触及文件 + 调用图邻居（深度 2，模块级骨架的相邻模块）。

### 2.2 diff 已经确认的漏洞

读取 `baseline/{last_date}/verified_vulns.json`，对比当前新增的 verified_vulns：

- 过去已确认的（baseline 中存在）不再重报
- 新出现的新 VULN → 入 `reports/pentest_report.md` 的"新增漏洞"部分
- 消失的 VULN（baseline 有但当前 CAND 找不到了）→ 入"已修复"部分（需人工确认是否真修复还是漏报）

### 2.3 diff 合规违规

读取 `baseline/{last_date}/compliance_violations.json`（若启用），对比当前：
- 过去已记录的 fail 项不再重报（除非新引入）
- 新 fail 入报告"新增违规"部分
- 已消失的 fail 入"已修复合规"部分

---

## 3. 报告含 baseline diff 章节

Phase 8d 全景报告必含 "baseline diff" 章节模板：

```markdown
## 章 11. Baseline 对比（持续审计模式）

| 项 | 数 | 说明 |
|---|---|---|
| 新增漏洞 | N | 比上次审计新出现的 VULN/CLUE |
| 已修复漏洞 | M | 上次有但本次找不到（需人工确认是否真修复） |
| 仍存在未修 | K | 上次有本次仍在 |
| 新增合规违规 | L | （若启用 --with-compliance） |
| 已修复合规违规 | J | （若启用 --with-compliance） |

### 11.1 新增漏洞详
[列出新增 VULN/CLUE]

### 11.2 待确认修复（消失的旧漏洞）
[列出 baseline 有但本次找不到的项，提示甲方确认是已修复还是漏报]
```

---

## 4. 版本兼容

| 情形 | 处理 |
|------|------|
| suite_version 一致 | 直接 diff 对比 |
| suite_version 不一致 | 降级为趋势对比——仅提示"上次基线版本不一致，无法精确 diff，差异仅供趋势参考" |
| 无 baseline | 跳过本章，报告标"首次审计无 baseline" |

**核心原则**：baseline 仅作 diff 对比，**baseline 永不替代当前审计结果**。即使版本一致，当前审计发现是权威，baseline 仅提供"上次有没有"的对比依据。

---

## 5. 反幻觉引用

| # | 引用 |
|---|------|
| #7 | baseline 永不替代当前——本规范 §4 核心原则 |
| #31 | 增量审计不许降级 baseline——本规范 §4 核心原则 |

baseline 的作用是**对比参考**不是**降级依据**。当前审计发现什么就是什么，baseline 没找到不代表本次不该找，baseline 找过的本次仍要找（除非确证已修复）。