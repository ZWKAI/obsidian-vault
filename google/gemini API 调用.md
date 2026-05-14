### api key管理：
[https://aistudio.google.com/api-keys](https://aistudio.google.com/api-keys)

### 请求示例：
curl:
 curl "https://generativelanguage.googleapis.com/v1beta/models/gemini-flash-latest:generateContent" \
  -H 'Content-Type: application/json' \
  -H 'X-goog-api-key: AIzaSyB1LvR1C4T6UxwxdcHZoO1Q4aOGI9GueN8' \
  -X POST \
  -d '{
    "contents": [
      {
        "parts": [
          {
            "text": "简单介绍ai是如何工作的"
          }
        ]
      }
    ]
  }'