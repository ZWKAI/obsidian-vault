## 启停
**方式1**
```
# 启动（需管理员密码）
sudo launchctl load /Library/LaunchDaemons/postgresql-18.plist

# 停止
sudo launchctl unload /Library/LaunchDaemons/postgresql-18.plist
```
**方式2**
```
sudo -u postgres /Library/PostgreSQL/18/bin/pg_ctl -D /Library/PostgreSQL/18/data start
sudo -u postgres /Library/PostgreSQL/18/bin/pg_ctl -D /Library/PostgreSQL/18/data stop
```
## 检查状态
```
pg_isready -h localhost -p 5432
# 或
/Library/PostgreSQL/18/bin/psql -U postgres -h localhost
```