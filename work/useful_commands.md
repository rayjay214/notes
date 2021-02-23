## 查看hadoop streamming帮助
```
hadoop jar /opt/cloudera/parcels/CDH-5.8.5-1.cdh5.8.5.p0.5/lib/hadoop-mapreduce/hadoop-streaming.jar -info|less
```

## 端口转发
ssh -CfNg -L 8300:192.168.7.104:5000 -p 60103 common@183.60.142.144
Z9w+aBeZSbf4uKDN
将192.168.7.104:5000映射到127.0.0.1:8300

## 抓包
`sudo tcpdump -i any port 9089 -Xnpls0 -nn`  
-nn: 指定将每个监听到的数据包中的域名转换成IP、端口从应用名称转换成端口号后显示  
-X: 需要把协议头和包内容都原原本本的显示出来  
-s snaplen  snaplen表示从一个包中截取的字节数。0表示包不截断，抓完整的数据包。默认的话 tcpdump 只显示部分数据包,默认68字节  
-l 使标准输出变为缓冲行形式  
-p 将网卡设置为非混杂模式，不能与host或broadcast一起使用  
-n 不把网络地址转换成名字  

## use of xargs
```
find . 2>/dev/null | grep '.h' | grep -v .svn | grep -v '_' | xargs -n 1 grep -n -H -i t_sms_num_getcount_byeid
find . -name "*.conf" |xargs sed -i -e 's/CAROL_/LITE_/g'
find . -name "*.cfg" |xargs sed -i -e 's/CAROL_/LITE_/g'
find . -name "*reload*" | xargs sed -i -e 's/\/data\/carol/\/home\/common\/lite/g'
```

## replace content in files
`sed -i -e 's/192.168.1.41/192.168.1.43/g' *.cfg`

## remove line with specific str
`sed -i '/pattern to match/d' ./infile`

## 计算access_token的步骤
加密的签名，算法为:md5(md5(经销商账号的密码) + time)，md5值使用32位小写字符串<br>
用date +%s得到当前的时间戳<br>
`date -d "2010-10-02" "+%s"`<br>
根据上面的公式，通过[此网站](https://md5jiami.51240.com/)算出token

## Finder前往文件夹
shift+command+G

## Fiddler显示ip
FiddlerObject.UI.lvSessions.AddBoundColumn("Server IP", 120, "X-HostIP");

## CURL显示本次调用时间
[详情](https://stackoverflow.com/questions/18215389/how-do-i-measure-request-and-response-times-at-once-using-curl)

## 查询两个匹配行之间所有的内容
`cat import_device.log |grep -A10000 "2018-08-16" |grep -B10000 "2018-08-16"`

## 查看进程的启动时间
`ps -eo pid,lstart,cmd`

## 查找一个文件中存在，另一个文件中不存在的内容
`diff file2 file1 | grep '^>' | sed 's/^>\ //'`<br>
`diff --new-line-format="" --unchanged-line-format=""  <(sort file1) <(sort file2)`

## 抓包
sudo tcpdump -i any port 29999 tcp -Xnpls0 -nn

## 复制软链接
本地：cp -d <br>
远程：rsync -Wav


## bash，时间戳日期转换
`date -d "2018-02-02 00:00:00" "+%s"`
`date -d @1519228800`

## bash字符串替换
### 换行替换成comma
`cat input.txt | tr '\n' ','`

### 每一行前后都加个双引号
```awk '{ print "\""$0"\""}' file```

### double quote to comma, then eliminate the whitespace
`cat wireless.txt |tr '"' ' '|tr -d ' '`

## bash操作
### 编辑常用快捷键
```
ctrl + w —往回删除一个单词，光标放在最末尾 
ctrl + k —往前删除到末尾，光标放在最前面（可以使用ctrl+a） 
ctrl + u 删除光标以前的字符 
ctrl + k 删除光标以后的字符 
ctrl + a 移动光标至的字符头 
ctrl + e 移动光标至的字符尾 
ctrl + l 清屏
```

