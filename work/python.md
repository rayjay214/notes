## 包机制

### 手动执行成功，crontab执行报错
查看报错信息，如果是某个库没找到，可能是因为crontab执行时LD_LIBRARY_PATH不同，在bash脚本中手动加入路径即可
