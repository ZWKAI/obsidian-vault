生成ssh公私钥

在虚拟机实例执行：
 echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCqa/9PY5E1UDHF/sLMyh//bPpMrtZrx4F/gwlXhjuZupSe+pOwWVCvVNNdDoV8ruTQPOQmCpBh1VzQ+n6FT/3OpZ80FbxY5OnLuw3l/X44mxxiQnuMU7saMI8g5NahDW03jr12d9JC7VBSwHKO3a7evVZYwx2g4cB92QeZzWegK59thNfOVtXo4U1acX+C2j8ApfqiSOnvTli1aMHXBqRLD+h0G6nC5pXjNWSQ/En/hiy+d/xR78WeQNgoIQkstP2CMe9mB+qewTsdaNSzYzcX/gQMcEngI61QyTi9oT/9P/b01uvgQqVQ1FjDCs3DUuYQhDfkVXmJWNb5B7mN2Afxaj6LW7zfu/C6lTi+liXbv4G+siNi7lay1tsTGSaDATeUS95yJStwi077RjmJzEcfsziFeVhnUkvrcKoflu0GgpOje26h2PwBCDBv7nwdBq5QwVymDehwIA1HafH7nhMqXSACpaHbgoaXwDO74d5Xk4yws4Nu5P+ueqdg1BZf6Pmg8VHtlirhuwt7g/9kWxkxUtsiPlUb2hCPchvHUykAEIAgef1in7fLxVdk6UOxb8dX6UCi5OvvODXBpiqSwhAePp/Lxqk2F2o3nqjXuxisRUzxiqpBlkRU69OiRqnf6VO+l4HqRkwGjUJuAX4TFziAbcZtYSnLpnh88rNbBTD6Zw== zhao_wk@suixingpay.com' > /tmp/mykey.pub && cat /tmp/mykey.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && rm /tmp/mykey.pub

使用cursor remote ssh连接使用


虚拟机：
alias openclaw='cd /home/zhaowenkai_china/openclaw-docker/openclaw && docker compose run --rm openclaw-cli'
然后执行：source ~/.bashrc


重启网关：
cd /home/zhaowenkai_china/openclaw-docker/openclaw
docker compose stop openclaw-gateway
docker compose up -d openclaw-gateway

启动服务：openclaw doctor
openclaw status
openclaw agents list

本地：
ssh -L 18789:127.0.0.1:18789 zhaowenkai_china@104.155.199.40

查看文件：
把整个配置目录的属主改成当前用户：
sudo chown -R "$USER:$USER" /home/zhaowenkai_china/.openclaw

 让容器内 node 用户 (uid 1000) 能读写
sudo chown -R 1000:1000 /home/zhaowenkai_china/.openclaw