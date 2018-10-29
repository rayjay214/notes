## 配置相关

[s3cmd多账户配置](http://linuxamination.blogspot.com/2017/12/s3cmd-configure-multiple-s3-accounts-on.html)
[s3cmd with radosgw](https://lollyrock.com/articles/s3cmd-with-radosgw/)

## 编码

[bash下通过http上传文件](http://www.tothenew.com/blog/file-upload-on-amazon-s3-server-using-curl-request/)

## 原理相关

### secret key的作用
当s3生成一个账户的时候，会产生一个公钥和一个私钥。一份私钥保存在服务端，一份私钥提供给管理员。当管理员通过http上传文件到s3的时候，需要通过这个私钥对http中的header生成消息摘要，s3服务端收到http请求时，使用它保存的私钥对header生成消息摘要，如果摘要和传过来的signature是一样的，证明这个请求是合法的，中间没有被篡改过。
