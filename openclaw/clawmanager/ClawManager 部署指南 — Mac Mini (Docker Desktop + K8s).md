
> 环境：macOS Apple Silicon / Docker Desktop / Kubernetes v1.34.3

> StorageClass：`standard`（default）/ `hostpath`，provisioner: `rancher.io/local-path`


## 一、环境验证（已完成）

  
```bash

kubectl get nodes

kubectl get storageclass

kubectl get ns

```

## 二、克隆项目

  

```bash

cd ~/it

git clone https://github.com/Yuan-lab-LLM/ClawManager.git

cd ClawManager

```

  

## 三、修改部署清单适配 Docker Desktop

  

ClawManager 的 k3s 清单中 StorageClass 硬编码为 `local-path`，

Docker Desktop 的默认 StorageClass 是 `standard`，需要修改两处。

  

### 3.1 复制一份用于修改

  

```bash

cp deployments/k3s/clawmanager.yaml deployments/k3s/clawmanager-dockerdesktop.yaml

```

  

### 3.2 修改 StorageClass 名称

  

```bash

# 将 PVC 中的 storageClassName: local-path 改为 standard

sed -i '' 's/storageClassName: local-path/storageClassName: standard/g' \

deployments/k3s/clawmanager-dockerdesktop.yaml

  

# 将环境变量 K8S_STORAGE_CLASS 的值也改为 standard

sed -i '' 's/value: "local-path"/value: "standard"/g' \

deployments/k3s/clawmanager-dockerdesktop.yaml

```

  

### 3.3 验证修改

  

```bash

grep -n 'local-path\|storageClassName\|K8S_STORAGE_CLASS' \

deployments/k3s/clawmanager-dockerdesktop.yaml

```

  

确认没有残留的 `local-path` 引用。

  

## 四、预拉取镜像（推荐）

  

Apple Silicon 上部分镜像可能拉取缓慢或不可用，建议先手动拉取。

  

```bash

# clawmanager 主应用（支持 ARM64）

docker pull ghcr.io/yuan-lab-llm/clawmanager:latest

  

# MySQL（支持 ARM64）

docker pull mysql:8.4.8

```

  

> **已知问题**：`ghcr.io/yuan-lab-llm/skill-scanner:latest` 没有 ARM64 版本，

> 无法在 Apple Silicon 上运行。该组件仅用于 Skill 安全扫描，不影响核心功能。

> 部署后需将其缩容为 0（见步骤 5.4）。

  

## 五、部署 ClawManager

  

### 5.1 应用部署清单

  

```bash

kubectl apply -f deployments/k3s/clawmanager-dockerdesktop.yaml

```

  

### 5.2 持续观察 Pod 启动

  

```bash

kubectl get pods -n clawmanager-system -w

```

  

等待以下 Pod 进入 `Running`：

  

| Pod | 说明 | ARM64 支持 |

|-----|------|-----------|

| `mysql-xxx` | 数据库（首次启动会初始化 schema） | ✅ |

| `clawmanager-app-xxx` | 核心应用（前端 + 后端 + Nginx） | ✅ |

| `skill-scanner-xxx` | 技能安全扫描服务 | ❌ 无 ARM64 镜像 |

  

> 首次启动需要从 `ghcr.io` 拉取镜像，可能需要几分钟。

> `clawmanager-app` 依赖 MySQL，如果 MySQL 未就绪会反复重启（CrashLoopBackOff），

> 等 MySQL Running 后会自动恢复。

  

### 5.3 检查 PVC 绑定

  

```bash

kubectl get pvc -n clawmanager-system

```

  

应看到 `mysql-data` 状态为 `Bound`。

  

### 5.4 处理 skill-scanner（Apple Silicon 必需）

  

`skill-scanner` 镜像不支持 ARM64，会一直处于 `ImagePullBackOff`。

将其缩容为 0 即可，不影响核心功能：

  

```bash

kubectl scale deployment skill-scanner -n clawmanager-system --replicas=0

```

  

### 5.5 检查服务

  

```bash

kubectl get svc -n clawmanager-system

```

  

### 5.6 最终确认

  

```bash

kubectl get pods -n clawmanager-system

```

  

预期结果：`mysql` 和 `clawmanager-app` 均为 `Running`（1/1）。

  

### 5.7 如果 Pod 异常，查看详情

  

```bash

# 查看 Pod 事件

kubectl describe pod -n clawmanager-system <pod-name>

  

# 查看日志

kubectl logs -n clawmanager-system <pod-name>

```

  

## 六、访问 Web 界面

  

### 6.1 启动 port-forward（Docker Desktop 必需）

  

Docker Desktop 的 K8s 环境下，NodePort 不会自动映射到 localhost。

需要通过 `kubectl port-forward` 手动转发：

  

```bash

kubectl port-forward -n clawmanager-system svc/clawmanager-frontend 30443:443

```

  

> 该命令需要保持终端运行，关闭终端则端口转发中断。

> 如需后台运行，可加 `&` 或在单独的终端窗口中执行。

  

### 6.2 打开浏览器

  

```

https://localhost:30443

```

  

> 由于是自签名证书，浏览器会提示不安全。

> Chrome：点击「高级」→「继续前往 localhost（不安全）」

> Safari：点击「显示详细信息」→「访问此网站」

  

## 七、首次登录与初始化

  

### 7.1 登录

  

- **用户名**：`admin`

- **密码**：`admin123`

- 建议首次登录后修改默认密码

  

### 7.2 配置安全模型（必须先做，否则无法创建实例）

  

1. 左侧菜单 → **AI 网关** → **模型**

2. 点击新增模型，填写以下信息：

- **显示名称**：自定义名称（如 `GPT-4o`）

- **厂商模板**：选择对应厂商（如 OpenAI）或 Local / Internal

- **协议**：OpenAI Compatible

- **Base URL**：模型服务地址（如 `https://api.openai.com/v1`）

- **API Key**：你的 API Key

- **Provider Model**：实际模型名（如 `gpt-4o`）

- 输入/输出价格：测试可填 `0`

3. **务必勾选**：

- ✅ 安全模型

- ✅ 启用

4. 点击 **保存**

  

### 7.3 创建 OpenClaw 实例

  

1. 点击左下角 **ADMIN** → 切换到 **工作台**

2. 点击 **创建实例**

3. **第 1 步 - 基础信息**：填写实例名称（≥3 字符）→ 下一步

4. **第 2 步 - 选择类型**：选择 **OpenClaw Desktop** → 下一步

5. **第 3 步 - 配置**：选择 Small 规格（2 CPU / 4GB RAM / 20GB Disk）→ 创建

  

> 首次创建需要拉取 OpenClaw 镜像，时间较长，请耐心等待。

  

## 八、常见问题排查

  

### 8.1 镜像拉取失败 / 超时

  

如果 K8s 自动拉取慢或失败，先用 `docker pull` 手动拉取：

  

```bash

docker pull ghcr.io/yuan-lab-llm/clawmanager:latest

docker pull mysql:8.4.8

```

  

拉取完成后删除所有 Pod 让 Deployment 重建（会自动使用本地缓存镜像）：

  

```bash

kubectl delete pod --all -n clawmanager-system

```

  

如果 `ghcr.io` 拉不下来，可考虑配置 Docker Desktop 的代理：

Settings → Docker Engine / Resources → Proxies

  

### 8.2 skill-scanner 无 ARM64 镜像（Apple Silicon）

  

`ghcr.io/yuan-lab-llm/skill-scanner:latest` 仅提供 amd64 版本。

尝试 `docker pull --platform linux/amd64` 可以下载但无法被 K8s containerd 在 ARM 节点上解包使用。

  

解决方案：缩容为 0，跳过该组件：

  

```bash

kubectl scale deployment skill-scanner -n clawmanager-system --replicas=0

```

  

该组件仅影响 Skill 安全扫描功能，不影响实例创建、AI 网关等核心能力。

  

### 8.3 PVC 一直 Pending

  

```bash

kubectl get pvc -n clawmanager-system

kubectl get events -n clawmanager-system --sort-by='.lastTimestamp'

```

  

如果 StorageClass 不匹配，确认修改是否正确生效。

  

### 8.4 clawmanager-app 启动失败

  

通常是 MySQL 尚未就绪导致。等 MySQL Pod 完全 Running 后，

clawmanager-app 会自动重试连接。

  

```bash

kubectl logs -n clawmanager-system deployment/clawmanager-app

```

  

### 8.5 无法创建 OpenClaw 实例

  

确认已在「AI 网关 → 模型」中配置并启用了**安全模型**。

  

### 8.6 需要本地构建镜像（ARM 兼容性问题）

  

如果官方镜像不支持 ARM64：

  

```bash

cd ~/it/ClawManager

docker build -t ghcr.io/yuan-lab-llm/clawmanager:latest .

```

  

## 九、验证检查清单

  

按顺序逐条检查：

  

```bash

# 1. 节点就绪

kubectl get nodes

  

# 2. StorageClass 可用

kubectl get storageclass

  

# 3. Pod 全部 Running

kubectl get pods -n clawmanager-system

  

# 4. PVC 全部 Bound

kubectl get pvc -n clawmanager-system

  

# 5. Service 端口正常

kubectl get svc -n clawmanager-system

  

# 6. 浏览器访问

open https://localhost:30443

```

  

## 十、清理环境

  

如果需要完全卸载：

  

```bash

kubectl delete -f deployments/k3s/clawmanager-dockerdesktop.yaml

kubectl delete ns clawmanager-system

```

  

## 附录：资源规划

  

| 组件 | CPU | 内存 | 磁盘 |

|------|-----|------|------|

| K8s 集群本身 | ~2 核 | ~2 GB | - |

| ClawManager 全套 | ~2 核 | ~4 GB | ~5 GB |

| OpenClaw 实例（Small） | 2 核 | 4 GB | 20 GB |

| macOS 系统预留 | ~4 核 | ~8-10 GB | - |

| **合计** | ~10 核 | ~18-20 GB | ~25 GB |

  

Mac Mini 10C/24G 可以满足测试需求，但建议 Docker Desktop 分配

至少 **8 核 / 16GB 内存 / 60GB 磁盘**。

  

在 Docker Desktop → Settings → Resources 中调整。