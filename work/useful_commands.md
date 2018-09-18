## 查看hadoop streamming帮助
hadoop jar /opt/cloudera/parcels/CDH-5.8.5-1.cdh5.8.5.p0.5/lib/hadoop-mapreduce/hadoop-streaming.jar -info|less

## 端口转发
ssh -CfNg -L 8300:192.168.7.104:5000 -p 60103 common@183.60.142.144
Z9w+aBeZSbf4uKDN
将192.168.7.104:5000映射到127.0.0.1:8300

## 抓包
sudo tcpdump -i any port 9089 -Xnpls0 -nn

## use of xargs
find . 2>/dev/null | grep '.h' | grep -v .svn | grep -v '_' | xargs -n 1 grep -n -H -i t_sms_num_getcount_byeid
find . -name "*.conf" |xargs sed -i -e 's/CAROL_/LITE_/g'
find . -name "*.cfg" |xargs sed -i -e 's/CAROL_/LITE_/g'
find . -name "*reload*" | xargs sed -i -e 's/\/data\/carol/\/home\/common\/lite/g'

## replace content in files
sed -i -e 's/192.168.1.41/192.168.1.43/g' *.cfg

## 计算access_token的步骤
加密的签名，算法为:md5(md5(经销商账号的密码) + time)，md5值使用32位小写字符串
用date +%s得到当前的时间戳
date -d "2010-10-02" "+%s"
根据上面的公式，通过[此网站](https://md5jiami.51240.com/)算出token

## Finder前往文件夹
shift+command+G

## Fiddler显示ip
FiddlerObject.UI.lvSessions.AddBoundColumn("Server IP", 120, "X-HostIP");

## CURL显示本次调用时间
[详情](https://stackoverflow.com/questions/18215389/how-do-i-measure-request-and-response-times-at-once-using-curl)

## 查询两个匹配行之间所有的内容
cat import_device.log |grep -A10000 "2018-08-16" |grep -B10000 "2018-08-16"

## 查看进程的启动时间
ps -eo pid,lstart,cmd

## 查找一个文件中存在，另一个文件中不存在的内容
diff file2 file1 | grep '^>' | sed 's/^>\ //'
diff --new-line-format="" --unchanged-line-format=""  <(sort file1) <(sort file2)
