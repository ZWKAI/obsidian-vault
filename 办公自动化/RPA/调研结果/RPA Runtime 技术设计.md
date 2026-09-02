# Deliverable 4｜RPA Runtime 技术设计

## 1. Runtime 组成

```text
Signed Job
  ↓ mTLS/WebSocket outbound
Runtime Supervisor（升级、心跳、租约、隔离）
  ├─ Browser Executor（Playwright）
  ├─ Windows Executor（C#/.NET UIA）
  ├─ Excel/File Executor（Open XML/Office bridge）
  ├─ API/DB/SFTP Executor
  ├─ Evidence Collector（事件/截图/下载 hash）
  └─ Secret Broker（短期 lease，不落盘）
```

Runtime 不解析完整业务 DSL；Control Plane 编译的 execution plan 只包含允许的 Activity、参数引用和策略。执行器永远采用 outbound connection，不开放公网入站管理端口。

## 2. Activity / Plugin 系统

### 2.1 Activity 描述文件

```json
{
  "id": "erp.invoice.post",
  "version": "4.2.0",
  "display_name": "Post AP invoice",
  "runtime": "linux-amd64",
  "capabilities": ["write", "finance.posting"],
  "input_schema_ref": "finance.InvoicePostInput",
  "output_schema_ref": "finance.InvoicePostOutput",
  "config_schema": {"type": "object", "properties": {"endpoint": {"type": "string"}}},
  "secret_scopes": ["erp.ap.writer"],
  "data_classification": "confidential",
  "idempotency": {"required": true, "key_field": "invoice_hash"},
  "retryable_errors": ["HTTP_429", "HTTP_502", "ERP_TIMEOUT"],
  "evidence": ["request_hash", "response_receipt", "business_document_no"],
  "healthcheck": {"type": "runtime_probe"},
  "entrypoint": "oci://signed-registry/erp-invoice-post@sha256:<digest>"
}
```

### 2.2 SDK 生命周期

```text
register(manifest + image/signature)
  → validate(schema/capability/license)
  → publish(version)
  → discover(catalog filtered by tenant/runtime/risk)
  → plan(activity_ref + input mapping)
  → execute(context, input, secret lease, cancellation token)
  → result(output, receipt, evidence refs) | typed error
  → deprecate/rollback (existing runs remain pinned)
```

### 2.3 TypeScript SDK 轮廓

```ts
export type ActivityContext = {
  executionId: string; nodeId: string; tenantId: string;
  logger: RedactingLogger; evidence: EvidenceWriter;
  secrets: SecretLease; signal: AbortSignal;
};

export interface Activity<I, O> {
  manifest(): Manifest;
  validate(input: unknown): I; // schema validation, no side effect
  run(ctx: ActivityContext, input: I): Promise<ActivityResult<O>>;
  compensate?(ctx: ActivityContext, input: I, output: O): Promise<void>;
  healthcheck?(): Promise<HealthStatus>;
}
```

`ActivityResult` 必须包括 `status`、typed output、external receipt、retry hint、evidence refs 和可观测 attributes；异常必须是 `TransientError | BusinessError | SecurityError | UnknownSideEffectError`，不能只抛字符串。

### 2.4 Activity 分类

```text
Browser: open / locate / click / fill / select / download / upload / screenshot
Desktop: window / UIA locate / click / type / hotkey / wait / screenshot
Excel: read / write / formula / filter / sort / pivot / export
File: list / move / hash / archive / decrypt（策略控制）
Email: list / attachment / send（收件人白名单）
HTTP: request / poll / webhook verify
Database: read-only query / stored procedure（禁任意写）
Document: classify / OCR / extract / evidence crop
ERP: read invoice/PO / post draft / reverse
AI: classify / suggest / explain / tool-call（默认只读）
Human: review / approve / reject / request-info / escalate
```

### 2.5 第三方 Activity 的安全边界

- OCI/WASM 包签名、SBOM、漏洞扫描、hash pin；插件没有宿主机管理员权限。
- capability 细分 `read`、`write`、`approve`、`network:domain`、`filesystem:path`、`secret:scope`。
- 沙箱默认 deny；每次网络/文件/secret 访问经 broker 授权并写审计。
- 版本不可变；新版本需 contract test、黄金数据、故障注入、canary；撤回不影响已运行实例。

## 3. Browser Automation：最终推荐 Playwright

| 维度 | Playwright | Selenium | Puppeteer |
|---|---|---|---|
| 稳定性 | 自动等待、locator、browser context 好 | WebDriver 生态成熟但显式等待较多 | Chromium 场景稳定 |
| 浏览器 | Chromium/Firefox/WebKit | Chrome/Firefox/Edge/Safari 等 WebDriver | 主要 Chromium |
| iframe/Shadow DOM | 原生能力较顺 | 可做但代码更繁琐 | 可做 |
| 下载/上传/网络 | API 丰富，trace 好 | 生态广，需组装 | Chromium 友好 |
| 多会话 | context 隔离 | profile/driver 隔离 | browser context |
| CAPTCHA | 不应绕过；交给人工/官方 API | 同 | 同 |
| 选型 | **首选**新平台 | 已有 WebDriver 资产/跨语言时 | 只做 Chromium worker |

Playwright locator 的 auto-wait/retry 可减少竞态，但不保证业务成功；每个关键写操作仍需从独立 API/UI 查询回读。[官方文档](https://playwright.dev/docs/locators)。

Browser Worker 规范：域名白名单、独立 context、下载临时目录、文件类型/大小限制、网络记录脱敏、每次写动作前后截图/DOM 摘要、禁止任意 JS 注入生产数据、验证码进入 Human Task。

## 4. Windows Runtime：C# UIA 为生产主线

### 4.1 技术比较

| 技术 | Win32/WPF/WinForms | Electron/Java Swing/老 ERP | 稳定性 | 建议 |
|---|---|---|---|---|
| Windows UI Automation（C#） | 控件树、AutomationPattern | 对实现质量依赖 | 高 | **生产主线**；微软 UI Automation 以控件树为测试/自动化模型，[文档](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-usefortesting) |
| WinAppDriver | WebDriver 风格 | 兼容性受限、维护风险 | 中 | 存量测试/过渡，不作为新核心 |
| pywinauto | Python 快速 | 依赖 backend，复杂桌面边界多 | 中 | 原型、数据采集辅助 |
| AutoHotkey/键鼠 | 几乎所有窗口但无语义 | 坐标/焦点/分辨率脆弱 | 低 | Attended 小工具/最后兜底 |
| 图像/CV | 自绘控件 | 误识别和 DPI 问题 | 低–中 | 只作候选定位，必须回读验证 |

### 4.2 Runtime 方案

```text
Windows Service（LocalSystem 最小权限，仅监督）
  ├─ Per-job Windows user session（专用服务账户）
  ├─ Signed .NET Executor（UIA/Office bridge）
  ├─ Secure desktop/session broker（不把 RDP 密码给流程）
  ├─ Local encrypted scratch + cleanup
  └─ Outbound mTLS agent ↔ Control Plane
```

每个 job 使用专用 Windows user、临时 profile 和窗口白名单；锁屏/VDI 兼容性在 PoC 验证。桌面 agent 只接受签名 execution plan，不接受任意 PowerShell/CMD。运行前做应用版本/DPI/语言/窗口健康检查；运行后清理剪贴板、下载和临时文件。

### 4.3 为什么绕不开 Windows Agent

企业仍有 SAP GUI、财税客户端、银行安全控件、Office 插件、OA/ERP 自绘控件和只有桌面入口的旧系统。API 改造排期长且常涉及供应商；RPA 的商业价值正是覆盖最后一公里。因此即使云端 Agent/Workflow 完整，执行仍需要一个能够访问内网和交互式桌面的 Windows 节点。

## 5. Excel Automation 最佳方案

| 任务 | 首选 | 原因/边界 |
|---|---|---|
| 读取/写入普通 XLSX | Python `openpyxl` 或 Java/Go OpenXML 库 | 无 Office 进程、易容器化；复杂宏/外部连接有限 |
| 大数据清洗/聚合 | Python `pandas`/Polars | 快、可测试；写回需保留格式时分层 |
| 公式、Pivot、宏、插件 | 隔离的 Windows Excel COM worker | 最高兼容性；单 job 独占进程，超时强杀并恢复副本 |
| Microsoft 365 云文件 | Microsoft Graph/SharePoint API | API 优先；权限/版本/锁需处理 |
| LibreOffice headless | Linux 低成本转换/公式子集 | 与 Excel 兼容性需逐模板基准，不作为财务真相源 |

生产流水线：`download → hash/virus scan → schema check → openpyxl/pandas transform → formula/format validation → optional isolated COM recalc → save temp → reopen/check totals → atomic move → evidence`。

必须记录 workbook hash、模板版本、sheet/row count、公式错误、总额/借贷校验；禁止直接覆盖原件。大文件分块读写，设置内存阈值；COM 进程异常由 supervisor 回收，失败后从输入副本重跑而非继续半写文件。

## 6. OCR / Document AI 分工

```text
PDF/image → malware/size check → classifier → OCR/layout → field extraction
         → field confidence + coordinate evidence → deterministic validation
         → business rules → human review → ERP/API
```

| 工作 | OCR/专用 Document AI | LLM/VLM |
|---|---|---|
| 文字、数字、表格坐标 | **首选** | 可辅助但不稳定 |
| 发票字段（金额、税额、号码） | **首选抽取** | 长尾/复杂版式候选 |
| 银行回单/对账单表格 | OCR + layout | 解释摘要、字段语义补齐 |
| 合同条款/政策问答 | OCR 后分段 + RAG | **理解、引用、摘要** |
| 报销合规理由 | 票据抽取 + 规则 | **解释/候选风险** |
| 是否真实、是否应付款 | 不能判断 | 不能单独决定；交给规则/人工 |

OCR 输出字段值、置信度、坐标和原图 hash；LLM/VLM 输出候选 JSON、证据引用和不确定性。模型不得直接改写原始文档或决定税额/付款。
