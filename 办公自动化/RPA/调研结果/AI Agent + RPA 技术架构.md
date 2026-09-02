# Deliverable 5｜AI Agent + RPA 技术架构

## 1. Agent 的正确定位

```text
User/Event
   ↓
Intent Gateway（身份、租户、数据级别、预算）
   ↓
Planner（有限步数，输出 typed plan）
   ↓
Policy Guard（工具/字段/金额/实体/时间/SoD）
   ↓
Workflow（持久状态、审批、补偿）
   ├─ API Tool
   ├─ RPA Tool（受限 UI 动作）
   ├─ Document/OCR Tool
   └─ Human Tool
   ↓
Verifier（独立重算/回读）→ Evidence/Audit
```

Agent 是“不确定性处理器”，不是系统权限的持有者。它只能看到被策略裁剪的 context；Tool Gateway 才是实际授权点。

## 2. Tool 契约

```json
{
  "name": "erp.read.open_items",
  "description": "Read open AR items for an entity and customer",
  "capability": "read",
  "input_schema": {"type":"object","required":["entity_id","customer_id"],"properties":{"entity_id":{"type":"string"},"customer_id":{"type":"string"},"limit":{"type":"integer","maximum":200}}},
  "output_schema_ref": "finance.OpenItem[]",
  "allowed_roles": ["ar_reviewer", "reconciliation_agent"],
  "data_classification": "confidential",
  "side_effect": "none",
  "rate_limit": "60/min",
  "audit_fields": ["entity_id", "customer_id", "result_count"]
}
```

工具分为 `read`、`suggest`、`prepare`、`write`、`approve`；Agent 默认只能调用 `read/suggest`。`write/approve` 必须由确定性 Workflow 节点和人类策略授权，且不能以自然语言参数直接传入金额/账户。

## 3. Agent 如何选择和规划

1. Intent Gateway 将用户请求转成 `intent + entity + time range + data classification`，缺失信息进入澄清任务。
2. Planner 从工具目录检索 capability 和 schema，生成有限步数的 typed plan，不生成任意代码。
3. Policy Guard 校验：角色、租户、法人实体、金额、账户白名单、期间、工具 side-effect、预算、最大步数。
4. Workflow 执行每一步；Agent 只在节点边界得到脱敏结果，可被暂停/替换/人工接管。
5. Verifier 对结果独立重算（金额、借贷、重复、状态、外部回执）；不通过则不推进。
6. 记录模型/提示/检索文档/工具调用/输入输出 hash/策略结果/人工决定。

## 4. 阻止误操作的机制

- **工具白名单：** 域名、HTTP 方法、ERP operation、文件路径和窗口标题都显式列举；默认 deny。
- **参数 schema：** 金额、币种、账户、期间、实体必须是枚举/引用，不接受模型自由文本。
- **风险门：** 高风险（付款释放、供应商主数据、税务申报、总账过账）强制 Human Task + 四眼原则；中风险需 reviewer；低风险可按阈值直通。
- **动作预览：** 在 UI RPA 执行前显示业务摘要、目标、金额、账户和证据；人工确认后签发短期 write token。
- **沙箱与预算：** 最大步数、token、调用次数、运行时长、单次金额和每日上限；熔断后只读。
- **提示注入防护：** 邮件/PDF/网页文本被视为不可信数据；指令与内容分离；模型不能修改 system policy 或工具目录。
- **回滚：** 外部系统支持撤销时采用反向 Activity；不支持撤销就将动作设为不可逆并强制审批，不承诺数据库回滚。

## 5. 财务禁用 Agent 自主执行的操作

默认禁止：银行付款提交/释放；收款账户和供应商主数据变更；总账最终过账/期间打开关闭；税务申报提交；信用额度/坏账核销最终决定；工资/个人敏感数据批量导出；删除原始凭证或审计日志；改变 SoD、额度、白名单和模型/策略配置。

可自动化但需规则校验：低金额、可逆、白名单实体的草稿创建；只读查询；对账候选；发票字段抽取；异常分类/解释；报表摘要。每个客户应根据内控和监管建立 allowlist，而不是照搬此清单。

## 6. 评估与发布

| 维度 | 指标 | 发布门槛示例 |
|---|---|---:|
| 工具选择 | 正确工具率、越权拒绝率 | 选择 ≥99%；越权 100% 拒绝 |
| 参数 | schema 通过率、金额/实体错误率 | schema ≥99.9%；关键金额 0 错误 |
| 文档 | 字段准确率、证据坐标覆盖 | 按字段设阈值，低置信度不直通 |
| 结果 | 业务端到端正确完成率 | ≥99.5%，高风险更高 |
| 安全 | 提示注入成功率、敏感泄露 | 红队用例全部阻断 |
| 成本 | 每单 tokens/延迟/云费 | 不超过场景预算 |

采用 golden set、shadow、canary、自动回归和人工抽样；保存 model/provider/version、prompt hash、检索版本。NIST AI RMF 可作风险管理框架参考，[NIST](https://www.nist.gov/itl/ai-risk-management-framework)。

## 7. “Agent + RPA”而不是“Agent 替代 RPA”

Computer Use 适合临时、低频、无 API、页面结构变化的辅助；固定高量财务流程仍用 Playwright/C# UIA 或 API，因为它们可测试、可回放和可解释。Anthropic 的 Computer Use 文档明确要求金融交易等高后果动作由人工确认，并提示 prompt injection 风险，[官方文档](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool)。

