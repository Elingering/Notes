## 验证redis链接
ps -ef | grep redis
netstat -antpl | grep redis
redis-cli -h ip -p port ping