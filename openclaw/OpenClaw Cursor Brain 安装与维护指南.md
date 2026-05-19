# OpenClaw Cursor Brain 安装与维护指南

> 将 [Cursor Agent CLI](https://cursor.sh) 作为 OpenClaw 的 AI 后端，通过 MCP 让 Cursor 与飞书、企业微信等插件工具互通。  
> 适用环境：macOS（Apple Silicon / Intel），OpenClaw ≥ 2026.4.x，Node.js ≥ 18。

---

## 一、前置条件

|   |   |
|---|---|
|项目|要求|
|OpenClaw|已全局安装：`npm i -g openclaw`|
|Cursor IDE|已安装并至少启动过一次|
|Cursor Agent CLI|终端可用 `agent` 或 `cursor-agent`（见第二节）|
|Node.js|≥ 18|
|Gateway|能正常启动（`openclaw gateway status` 为 running）|

---

## 二、安装 Cursor Agent CLI

插件依赖 **`agent` 命令**（不是 IDE 里的 `cursor` 壳命令）。任选一种方式：

### 方式 A：官方安装脚本（推荐）

```bash
curl -fsSL https://cursor.com/install | bash
```

安装后二进制位于：

- `~/.local/bin/agent`（主命令）
    
- `~/.local/bin/cursor-agent`（兼容别名）
    

### 方式 B：Cursor IDE 命令面板

1. 打开 Cursor
    
2. `Cmd+Shift+P` → 输入 **Install 'cursor' command** → 回车
    
3. 若仍无 `agent`，仍建议执行方式 A
    

### 配置 PATH

确保 `~/.local/bin` 在 PATH 中（zsh 示例）：

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

验证：

```bash
which agent
agent --version
```

> OpenClaw Gateway 的 LaunchAgent 通常已包含 `~/.local/bin`；若仅终端可用、服务找不到 CLI，检查 `~/Library/LaunchAgents/ai.openclaw.gateway.plist` 中的 `PATH`。

### 登录 Cursor 账号（必做）

未登录时 `agent --list-models` 会显示 `No models available for this account`，插件无法选模型：

```bash
agent login
agent --list-models   # 应列出可用模型
```

---

## 三、安装 openclaw-cursor-brain 插件

### 3.1 使用 plugins，不要用 hooks

这是 **Gateway 插件**（含 `openclaw.plugin.json`），不是 Hook 包：

```bash
# 正确
openclaw plugins install openclaw-cursor-brain ...

# 错误（会报 openclaw.hooks 相关错误）
openclaw hooks install openclaw-cursor-brain
```

### 3.2 带安全确认的安装命令

OpenClaw 会扫描插件中的 `child_process`、网络访问等模式并**默认拦截**。该插件需要启动 `cursor-agent` 与本地 proxy，属正常行为，需显式放行：

```bash
openclaw plugins install openclaw-cursor-brain --dangerously-force-unsafe-install
```

安装过程中会出现 `WARNING: dangerous code patterns`，可忽略。

### 3.3 首次安装后的标准流程

```bash
export PATH="$HOME/.local/bin:$PATH"

openclaw plugins install openclaw-cursor-brain --dangerously-force-unsafe-install
openclaw cursor-brain setup      # 交互选择主模型 / 备用模型（需 TTY）
openclaw gateway restart
openclaw cursor-brain doctor
```

非交互环境（无 TTY）下 `setup` 不会自动弹出，安装后手动执行：

```bash
openclaw cursor-brain setup
openclaw gateway restart
```

### 3.4 重装 / 升级

目录已存在时：

```bash
# 覆盖重装
openclaw plugins install openclaw-cursor-brain --dangerously-force-unsafe-install --force

# 或升级
openclaw cursor-brain upgrade openclaw-cursor-brain
# 或
openclaw plugins update openclaw-cursor-brain
```

---

## 四、验证安装

### 4.1 Doctor 健康检查

```bash
openclaw cursor-brain doctor
```

期望大部分为 ✓，典型通过项：

|   |   |
|---|---|
|检查项|说明|
|Plugin version|如 v1.5.4|
|Cursor Agent CLI|`~/.local/bin/agent`|
|Cursor mcp.json|`openclaw-gateway` 已写入|
|Streaming provider|`cursor-local` → `http://127.0.0.1:18790/v1`|
|Output format|一般为 `stream-json`|

以下两项常为**误报或可忽略**：

- **Discovered tool candidates**：按插件 `src/*.ts` 启发式统计，无 `src` 目录的插件会显示 0
    
- **Gateway REST API**：Gateway 刚重启时可能短暂 Unreachable，稍后再测
    

### 4.2 其他自检命令

```bash
openclaw gateway status
openclaw cursor-brain status
openclaw cursor-brain proxy          # Streaming proxy 应在 :18790
curl -s http://127.0.0.1:18790/v1/health | head
```

### 4.3 自动生成的配置

**`~/.cursor/mcp.json`**（Cursor IDE 调用 OpenClaw 工具）：

```json
{
  "mcpServers": {
    "openclaw-gateway": {
      "command": "node",
      "args": ["~/.openclaw/extensions/openclaw-cursor-brain/mcp-server/server.mjs"],
      "env": {
        "OPENCLAW_GATEWAY_URL": "http://127.0.0.1:18789",
        "OPENCLAW_GATEWAY_TOKEN": "<你的 token>",
        "OPENCLAW_CONFIG_PATH": "~/.openclaw/openclaw.json"
      }
    }
  }
}
```

**`~/.openclaw/openclaw.json`** 中由 `setup` 写入的关键段：

- `models.providers.cursor-local` → 指向本地 proxy `http://127.0.0.1:18790/v1`
    
- `agents.defaults.model.primary` → 如 `cursor-local/auto`
    
- `plugins.entries.openclaw-cursor-brain` → 插件开关与 proxy 参数
    

---

## 五、日常维护

### 5.1 常用 CLI

|   |   |
|---|---|
|命令|作用|
|`openclaw cursor-brain doctor`|健康检查|
|`openclaw cursor-brain status`|版本、模型、proxy 状态|
|`openclaw cursor-brain setup`|重新选择主/备模型|
|`openclaw cursor-brain proxy`|查看 proxy PID / 端口|
|`openclaw cursor-brain proxy restart`|仅重启流式代理（不重启整个 Gateway）|
|`openclaw cursor-brain proxy log`|查看 proxy 日志|
|`openclaw gateway restart`|修改配置或升级插件后加载|

### 5.2 日志位置

|   |   |
|---|---|
|文件|内容|
|`~/.openclaw/logs/cursor-proxy.log`|Streaming proxy 主日志|
|`~/.openclaw/logs/cursor-proxy.stderr.log`|Proxy 标准错误|
|`~/.openclaw/logs/gateway.log`|Gateway 日志|
|`/tmp/openclaw/openclaw-*.log`|当日 Gateway 文件日志|

### 5.3 插件配置项

在 `openclaw.json` → `plugins.entries.openclaw-cursor-brain.config`：

|                   |           |                        |
| ----------------- | --------- | ---------------------- |
| 参数                | 默认值       | 说明                     |
| `cursorPath`      | 自动探测      | `agent` 可执行文件路径        |
| `proxyPort`       | `18790`   | Streaming proxy 端口     |
| `outputFormat`    | 自动        | `stream-json` 或 `json` |
| `instantResult`   | `true`    | 批量结果是否即时返回             |
| `forwardThinking` | `content` | 推理过程转发方式               |

主模型 / 备用模型通过 `setup` 写入 `agents.defaults.model`，不在插件 config 内。

### 5.4 卸载

```bash
openclaw cursor-brain uninstall
# 或在插件目录
cd ~/.openclaw/extensions/openclaw-cursor-brain && npm run uninstall
openclaw gateway restart
```

---

## 六、故障排查

### 6.1 `package.json missing openclaw.hooks`

**原因**：插件被安全扫描拦截后，安装器回退尝试按 Hook 包解析失败。  
**处理**：使用 `plugins install` + `--dangerously-force-unsafe-install`，不要理会 `openclaw.hooks` 这条次要报错。

### 6.2 `Plugin installation blocked: dangerous code patterns`

**原因**：同上，属 OpenClaw 安全策略。  
**处理**：

```bash
openclaw plugins install openclaw-cursor-brain --dangerously-force-unsafe-install
```

### 6.3 `plugin already exists`

**原因**：重复执行 install。  
**处理**：已安装则跳过；需重装加 `--force`，或 `openclaw plugins update openclaw-cursor-brain`。

### 6.4 `Cursor Agent CLI not found`

**原因**：未安装 `agent` 或未加入 PATH。  
**处理**：见第二节；或在 config 中指定路径：

```bash
openclaw config set plugins.entries.openclaw-cursor-brain.config.cursorPath "/Users/<你>/.local/bin/agent"
```

### 6.5 `No models available for this account`

**原因**：未执行 `agent login`。  
**处理**：`agent login` 后重新 `openclaw cursor-brain setup`。

### 6.6 `plugins.allow: plugin not found: modelstudio`

**原因**：`openclaw.json` 的 `plugins.allow` 中残留已卸载插件名。  
**处理**：编辑 `~/.openclaw/openclaw.json`，从 `plugins.allow` 删除 `modelstudio` 等无效项，然后 `openclaw gateway restart`。

### 6.7 默认模型不是 cursor-local

**现象**：报 `No API key found for provider "anthropic"` 等。  
**处理**：

```bash
openclaw config set agents.defaults.model.primary "cursor-local/auto"
openclaw cursor-brain setup
openclaw gateway restart
```

### 6.8 Proxy 未启动 / 18790 无响应

```bash
openclaw cursor-brain proxy log
openclaw cursor-brain proxy restart
openclaw gateway restart
curl http://127.0.0.1:18790/v1/health
```

### 6.9 Gateway REST API 检查失败

Doctor 通过 curl 探测 `http://127.0.0.1:18789/tools/invoke`；Gateway 重启瞬间可能失败。  
确认 `openclaw gateway status` 为 running 后隔几秒再跑 `doctor`。

---

## 七、安装检查清单

按顺序勾选：

- OpenClaw 已安装且 `openclaw --version` 正常
    
- Cursor IDE 已安装
    
- `agent --version` 有输出
    
- `agent login` 已完成
    
- `agent --list-models` 能列出模型
    
- `openclaw plugins install openclaw-cursor-brain --dangerously-force-unsafe-install` 成功
    
- `openclaw cursor-brain setup` 已完成
    
- `openclaw gateway restart` 已执行
    
- `openclaw cursor-brain doctor` 核心项为 ✓
    
- `curl http://127.0.0.1:18790/v1/health` 返回 `status: ok`
    

---

## 八、参考链接

- 插件仓库：[andeya/openclaw-cursor-brain](https://github.com/andeya/openclaw-cursor-brain)
    
- npm：[openclaw-cursor-brain](https://www.npmjs.com/package/openclaw-cursor-brain)
    
- OpenClaw 插件文档：[docs.openclaw.ai/cli/plugins](https://docs.openclaw.ai/cli/plugins)
    
- Cursor Agent 安装：`curl -fsSL https://cursor.com/install | bash`
    

---

_文档基于 OpenClaw 2026.4.22 + openclaw-cursor-brain 1.5.4 在 macOS 上的实际安装过程整理。_