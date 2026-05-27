 **k3s 环境可直接执行** ：

```bash
# 1) 进入项目
cd /root/ClawManager

# 2) 备份当前关键配置（可回滚参考）
k3s kubectl get all -n clawmanager-system -o yaml > clawmanager-system-backup-$(date +%F-%H%M).yaml
k3s kubectl get cm -n clawmanager-system -o yaml > clawmanager-cm-backup-$(date +%F-%H%M).yaml
k3s kubectl get secret -n clawmanager-system -o yaml > clawmanager-secret-backup-$(date +%F-%H%M).yaml

# 3) 更新代码
git fetch --all --tags
git pull

# 4) 应用最新清单（k3s）
k3s kubectl apply -f deployments/k3s/clawmanager.yaml

# 5) 观察滚动升级
k3s kubectl get pods -n clawmanager-system -w
```

当你看到 `clawmanager-app/mysql/minio/skill-scanner` 都变成 `Running` 后，再执行验证：

```bash
# 6) 快速健康检查
k3s kubectl get pods -n clawmanager-system
k3s kubectl get svc -n clawmanager-system
```

如果升级后页面异常，先看后端日志：

```bash
k3s kubectl logs -n clawmanager-system deploy/clawmanager-app --tail=200
```

你执行过程中把报错贴我，我可以按报错给你继续下一步处理。


### 更新镜像
**拉取新镜像（示例）**
k3s crictl pull ghcr.io/yuan-lab-llm/agentsruntime/openclaw:v2026.5.27

#k3s拉取镜像**若仍用 :latest 且怀疑是旧缓存，可先删本地镜像再 pull**
k3s crictl rmi ghcr.io/yuan-lab-llm/agentsruntime/openclaw:latest
k3s crictl pull ghcr.io/yuan-lab-llm/agentsruntime/openclaw:latest

