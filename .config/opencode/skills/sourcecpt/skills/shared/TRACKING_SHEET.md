# Excel 跟踪表字段定义与类型映射（TRACKING_SHEET）

> 适用范围：Phase 8d 全景报告产出 Excel 跟踪表（issue_tracking.csv + .xlsx）。
> 此文件预备 Phase 8d 使用，本子项目 A 写规范不产出实际跟踪表。
> 约束力：**强制**。违反即触发反幻觉#58、#59。

---

## §1 跟踪表字段定义（19 列）

CSV/xlsx 的 Issues sheet 必含以下 19 列：

| # | 列名 | 类型 | 必填 | 示例 |
|---|------|------|------|------|
| 1 | ID | string | 是 | VULN-001 |
| 2 | 类型编号 | string | 是 | A01 / CWE-89 |
| 3 | 类型名称 | string | 是 | SQL注入 |
| 4 | 子类型 | string | 是 | 字符串拼接式 |
| 5 | 标题 | string | 是 | SQL注入-UserController.getUser |
| 6 | 可信度 | enum | 是 | C1 / C2 / C3 |
| 7 | 位置 | string | 是 | UserDao.java:102 |
| 8 | 入口 | string | 是 | GET /api/users/{id} |
| 9 | 鉴权依赖 | enum | 是 | 是 / 否 |
| 10 | 影响 | string | 是 | 任何人窃取用户口令哈希 |
| 11 | 链 ID | string | 否 | CHAIN-001 (step1) |
| 12 | Sibling | string | 否 | VULN-002..005 |
| 13 | POC 引用 | string | 否 | POC-001.http |
| 14 | 修复优先级 | enum | 是 | P0 / P1 / P2 / P3 |
| 15 | 修复建议引用 | string | 否 | patch_suggestion.md#vuln-001 |
| 16 | 状态 | enum | 是 | 未修复 / 已修复 / 已确认 / 已驳回 / 待复检 |
| 17 | 残余风险 | string | 否 | 历史数据已泄露 |
| 18 | 审计日期 | ISO8601 | 是 | 2026-06-28 |
| 19 | 备注 | string | 否 | (额外信息) |

### 强制约束

- **状态初值必须为 "未修复"**，不许留空（反幻觉#58）
- 类型字段（#2/3/4）必三列俱全，类型名称必从 §2 映射表查填，不许凭记忆瞎填（反幻觉#59）

## §2 类型编号 → 类型名称 → 子类型映射表（20 类）

| CWE 编号 | 类型名称 | 常见子类型 |
|---------|---------|----------|
| CWE-89 | SQL注入 | 字符串拼接式 / ORM原生SQL式 / MyBatis ${}占位符式 |
| CWE-78 | 命令注入 | Runtime.exec拼接式 / ProcessBuilder拼接式 / Shell -c注入式 |
| CWE-22 | 路径遍历 | tar解压未过滤式 / 用户输入拼路径式 / 符号链接绕过式 |
| CWE-502 | 反序列化 | Java原生式 / fastjson式 / XStream式 / Log4Shell JNDI式 |
| CWE-79 | XSS | 反射式 / 存储式 / DOM式 |
| CWE-918 | SSRF | URL拼接式 / 重定向跟随式 / DNS重绑定式 |
| CWE-352 | CSRF | 无Token式 / 弱Token式 / Referer缺失式 |
| CWE-287 | 认证缺陷 | 口令硬编码式 / 弱口令策略 / 会话固定式 |
| CWE-862 | 授权缺失 | 路径无鉴权式 / 水平越权IDOR式 / 垂直越权式 |
| CWE-798 | 硬编码凭证 | 口令 / APIKey / Token / 连接串式 |
| CWE-327 | 弱加密算法 | MD5 / SHA1 / DES / RC4式 |
| CWE-330 | 弱随机数 | new Random / Math.random式 |
| CWE-209 | 信息泄露 | 详细错误回显 / 调试端点 / 版本信息式 |
| CWE-532 | 日志泄露 | 日志打印请求体 / 敏感字段式 |
| CWE-611 | XXE | DocumentBuilder / SAXParser式 |
| CWE-434 | 文件上传 | 无类型白名单 / 路径穿越式 |
| CWE-94 | 表达式注入 | SpEL / OGNL / EL / MVEL式 |
| CWE-1336 | SSTI | Freemarker / Velocity / Thymeleaf式 |
| CWE-74 | 通用注入 | CRLF / 邮件头 / XPath / LDAP / NoSQL式 |
| CWE-1004 | 口令技术控制 | 口令复杂度不足 / 无强度校验式 |

**新增 CWE 规则**：若 Phase 9 模式进化晋升新模式，新 CWE 必追加到此映射表，保持三列俱全。

## §3 xlsx 3 sheet 结构

| Sheet | 内容 | 必填列 |
|-------|------|-------|
| Issues | 全部发现（C1+C2+C3+NOVULN+SUSPENDED 各一行） | §1 全 19 列 |
| Chains | 全部漏洞链 + 步骤 | chain_id / 总影响 / 步数 / 各步骤 node_ref / 最大可信度 |
| Summary | 统计摘要 | 可信度分布（C1/C2/C3 数）/ Top 5 风险 / Phase 6 推翻数 / attack 面覆盖矩阵摘要 |

## §4 CSV 与 xlsx 选择

- `issue_tracking.csv` — **必产**（最小依赖，Excel/Sheets 直接打开）
- `issue_tracking.xlsx` — **可选**（含 3 sheet 格式，需 openpyxl 等环境，无则回退 csv）

若环境支持 xlsx 生成（通过 opencode bash 调用 python + openpyxl），产 xlsx；否则仅产 csv + 在 pentest_report.md 中声明"xlsx 因环境未生成"。

## §5 反幻觉引用

| # | 引用 |
|---|------|
| #58 | 跟踪表必含完整字段——本规范 §1 强制状态初值"未修复" |
| #59 | 跟踪表类型必含三列——本规范 §2 映射表必查 |