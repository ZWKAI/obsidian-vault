**Step 1：跳过官方登录验证**

bash

mkdir -p ~/.claude && echo '{"hasCompletedOnboarding": true}' > ~/.claude.json

**Step 2：配置国产模型**（以阿里云百炼为例）

编辑配置文件：

bash

mkdir -p ~/.claude
nano ~/.claude/settings.json

写入以下内容[](https://help.aliyun.com/zh/model-studio/claude-code)[](https://developer.aliyun.com/article/1736024)：

json

{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "你的API Key",
    "ANTHROPIC_BASE_URL": "https://dashscope.aliyuncs.com/compatible-mode/v1",
    "ANTHROPIC_MODEL": "qwen3.6-plus"
  }
}
