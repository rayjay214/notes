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
- [好用的vim配置](https://github.com/amix/vimrc)


### 视图
- show tags, :Tlist
- show files and directories, :NERDTree
- 进入原样粘贴模式, :paste

### 操作
- 复制当前行 : yy
- 复制多行 : nyy
- 删除当前行 ：dd
- 删除多行 : ndd
- jump to defination : ctrl + ]
- 复制单词：光标放在单词上面，按#或者*即可
- 删除当前光标下的字符 ：x
- 删除光标之后的单词剩余部分 ：dw
- 删除光标之后的该行剩余部分 ：d$
- 删除当前行，进入INSERT MODE
- 单字符全局替换  :%s/old/new/g <br>
- 批量缩进反缩进: shift+v进入可视模式，选中内容，shift+>/<
  [reference](https://www.linux.com/learn/vim-tips-basics-search-and-replace)<br>
  [whole word only](https://stackoverflow.com/questions/1778501/find-and-replace-whole-words-in-vim)


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

## scons编译问题解决
scons: *** [SConstruct] ValueError : unsupported pickle protocol: 4<br>
python2和python3不能混用，比如之前用python3编译，再用python2编译就会报这个错误，解决：删除.sconsign.dblite文件<br><br>
unknown pseudo-op: `.cfi_startproc', 报很多类似这样的错误<br>
汇编器（/usr/bin/as）的版本太低，解决：参照以下两个链接<br>
[原因](https://stackoverflow.com/questions/8872517/g-4-6-1-compiler-error-error-unknown-pseudo-op-cfi-personality)<br>
[升级binutil](https://blog.csdn.net/u011334738/article/details/81186345)<br>

## ICE编译问题解决
下载源码包安装，如果直接去make，是不能支持C++11的，需要在make的时候增加这两个选项make CXXFLAGS=-std=c++11 CONFIGS=cpp11-shared<br>
如果碰到Compilation fails with "relocation R_X86_64_32 against `.rodata.str1.8' can not be used when making a shared object"，这类问题，需要编译的时候增加-fPIC选项 make CXXFLAGS="-std=c++11 -fPIC"<br>
如果碰到undefined reference to symbol 'pthread_create@@GLIBC_2.2.5', 需要加上 -pthread CXXFLAGS="-std=c++11 -fPIC -pthread"<br>
[refer](https://doc.zeroc.com/ice/3.6/ice-release-notes/using-the-linux-binary-distributions#UsingtheLinuxBinaryDistributions-C++)

## 亿赛通破解
把cpp文件改成hpp后缀，拷贝到git目录下，用vscode打开这个目录，再提交到github上   
有时可能拷贝过来的文件就是乱码，这时只能新建一个xxx.hpp文件，然后在vscode里面将内容拷贝过去，然后提交xxx.hpp
