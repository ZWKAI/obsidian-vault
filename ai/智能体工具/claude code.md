**Step 1：跳过官方登录验证**

mkdir -p ~/.claude && echo '{"hasCompletedOnboarding": true}' > ~/.claude.json

**Step 2：配置国产模型**

编辑配置文件：

mkdir -p ~/.claude
nano ~/.claude/settings.json

写入以下内容[](https://help.aliyun.com/zh/model-studio/claude-code)[](https://developer.aliyun.com/article/1736024)：

{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "28e0183ae6195a42b0ff4fbc470f962c0981fbe6173f8ab9e5b598877940c915",
    "ANTHROPIC_BASE_URL": "https://unitoken-api.suixingpay.com/v1",
    "ANTHROPIC_MODEL": "deepseek/deepseek-v4-pro"
  }
}
