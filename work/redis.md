## 维护

### 批量删除
```
redis-cli -p 9200 --scan --pattern 'T_MOBILE_PICTURE#*'  >  del_key_result
cat del_key_result  | xargs ./redis-cli -p 9200  del 
```
