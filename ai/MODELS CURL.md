### 支持的模型列表
curl -H "Authorization: Bearer 28e0183ae6195a42b0ff4fbc470f962c0981fbe6173f8ab9e5b598877940c915" https://unitoken-api.suixingpay.com/v1/models



### 测试模型是否可用
curl -X POST https://unitoken-api.suixingpay.com/v1/messages \
  -H "Authorization: Bearer 28e0183ae6195a42b0ff4fbc470f962c0981fbe6173f8ab9e5b598877940c915" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek/deepseek-v4-pro",
    "messages": [{"role": "user", "content": "hi"}],
    "max_tokens": 100
  }'


curl -X POST https://unitoken-api.suixingpay.com/v1/messages \
  -H "Authorization: Bearer 28e0183ae6195a42b0ff4fbc470f962c0981fbe6173f8ab9e5b598877940c915" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "deepseek/deepseek-v4-pro",
    "messages": [{"role": "user", "content": "Say hello"}],
    "max_tokens": 100
  }'