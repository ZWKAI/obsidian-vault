# Deliverable 1｜财务 RPA 竞品逆向分析

**研究截止：2026-09-02｜方法：以官方文档/产品页为主，营销材料只作为能力线索；没有统一实测的数据不做“谁更强”的绝对判断。**

## 1. 竞品共同的真实产品结构

```text
RPA Platform
├── Experience
│   ├── Home / KPI / Dashboard
│   ├── Studio / Process Designer
│   ├── Process & Version Library
│   ├── Task / Human Work Queue
│   └── Operations / Reports
├── Control Plane / Orchestrator
│   ├── Job / Queue / Schedule / Trigger
│   ├── Worker / Machine / Runtime / Capacity
│   ├── Package / Environment / Release
│   └── Retry / Resume / SLA / Dependency
├── Runtime
│   ├── Browser / Desktop / Excel / File / Email
│   ├── API / Database / SFTP / ERP Connector
│   └── Credential / Session / Screenshot / Artifact
├── Intelligence
│   ├── OCR / Document Understanding
│   ├── Process / Task Mining
│   ├── LLM / Agent / Recommendation
│   └── Test / Evaluation / Healing
└── Governance
    ├── Tenant / Folder / Environment
    ├── User / Group / Role / Policy / SoD
    ├── Secret / Key / Data Retention
    ├── Audit / Log / Alert / SIEM
    └── ALM / Approval / Signed Package / Rollback
```

**逆向结论：** 设计器只是获客入口；平台真正昂贵的是运行身份、Windows 会话、队列恢复、版本治理、连接器矩阵和审计证据。

## 2. 产品逐家拆解

### 2.1 UiPath

**事实（官方）：** Orchestrator 将 Job 作为 Process 在 Robot 上的执行，支持队列触发、停止/终止/恢复/重启；Robot 对象关联机器、权限和凭证；Folder 连接流程、账户、机器和容量。[Jobs](https://docs.uipath.com/orchestrator/automation-cloud/latest/user-guide/about-jobs)、[Robots](https://docs.uipath.com/orchestrator/automation-cloud/latest/user-guide/about-robots)、[Orchestrator](https://www.uipath.com/product/orchestrator)。

```text
首页/KPI
  ├─ Automation Cloud / Organization / Tenant / Folder
  ├─ Studio / Studio Web / Apps
  ├─ Process / Package / Release
  ├─ Jobs / Queues / Triggers / Assets
  ├─ Robots / Machines / Runtime
  ├─ Logs / Alerts / Audit / Insights
  ├─ Credentials / CyberArk 等外部存储
  └─ Document Understanding / Process Mining / Task Mining / Maestro
```

- **路线：** 从传统 Studio + Orchestrator 发展为全栈自动化；Maestro 用 BPMN/DMN 连接 agents、robots 和 people，[产品页](https://www.uipath.com/product/maestro)。Document Understanding 采用分类、数字化、字段抽取、验证的人机闭环，[官方基础能力](https://docs.uipath.com/document-understanding/automation-cloud/latest/user-guide/fundamental-capabilities)。
- **控制台强项：** 多租户/Folder、队列、机器池、运行身份、包版本、日志和资产治理。
- **Runtime：** 浏览器、Windows、Office、API、SAP/ERP 生态成熟；后台与前台执行区分，前台 unattended 需要可登录的 Windows 身份。
- **AI：** 文档理解、流程/任务挖掘、Maestro、Agent/Autopilot/Healing 等组合；官网能力很强，但每项功能的许可、区域和版本需要逐项核验。
- **弱点（推断）：** 产品面积和许可复杂；实施伙伴依赖高；财务语义、匹配规则和内控模板仍需客户/伙伴建设。

### 2.2 Microsoft Power Automate

```text
Power Platform
├─ Cloud Flows / Approvals / Connectors
├─ Power Automate for desktop（Desktop flow）
├─ Monitor: Runs / Machines / Machine groups / Queues
├─ Dataverse / AI Builder / Copilot
├─ Environments / Solutions / ALM / DLP Policies
└─ Entra ID / Intune / Windows 365 Hosted Machines
```

**事实（官方）：** Hosted Machines 允许在门户创建、测试和运行 attended/unattended desktop flows；Hosted Machine Group 按队列和容量自动配置机器并负载均衡。[Hosted machines](https://learn.microsoft.com/en-us/power-automate/desktop-flows/hosted-machines)、[Hosted machine groups](https://learn.microsoft.com/en-us/power-automate/desktop-flows/hosted-machine-groups)。

- **路线：** 云流/DPA 是上层编排，Desktop Flow 是 Windows 执行；连接器、Dataverse、AI Builder、Power BI、Copilot 与 Microsoft 365/Dynamics/Azure 形成粘性。
- **控制台强项：** Environment、Solution/ALM、DLP、连接引用、运行历史、机器/机器组。
- **Runtime：** Web/desktop/Excel/Office/Windows 生态好；非微软遗留系统仍需 UI 自动化和自建治理。
- **商业：** Premium 15 美元/用户/月、Process 150 美元/bot/月、Hosted Process 215 美元/bot/月（地区税费、先决条件和合同另计），[官方定价](https://www.microsoft.com/en-us/power-platform/products/power-automate/pricing)。
- **弱点（推断）：** 许可和环境边界复杂；财务领域对象/审计包不是原生核心；跨云/本地和非微软 ERP 需 connector/实施。

### 2.3 Automation Anywhere

```text
Automation Anywhere
├─ Control Room / Workspaces / Bots / Schedules / Queues
├─ Bot Creator / Bot Runner
├─ API/Packages / Credential Vault
├─ Document Automation / IQ Bot lineage
├─ AI Agent Studio / Process Reasoning
└─ Cloud or Enterprise on-prem deployment
```

官方定位已将 Agentic Process Automation System 描述为 AI agents、RPA、文档处理和编排的组合，[产品页](https://www.automationanywhere.com/products/agentic-process-automation-system)；AI Agent Studio 提供 agent 创建与工具连接，[产品页](https://www.automationanywhere.com/products/ai-agent-studio)。

- **强项：** 云优先、Bot/Agent/Document 的产品叙事统一；企业级 Control Room、队列和凭证；纯云与本地部署路径都存在。
- **弱点（推断）：** 公开材料难以比较实际端到端稳定性；高级 AI、容量、实施和迁移成本需合同确认。

### 2.4 SS&C Blue Prism

- **产品心智：** Digital Workforce、Control Room、Process Studio、Object Studio、Work Queues、Credentials、Release/Environment。
- **强项：** 后台数字员工、集中治理、金融服务客户基础；产品文档入口为[官方文档](https://documentation.blueprism.com/)，企业产品页为[Blue Prism Enterprise](https://www.blueprism.com/products/enterprise/)。
- **弱点（推断）：** 传统建模和发布流程对新开发者不如低代码产品顺滑；AI/Document/Agent 常依赖附加产品或集成。

### 2.5 SAP Build Process Automation

```text
SAP BTP
├─ Process Builder / Decisions / Forms / Approvals
├─ Actions（S/4HANA 与非 SAP API）
├─ RPA Bots / Desktop Agent
├─ Monitoring / Transport / Identity
└─ Joule/AI 及 SAP 业务语义
```

**事实（官方）：** Actions 可用于 SAP S/4HANA 及非 SAP 系统，平台提供 API；定价采用 BTP capacity/合同模型，[Actions](https://help.sap.com/docs/build-process-automation/sap-build-process-automation/set-up-and-use-actions-with-sap-build-process-automation)、[API](https://help.sap.com/docs/build-process-automation/sap-build-process-automation/using-sap-build-process-automation-apis?locale=en-US)、[定价](https://www.sap.com/uk/products/technology-platform/process-automation/pricing.html)。

- **强项：** SAP 业务对象、身份、传输和流程治理；适合已有 BTP/S/4HANA 投资的集团。
- **弱点（推断）：** 非 SAP 系统和中国本地银行/税务适配不一定占优；需要 SAP 许可和实施能力。

### 2.6 ServiceNow

- **产品心智：** Automation Engine/RPA Hub、IntegrationHub、Flow Designer、Automation Center、Task Mining、AI Agents。
- **强项：** 工单、IT/员工服务、审批和运营治理；Automation Center 可聚合自动化/流程挖掘数据，[集成文档](https://www.servicenow.com/docs/r/integrate-applications/automation-center/automation-center-integrations.html)。
- **边界：** 它不是财务总账/应付系统；财务自动化需依赖 ERP、RPA 和连接器。

### 2.7 国内主要 RPA

| 产品 | 公开定位/证据 | 适合场景 | 需要核验 |
|---|---|---|---|
| 来也 Laiye | RPA + IDP + RAG + MCP/Agent，[产品页](https://laiye.com/product) | 中文文档、共享中心、本地部署 | 具体 connector、版本和许可 |
| 金智维 | 企业级 RPA、Ki-Agent 私有化/监督式 Agent，[Ki-Agent](https://www.kingsware.cn/ki-agent) | 金融/政企、国产化、私有部署 | 运行时跨版本稳定性与 AI 评估 |
| 艺赛旗 | RPA 2024.1、微服务、流程/任务挖掘，[产品说明](https://support.i-search.com.cn/article/1733733045144) | 共享中心、国内 ERP/税务 | 多租户/开放 SDK 深度 |
| 影刀 ShadowBot | Windows/Web RPA 与行业案例，[产品页](https://www.yingdao.com/product/) | 电商、运营、批处理、中文桌面 | 财务审计/SoD 的平台化程度 |
| 弘玑/实在/九科/容智/云扩等 | 企业 RPA、IDP、Agent 或国产化方案 | 政企、制造、共享中心 | 公开技术细节/统一基准有限 |

**国内共同优势（推断）：** 本地交付、中文 UI、国产操作系统/私有化、税务/银行/本土 ERP 适配和采购关系。**共同弱点（基于公开资料可见性）：** 版本/性能/故障率/真实 TCO 的可比证据少；产品边界与实施服务容易混合。

## 3. 功能差距与逆向机会

| 需求 | 通用厂商普遍较强 | 财务团队仍常自行补齐 | 我们的切入 |
|---|---|---|---|
| Bot/Job/Queue/机器 | 是 | 财务业务语义、异常 reason code | Finance-native domain model |
| UI 自动化 | 是 | 业务成功回读、反冲、证据 | 结果验证和财务证据包 |
| OCR/LLM | 是/组合产品 | 字段置信度与会计规则闭环 | 抽取→校验→复核→过账 |
| 审批/RBAC | 是基础能力 | SoD、金额/账户/法人实体策略 | Policy-as-code + 四眼门禁 |
| 流程挖掘 | 产品化程度不一 | 事件标准化与 ROI 归因 | 先做场景指标，不首版做 mining |
| Connector | 数量多 | 本地银行、税务、国产 ERP 版本 | 认证适配器与 contract test |

### CTO 判断

不要与 UiPath 争“活动数量”，不要与 Power Automate 争“连接器数量”，不要与国内厂商争“项目人力”。应争：一个财务场景从输入到结果的**端到端正确率、审计重建时间、异常闭环速度、每千单 TCO 和模板复用率**。

## 4. 关键来源与限制

- 产品能力以官方资料为证据：[UiPath Jobs/Robots](https://docs.uipath.com/orchestrator/automation-cloud/latest/user-guide/about-jobs)、[Power Automate Hosted Machines](https://learn.microsoft.com/en-us/power-automate/desktop-flows/hosted-machines)、[AA APA](https://www.automationanywhere.com/products/agentic-process-automation-system)、[SAP BPA](https://help.sap.com/docs/build-process-automation/sap-build-process-automation/using-sap-build-process-automation-apis?locale=en-US)。
- 没有公开统一 hands-on benchmark；功能勾选不是采购结论。必须让候选产品跑同一批脱敏发票、流水、ERP 版本和故障注入。
