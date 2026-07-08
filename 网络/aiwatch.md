**停止：**
launchctl unload ~/Library/LaunchAgents/com.aiwatch.aiwatchd.plist
**开启：**
launchctl load ~/Library/LaunchAgents/com.aiwatch.aiwatchd.plist

curl -fsSL http://22.50.9.131:9527/install/aiwatchd.sh | bash -s -- \
  --clean \
  --user-code SXF3594 \
  --user-name '赵文凯' \
  --department '科技研发部' \
  --server-url http://22.50.9.131:9527