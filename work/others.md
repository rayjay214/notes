## 浏览器url中特殊字符
-  +号表示空格 %2B  
-  空格可以用+号或者编码 %20  
-  /分隔目录和子目录 %2F  
-  ?分隔实际的URL和参数 %3F  
-  %指定特殊字符 %25  
-  #表示书签 %23  
-  &参数间的分隔符 %26  
-  =指定参数的值 %3D

- [所有特殊字符一览](https://stackoverflow.com/questions/6182356/what-is-2c-in-a-url)


## 关于字节，字符，编码
[字节，字符，编码问题](http://www.regexlab.com/zh/encoding.htm)

## VIM RELATED
### 问题
- 安装vimdiff: yum install vim-enhanced
- vim ctag不能跳转：将.也当成单词的一部分了，所以跳转的时候提示找不到tag，在~/.vimrc中将set iskeyword+=. 这行注释
[好用的vim配置](https://github.com/amix/vimrc)


### 视图
- show tags : :Tlist
- show files and directories: :NERDTree
- 进入原样粘贴模式 : :paste

### 操作
- jump to defination : ctrl + ]
- 复制单词：光标放在单词上面，按#或者*即可
- 删除当前光标下的字符 ：x
- 删除光标之后的单词剩余部分 ：dw
- 删除光标之后的该行剩余部分 ：d$
- 删除当前行 ：dd
- 删除当前行，进入INSERT MODE


### 配置
- 高亮显示当前列 ：set cuc


## C++ 打印文件名行号宏
[示例](https://blog.csdn.net/u013187074/article/details/78874976)<br>
[__VA_ARGS__](https://blog.csdn.net/cqupt_chen/article/details/8055215)
```
#define DBG_OUTPUT(fmt,args...) printf("CK File[%s:%s(%d)]:" fmt "\n", __FILE__,__FUNCTION__, __LINE__, ##args)
DBG_OUTPUT("j[%d]k[%d]", j, k);
/* printf("CK File[%s:%s(%d)]:" "j[%d]k[%d]" "\n", __FILE__, __FUNCTION__, __LINE__, j, k) */
```
