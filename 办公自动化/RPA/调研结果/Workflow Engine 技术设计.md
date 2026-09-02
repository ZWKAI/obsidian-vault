# Deliverable 3｜Workflow Engine 技术设计

## 1. 一个 RPA Workflow 到底是什么

生产定义：**一个可版本化、可暂停、可恢复、可审计的业务执行图**，不是“按顺序点击的脚本”。它包含：输入契约、节点图、策略、运行身份、外部副作用、人工状态、证据要求和补偿路径。

### 1.1 模型比较

| 模型 | 优点 | 缺点 | 财务适用性 |
|---|---|---|---|
| DAG | 简单、易校验、并行自然 | 长等待/人工/补偿弱 | 批量导入、报表清洗 |
| BPMN | 业务可读、审批/事件/边界错误丰富 | 引擎和模型复杂，代码扩展要谨慎 | 共享中心、跨部门审批 |
| State Machine | 状态和守卫明确，易审计 | 复杂分支/可视化需自建 | 对账批次、付款生命周期 |
| JSON/DSL | API/代码友好、可版本控制 | 需要自建设计器/验证器 | MVP 的规范化 DSL |
| XML | BPMN 互操作 | 冗长、迁移和 diff 不友好 | 兼容 BPMN，不宜作为唯一内部模型 |
| 自研 Durable Engine | 可完全定制 | 恢复、事件历史、并发、定时器是多年工程 | No-Go，除非已有平台团队 |

**推荐：** 对外可提供 BPMN/受限画布；内部采用**版本化 JSON DSL → 静态校验 → 编译为 Temporal/Camunda 执行计划**。业务状态以显式 State/Guard 表达，节点以 Activity/Tool 绑定，避免在 UI JSON 内嵌任意代码。

## 2. 生产级最小 DSL

```json
{
  "schema": "finance.workflow/v1",
  "workflow": {
    "id": "ap_invoice_control",
    "version": 7,
    "status": "published",
    "tenant_id": "t_acme",
    "owner": "finance-shared-service",
    "description": "Capture, validate and approve AP invoice before posting",
    "triggers": [
      {"type": "event", "source": "mailbox.ap", "event": "attachment.received"},
      {"type": "schedule", "cron": "0 */10 * * * *", "timezone": "Asia/Shanghai"}
    ],
    "input": {
      "schema": {"type": "object", "required": ["message_id"], "properties": {"message_id": {"type": "string"}}},
      "secret_refs": ["conn.mailbox.ap"]
    },
    "variables": [
      {"name": "invoice", "type": "object", "schema_ref": "finance.Invoice", "sensitivity": "confidential"},
      {"name": "match_result", "type": "object", "schema_ref": "finance.MatchResult"},
      {"name": "risk", "type": "string", "enum": ["low", "medium", "high"]}
    ],
    "policy": {
      "max_runtime": "PT2H",
      "data_residency": "cn",
      "require_evidence": ["input_hash", "rule_hits", "external_receipt"],
      "write_limits": {"currency": "CNY", "max_amount": 100000},
      "approval": {"high_risk": "finance.double_entry"}
    },
    "nodes": [
      {"id": "get_attachment", "type": "email.attachment.download", "version": "2.1.0", "input": {"message_id": "$.input.message_id"}, "retry": {"max_attempts": 3, "backoff": "exponential", "retry_on": ["transient"]}, "timeout": "PT2M", "evidence": "metadata_only"},
      {"id": "classify", "type": "document.classify", "version": "1.4.0", "input": {"file_id": "$.nodes.get_attachment.output.file_id"}, "ai": {"model_ref": "doc-router-v3", "min_confidence": 0.92, "on_low_confidence": "review"}, "timeout": "PT1M"},
      {"id": "extract", "type": "document.extract", "version": "3.0.0", "input": {"file_id": "$.nodes.get_attachment.output.file_id", "schema_ref": "finance.Invoice"}, "ai": {"model_ref": "invoice-zh-v5", "min_confidence": 0.90, "require_evidence": true}, "timeout": "PT3M"},
      {"id": "validate", "type": "finance.invoice.validate", "version": "1.2.0", "input": {"invoice": "$.nodes.extract.output.document"}, "ruleset": "ap-cn-v12", "timeout": "PT30S"},
      {"id": "match", "type": "finance.three_way_match", "version": "1.1.0", "input": {"invoice": "$.nodes.extract.output.document", "po": "$.nodes.validate.output.po_ref"}, "retry": {"max_attempts": 2, "retry_on": ["transient"]}, "timeout": "PT2M"},
      {"id": "risk_agent", "type": "ai.exception.explain", "version": "1.0.0", "input": {"validation": "$.nodes.validate.output", "match": "$.nodes.match.output"}, "tools": ["erp.read.purchase_order", "policy.read.ap_rule"], "mode": "suggestion_only", "timeout": "PT2M"},
      {"id": "review", "type": "human.approval", "input": {"case": "$.nodes.risk_agent.output", "evidence": "$.context.evidence_pack"}, "assignee": {"role": "ap_reviewer"}, "sla": "PT8H", "on_timeout": "escalate"},
      {"id": "post", "type": "erp.invoice.post", "version": "4.2.0", "input": {"invoice": "$.nodes.extract.output.document", "approval": "$.nodes.review.output"}, "credential_ref": "conn.erp.ap.writer", "idempotency_key": "$.nodes.extract.output.document.invoice_hash", "risk_level": "high", "timeout": "PT2M", "compensation": {"type": "erp.invoice.reverse", "version": "1.0.0"}},
      {"id": "notify", "type": "email.send", "version": "1.2.0", "input": {"to": "$.context.requester", "template": "ap.posted", "values": {"document_no": "$.nodes.post.output.document_no"}}, "evidence": "metadata_only"}
    ],
    "edges": [
      {"from": "get_attachment", "to": "classify", "when": "success"},
      {"from": "classify", "to": "extract", "when": "output.document_type == 'invoice'"},
      {"from": "classify", "to": "review", "when": "output.confidence < 0.92", "reason_code": "LOW_CLASSIFICATION_CONFIDENCE"},
      {"from": "extract", "to": "validate", "when": "success"},
      {"from": "extract", "to": "review", "when": "output.confidence < 0.90", "reason_code": "LOW_FIELD_CONFIDENCE"},
      {"from": "validate", "to": "match", "when": "output.is_valid == true"},
      {"from": "validate", "to": "review", "when": "output.is_valid == false", "reason_code": "VALIDATION_EXCEPTION"},
      {"from": "match", "to": "post", "when": "output.status == 'matched' && output.risk != 'high'"},
      {"from": "match", "to": "risk_agent", "when": "output.status != 'matched' || output.risk == 'high'"},
      {"from": "risk_agent", "to": "review", "when": "success"},
      {"from": "review", "to": "post", "when": "output.decision == 'approved'"},
      {"from": "review", "to": "notify", "when": "output.decision == 'rejected'", "reason_code": "HUMAN_REJECTED"},
      {"from": "post", "to": "notify", "when": "success"}
    ],
    "output": {"schema": {"type": "object", "properties": {"status": {"type": "string"}, "document_no": {"type": "string"}}}, "mapping": {"status": "$.nodes.post.output.status", "document_no": "$.nodes.post.output.document_no"}},
    "on_error": {"default": "create_exception", "notify": "finance_ops", "dead_letter_after": 3}
  }
}
```

### 2.1 字段契约

| 字段 | 规则 |
|---|---|
| `schema` | DSL 版本；不兼容升级必须迁移器，不静默解释 |
| `workflow.id/version/status` | 发布版本不可变；运行实例 pin 版本；draft/review/published/retired |
| `trigger` | event/schedule/manual/webhook；需去重键与时区 |
| `input/output` | JSON Schema；禁止隐式类型转换；敏感字段只引用，不写日志 |
| `variables` | 类型、schema、敏感级别、作用域（workflow/node/item） |
| `node.type/version` | Activity 注册表中的不可变实现；允许 capability 约束 |
| `input` | JSONPath/表达式白名单；禁止任意脚本和 SQL |
| `edges.when` | 纯函数表达式；有超时/else；每条边 reason code |
| `retry/timeout` | 节点级；按异常类型决定是否重试；含 jitter、最大总时长 |
| `exception/compensation` | business/security/transient 分类；补偿不是回滚幻想，而是显式反向动作 |
| `credential_ref` | 只引用连接/secret，执行器按策略取临时凭证 |
| `human.approval` | assignee、SoD、金额/实体策略、SLA、升级、审计 |
| `ai/tools` | 模型、预算、schema、工具白名单、模式（suggestion/guarded execute） |
| `evidence` | 输入 hash、规则命中、外部收据、人工决定、屏幕证据策略 |

### 2.2 组合节点（Loop / Parallel / Sub Workflow）

DSL 允许以下三类结构节点；它们仍必须有超时、取消、失败策略和审计范围：

```json
{
  "id": "per_invoice",
  "type": "loop",
  "items": "$.nodes.get_attachment.output.invoice_files",
  "item_var": "invoice_file",
  "max_concurrency": 4,
  "max_items": 5000,
  "body": {"sub_workflow_ref": "ap.invoice.single@7"},
  "on_item_error": "continue_and_create_exception"
}
```

```json
{
  "id": "fetch_sources",
  "type": "parallel",
  "branches": ["bank_api", "erp_open_items", "fx_rate"],
  "join": "all",
  "on_branch_error": "fail_fast",
  "max_concurrency": 3
}
```

`sub_workflow_ref` 指向已发布的不可变版本；子流程的输入输出通过 schema 显式映射，不能共享隐式全局变量。Loop 的 item-level idempotency key 为 `parent_execution + item_hash`；Parallel 的 join 记录每个 branch 状态，部分成功不能静默当作全部成功。

## 3. Compiler / Parser

1. 解析 JSON，校验 schema、唯一 node id、无悬空 edge、无非法环（仅 loop 节点允许）、输入引用存在。
2. 做静态安全检查：写操作必须有 `credential_ref + idempotency_key + risk_level`；高风险必须有 approval；AI 节点不得拥有付款/过账工具。
3. 规范化表达式、展开 sub-workflow、生成 typed execution plan。
4. 编译为 Temporal Workflow（或 Camunda process definition）并把 Activity implementation/version 绑定为 immutable reference。
5. 生成测试图：happy path、每个条件分支、超时、重试耗尽、worker crash、审批拒绝、补偿失败。
6. 发布前由 reviewer 签名；运行时验证 `workflow_version + rule_version + connector_version + model_version`。

## 4. Runtime 状态与恢复

```text
Workflow Run
 ├─ context（tenant/entity/correlation/idempotency）
 ├─ node execution（pending/running/waiting/succeeded/failed/compensating/paused）
 ├─ event history（append-only，含 attempt）
 ├─ timer / signal / human task
 └─ evidence refs / secret lease refs
```

- **Worker 挂掉：** heartbeat 超时，任务 lease 失效；引擎从事件历史重放，未确认 Activity 以 at-least-once 重派；Activity 必须靠业务幂等键避免重复副作用。
- **Resume：** 从最后一个 committed event 继续，不从 UI 光标猜测；外部写操作先 query by idempotency key，再决定 skip/retry/compensate。
- **Retry：** transient 指数退避+jitter；business exception 转 human；security exception 立即熔断并告警。
- **Timeout：** workflow、node、activity、human task 分层；timeout 触发 cancel/compensation/exception policy。
- **并发：** Queue item lease + entity/account lock；同一银行账户/期间可配置串行，实体之间并行；禁止用数据库长锁覆盖网络调用。
- **长事务：** durable timer、signal、human task；不保持线程/HTTP 连接，不让一个 Worker 阻塞 8 小时。
- **断点续跑：** 节点状态与输出结构化保存；不可确定重放的 UI Activity 标记为 non-replayable，恢复时必须 query/人工确认。

## 5. 70% 崩溃的自动付款例子

1. `payment.prepare` 生成唯一 `payment_batch_id`，写入本地 outbox，状态 `prepared`。
2. Worker 在提交网银前崩溃；引擎将 Activity 标记为 `unknown`，不直接重试。
3. 新 Worker 领取后用只读 API/受控 UI 查询 `payment_batch_id`、金额、收款账户和银行状态。
4. 若银行无记录：依据策略重试提交；若 `pending/processing`：等待 webhook/轮询；若 `success`：记录外部回执并继续；若金额/账户不一致：熔断、建高风险工单。
5. 若提交成功后通知节点失败，只重跑通知（通知幂等），不重跑付款。
6. 若需取消，走银行支持的撤销/冲正 Activity；“撤销”不等于数据库回滚，必须保留原付款与反向证据。
7. 全链路记录 operator、审批、policy/rule/workflow version、模型不参与付款决策的证明和外部回执。

## 6. Workflow 不应承担的职责

不要把会计规则、连接器细节、LLM prompt、UI selector、secret 值直接写进 DSL。DSL 描述**意图和约束**；Domain service/Activity 实现具体行为；Policy service 决定允许与否；Evidence service 负责留痕。

## 7. Dead Letter Queue、事务与补偿

- 重试耗尽、schema 不兼容、长期业务异常和安全拒绝进入 **Dead Letter Queue（DLQ）**，保留原始 execution/node 引用、错误分类、最后一次外部查询结果和建议处置；DLQ 只能由有权限的运营人员 re-drive/resolve，不能直接删除。
- 本地数据库采用短事务写状态和 outbox；外部 ERP/银行调用不包在数据库长事务内。Activity 先记录 `intent/prepared`，拿到外部 receipt 后再提交 `committed`；未知副作用进入 verifier。
- Compensation 是显式业务反向动作（如 invoice.reverse、payment.cancel），需要自己的权限、幂等键和证据；不可撤销动作必须在执行前标记并强制审批。
- **Distributed Lock：** 只锁业务键（如 `entity:bank_account:statement_date`）并设置 TTL/续租；锁服务故障时宁可暂停，不降级为无锁写入。
- **Checkpoint：** 每个可恢复边界提交结构化 checkpoint（输入 hash、已完成 Activity、外部 receipt、下一步）；UI 光标和截图不能作为唯一 checkpoint。
