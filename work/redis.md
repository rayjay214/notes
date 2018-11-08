## 命令相关

### 批量删除
`redis-cli KEYS "prefix:*" | xargs redis-cli DEL`
