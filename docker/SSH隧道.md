
ssh -L 18789:127.0.0.1:18789 zhaowenkai_china@35.189.181.54

访问地址：
http://127.0.0.1:18789/#token=c5b0ed9b1605e2ee1c21c081d65a208abcaa50b49db4f1b1da348ccad135c830

配对：
## 在虚拟机上批准配对

1. 在虚拟机上查看待批准设备：

cd /home/zhaowenkai_china/openclaw-docker/openclaw

openclaw devices list
结果：bb4a5cd8-8cff-4a2d-9c28-efdf84e48cad
（若没有 `openclaw` 命令，用：  
`docker compose run --rm openclaw-cli devices list`）

输出里会有一列 pending 的请求，记下对应的 requestId（或类似 ID）。

2. 批准该设备：

openclaw devices approve <requestId>

openclaw devices approve 69140ef8-b60a-4044-9a22-fbcf05a6f3ee

把 `<requestId>` 换成上一步看到的 ID。  
（用 Docker 时：  
`docker compose run --rm openclaw-cli devices approve <requestId>`）

3. 回到浏览器

刷新控制台页面，再点「连接」，一般就能连上。


连接：
ssh -L 18789:127.0.0.1:18789 zhaowenkai_china@34.81.222.154
配对：

openclaw pairing approve feishu MUC7A9CE