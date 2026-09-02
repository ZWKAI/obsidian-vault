# Deliverable 2｜Finance Automation 产品功能架构

## 1. 产品定义

### 1.1 是否合理

“Workflow + RPA + AI Agent + 财务系统”方向合理，但必须加上三个横切面：**Finance Domain、Policy/Security、Evidence/Operations**。没有这三层，产品只是一个会调用工具的自动化平台，无法通过财务内控或兑现 ROI。

```text
Finance Automation Platform
├── Finance Domain（核心壁垒）
│   ├── Invoice / PO / Receipt / Payment
│   ├── Bank Transaction / Reconciliation
│   ├── Journal / Period / Account / Entity
│   ├── Tolerance / Match / Exception Reason
│   └── Evidence Pack / Close Calendar / SLA
├── Workflow（确定性过程）
│   ├── Durable Execution / State / Compensation
│   ├── Rule / Decision / Loop / Parallel
│   ├── Human Task / Approval / Escalation
│   └── Trigger / Schedule / Version / Release
├── RPA（无 API 的执行器）
│   ├── Browser / Windows / Excel / File / Email
│   └── Session / Screenshot / Download / Result Verification
├── Intelligence（受约束的不确定性）
│   ├── OCR / Classification / Extraction
│   ├── RAG / Policy Q&A / Anomaly Explanation
│   └── Agent Planner / Tool Calling / Evaluation
├── Connectors（插件生态）
│   ├── ERP / Bank / Tax / OA / Expense / CRM
│   ├── REST / SOAP / SFTP / DB / Email
│   └── Custom SDK / Secrets / Rate Limit
└── Trust Plane（不可削弱）
    ├── IAM / SSO / MFA / RBAC / ABAC / SoD
    ├── Secret / KMS / DLP / Data Residency
    ├── Audit / Evidence / Retention / SIEM
    └── SLO / Alert / Cost / Change / Rollback
```

## 2. 三者分工

| 组件 | 做什么 | 不做什么 | 典型财务例子 |
|---|---|---|---|
| Workflow | 状态、顺序、等待、审批、SLA、重试/补偿、版本 | 不理解任意自然语言，不代替 ERP | 发票→匹配→异常→审批→凭证 |
| RPA | 受控地操作稳定网页/Windows/Excel UI | 不持有业务规则真相，不决定付款 | 无 API 网银下载、老 ERP GUI 录入 |
| AI Agent | 分类、抽取、候选、解释、选择允许的只读工具 | 不绕过策略、不自由写 SQL/付款/过账 | 识别汇款附言并给出核销候选 |
| Finance Domain | 会计对象、容差、业务键、内控、证据 | 不承担 UI 驱动 | 3 分钟容差、借贷平衡、重复发票 |

## 3. 功能树与优先级

### P0（3 个月 MVP 必须）

```text
P0
├── Home
│   ├── Today: pending / failed / approval / SLA
│   └── Value: processed / touchless / minutes saved（可配置）
├── Finance Workbench
│   ├── Batch / Document / Reconciliation / Journal
│   ├── Exception Inbox（reason code、证据、重试/转人工）
│   └── Approval Inbox
├── Workflow
│   ├── Template + JSON/受限画布
│   ├── Trigger / Variable / Condition / Loop / Parallel
│   ├── Retry / Timeout / Compensation / Human Approval
│   └── Draft → Review → Publish（不可变版本）
├── Execution
│   ├── Job / Queue / Worker / Lease
│   ├── API / Browser / Excel / File / Email Activity
│   └── Run / Resume / Cancel / Dry-run
├── Control
│   ├── Tenant / Legal Entity / Environment
│   ├── User / Role / Permission / SoD
│   ├── Credential / Connection（仅引用 secret）
│   └── Audit / Evidence / Redaction
└── Operations
    ├── Run/Node logs、截图、外部响应摘要
    ├── Health / SLO / Alert / Version / Rollback
    └── API docs / Import / Export
```

### P1（上线后 3–9 个月）

OCR/Document AI、字段复核台、规则管理 UI、流程/规则版本 diff、C# Windows UIA agent、流程模板市场、连接器认证、ROI/chargeback、Process/Task Mining 的轻量事件采集、SIEM/审计导出、多活/灾备。

### P2（谨慎投资）

自然语言生成流程草稿、Agent 异常调查、Computer Use、selector healing、跨流程自主规划、自动流程优化、通用 mining 算法。P2 都不能绕过 P0 的策略和证据。

## 4. 插件/Connector 原则

- **平台核心：** 统一 schema、执行语义、版本、权限、证据、状态、重试和运营指标。
- **插件：** 所有外部系统协议、页面定位、字段映射、限流和特定业务 API。
- 插件必须声明 `capabilities`（read/write/approve）、输入输出 JSON Schema、数据分类、所需 secret scope、幂等支持、最大并发、版本和回滚策略。
- Write connector 不得被 Agent 直接暴露为通用工具；必须被 Workflow 节点包裹，按实体/金额/期间授权。

## 5. 第一阶段明确不做

通用 RPA Studio、所有 ERP/银行兼容、任意 PDF/发票识别、付款自动释放、自动纳税申报、开放式 Computer Use、模型训练平台、独立 BPMN 全套、流程挖掘算法、企业级低代码市场和跨国会计准则。

## 6. 验证指标（产品层）

| 指标 | 定义 | MVP 建议目标 |
|---|---|---:|
| 端到端正确完成率 | 业务结果正确且证据完整/总单数 | ≥99.5%（关键写入更高） |
| 直通率 | 无人工接管且经独立校验成功/总单数 | ≥70% 起步 |
| 例外分类覆盖 | 有标准 reason code 的例外/全部例外 | 100% |
| 变更恢复时间 | UI/API 变化到恢复生产的时间 | ≤1 工作日（试点） |
| 审计重建时间 | 从 execution 找齐输入、规则、审批、结果 | ≤5 分钟/单 |
| 每千单 TCO | 许可+基础设施+运营+人工例外 | 逐月下降 |
| 模板复用率 | 无核心代码改动部署第二实体/客户的流程 | ≥60% 起步 |

## 7. 四个核心场景的端到端蓝图

### 场景 A：供应商发票

```text
Mailbox API/SFTP → file hash/virus scan → OCR/classifier
→ invoice schema + field evidence → supplier master API read
→ PO/GR API read → deterministic 3-way match
→ [matched & low risk] draft/post API
→ [exception] AI explanation(read-only) → human approval
→ ERP post/receipt → evidence pack → supplier notification
```

| 步骤 | 首选技术 | RPA/AI/人工边界 |
|---|---|---|
| 附件获取 | Mail API/SFTP | 邮箱 UI 仅无 API 时 RPA |
| 抽取 | OCR/Document AI | VLM 仅长尾；字段坐标和置信度必存 |
| 匹配 | ERP API + 规则引擎 | AI 只能给候选，不能改 PO/金额 |
| 例外 | Workflow + reason code | Agent 解释/检索，人工决定 |
| 过账 | ERP API；无 API 用受限 RPA | 需要 idempotency、回读、审批 |

### 场景 B：银行流水自动对账

```text
Bank API/SFTP/UI → normalize BankTxn
→ fetch ERP open items → candidate generation
→ score/match/duplicate check → auto-clear low-risk
→ exception inbox → human resolve → ERP clearing
→ daily reconciliation evidence and aging metrics
```

候选匹配建议：`score = 0.35×amount + 0.15×date + 0.20×account/name + 0.20×reference(invoice/order) + 0.10×memo_semantic`。金额/币种不一致直接拒绝；账号精确命中高于户名；订单号/发票号 hash 精确命中最高；摘要语义仅用于候选而非直接核销。使用 blocking（实体、币种、金额桶、日期窗）减少全表笛卡尔积；一对多/多对一通过总额和唯一性验证；低于阈值进入人工。权重和 reason code 必须版本化，并在历史黄金集上评估 precision/recall。

### 场景 C：Excel 财务报表

```text
API/SFTP/ERP export → hash + schema check → read XLSX/CSV
→ pandas/Polars clean → domain calculation
→ write template workbook → formula/total/period validation
→ optional isolated Excel COM recalc → reopen/check
→ atomic publish → email/API notification → evidence
```

模板版本、Sheet 名/列类型、公式错误、借贷/总额和截止时间都是发布门禁。原件只读；输出写临时文件并原子替换。Excel COM 卡死时，supervisor 杀掉孤儿进程，从输入副本恢复；禁止在半写文件上续跑。

### 场景 D：报销自动审核

```text
Expense API/OA → receipt files → OCR/分类/字段证据
→ employee/department/policy lookup → deterministic checks
→ AI anomaly suggestion（重复/异常商户/解释）
→ low-risk auto-approve or human approval
→ expense/GL API → reimbursement status notification
```

规则先行：金额上限、日期/出差区间、成本中心、重复票据 hash、税额和发票验真；AI 只能给异常原因和引用证据。涉及个人信息时按字段最小化、脱敏和组织权限裁剪；审批人与报销人/记账人 SoD 冲突必须拒绝。

## 8. 领域服务边界

```text
document-service      OCR/classification/extraction/evidence
finance-match-service amount/date/entity/reference matching
policy-service        rules/version/SoD/limits/decision explanation
connector-service     ERP/bank/mail/tax protocol adapters
workflow-service      durable state/human/task/compensation
execution-service     worker lease/activity result/observability
```

领域服务返回确定性 typed result；Agent 不直接访问数据库或连接器 secret。所有“自动核销/过账”都必须回到 policy-service 与 workflow-service 的门禁。
