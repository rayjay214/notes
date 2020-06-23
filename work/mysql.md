# 背景相关
[官方版本，Percona，MariaDB之间的关系](https://www.cnblogs.com/wish123/p/11270851.html)


# sql related

## 增加权限
`grant all on *.* to "gns_rw"@"%" identified by '2f0gNsDbWriTExy7s';`<br>
`grant all on 192.168.7.* to "gpsbox_prod_rw"@"%" identified by 'pVwO8sNbh4QpBIAI';`<br>
`grant select on *.* to "gpsbox_prod_r"@"%" identified by 'CkephhAz6uZJmbuV';`<br>
**记得执行flush privileges让权限生效**

## 关联表更新语句
```
update t_user_extend
  INNER JOIN t_user ON t_user.USER_ID = t_user_extend.USER_ID
  set t_user_extend.out_time='2030-06-25 00:00:00'
  where t_user.SCHOOL_ID=3108780;
```

## 修改表结构
```
ALTER TABLE tb_article MODIFY COLUMN NAME CHAR(50);
ALTER TABLE tb_article ADD COLUMN name1 VARCHAR(30); 
ALTER TABLE tb_article CHANGE name1 name2 VARCHAR(30);
ALTER TABLE tb_article DROP COLUMN name2;
```

## 导出表
1. 导出数据库所有表结构<br>
`mysqldump -uroot -pgoome -d dbname -P4000>db.sql;`
2. 导出某张表的表结构<br>
`mysqldump -uroot -pgoome -d dbname test -P4000>db.sql;`
3. 导出数据库所有表结构及数据<br>
`mysqldump -uroot -pgoome dbname -P4000>db.sql;`
4. 导出某张表的表结构及数据<br>
`mysqldump -uroot -pgoome mydbname t1 t2 t3 -P4000>db.sql;`

## 插入主键冲突则更新
`INSERT INTO table (id, name, age) VALUES(1, "A", 19) ON DUPLICATE KEY UPDATE name="A", age=19;`

## 如果记录不存在则插入
pk is not auto increment
```
INSERT IGNORE INTO list 'code' = 'ABC', 'name' = 'abc','place' = 'Johannesburg',;
```
pk is auto increment
```
INSERT INTO list (code, name, address,)
SELECT 'ABC', 'ABC name', 'Johannesburg'
FROM list
WHERE NOT EXISTS(
    SELECT code, name, address
    FROM list
    WHERE code = 'ABC'
	AND name = 'ABC name'
	AND address = 'Johannesburg'
)
LIMIT 1
```

## 通过select执行insert并修改某些其中某些值
```
INSERT INTO keyword_history 
(
   keyword_id, 
   searchengine, 
   location, 
   rec_date, 
   currentrank
) 
SELECT  keyword_id, 
        searchengine, 
        location,
        curdate(),
        '4' 
FROM    keyword_history 
WHERE   history_id = 21;
```


# 维护相关
## 多实例(端口)启动
- 查看实例运行状态: /usr/local/mysql/bin/mysqld_multi report
- 启动某实例: /usr/local/mysql/bin/mysqld_multi start 4000
- 停止某实例: /usr/local/mysql/bin/mysqld_multi stop 4000

配置文件默认是/etc/my.cnf，其中通过[xxxx]符号区分不同实例

ps:启动失败可能是由于环境变量未添加, export PATH=$PATH:/usr/local/mysql/bin

## 查看主从关系
`show slave status\G;`

## 查看同步方式
`show variables like 'binlog_format';`

## 配置同步遇到的问题
同步时遇到1594错误，可能是主从库的mysqlbinlog的版本不一致，如5.6对应的是3.4，5.5对应的是3.3。按照网上的说法，高版本同步到低版本是没问题，低版本同步到高版本可能会有问题。

# 原理、思想相关
## 事务
[事务隔离级别](https://www.jianshu.com/p/4e3edbedb9a8)

## binlog
- [binlog文件解析](https://blog.csdn.net/sgbfblog/article/details/50444822)
- [An API for Reading the MySQL Binary Log](http://assets.en.oreilly.com/1/event/61/Binary%20log%20API_%20A%20Library%20for%20Change%20Data%20Capture%20using%20MySQL%20Presentation.pdf)
- [binlog解析工具的思想](https://www.jianshu.com/p/be3f62d4dce0)
- [binlog伪造slave（gblq）原理](http://blog.51cto.com/11461240/1765412)

## 同步的疑问
主主同步最好用statement？<br>
主从同步用row？<br>
row同步有时会产生多余数据？update找不到对应行的时候会insert。


# 工具相关

## DBeaver配置mysql client
[详情看这里](https://blog.csdn.net/ngl272/article/details/70217499)


