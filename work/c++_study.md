## C++11
[教程](https://changkun.de/modern-cpp/zh-cn/05-pointers/index.html)  
[新特性总结](https://zhuanlan.zhihu.com/p/139515439)  
[右值引用，转移，完美转发等](https://zhuanlan.zhihu.com/p/107445960)  
[move和forward的通俗解释](https://zhuanlan.zhihu.com/p/55856487)  

### 智能指针
非智能指针, 第一次申请的空间不会被释放
```
int * ptr = new int; //第一次申请空间
*ptr = 5;
// ... do some stuff here
ptr = new int; //第二次申请空间
// ... reuse ptr to do some other stuff
```
智能指针  
```
shared_ptr<int> ptr = make_shared<int>();   // allocate an int
*ptr = 5;
// ... do some stuff here
ptr = make_shared<int>();  // the old object is no longer used so deleted automatically
// ... reuse ptr to do some other stuff
```
自定义delete函数，以lambda函数表示，作为shared_ptr构造函数中所需的deleter，示例如下：  
```
auto result = std::shared_ptr<const CassResult>(cass_future_get_result(resultFuture.get()), [](CassResult* p) {
   cass_result_free(p);
});
```
shared_ptr析构部分的部分源码如下：  
```
~SharedPointer() { release(); }

template <typename T>
void SharedPointer<T>::release()
{
	if (--*use_c == 0) {
		if (p) {
			deleter(p);
		}
		delete use_c;
	}
	use_c = nullptr;
	p = nullptr;
}
```


## 编译问题
- [Include issue: 'multiple definition', 'first defined here'](https://stackoverflow.com/questions/45667393/include-issue-multiple-definition-first-defined-here)
- [循环引用问题](https://blog.csdn.net/stockholmrobber/article/details/81161546)

### 前向声明
简化头文件的依赖关系，避免暴露内部类  
前向声明某个类之后，只能使用该类的指针和引用，而不能使用对象更不能访问成员，因为根本不知道这个类含有什么东西

## Initializer List
[details](https://www.geeksforgeeks.org/when-do-we-use-initializer-list-in-c/)

## 虚函数和纯虚函数
**定义一个函数为虚函数，不代表函数为不被实现的函数。<br>**
**定义他为虚函数是为了允许用基类的指针来调用子类的这个函数。<br>**
**定义一个函数为纯虚函数，才代表函数没有被实现。<br>**
定义纯虚函数是为了实现一个接口，起到一个规范的作用，规范继承这个类的程序员必须实现这个函数。
在基类中实现纯虚函数的方法是在函数原型后加"=0"<br>
　virtual void funtion1()=0<br>
[理解虚函数表](https://www.jianshu.com/p/64f3b9c22898)

## erase a element in map
Erasing an element of a map invalidates iterators pointing to that element (after all that element has been deleted). You shouldn't reuse that iterator.

Since C++11 erase() returns a new iterator pointing to the next element, which can be used to continue iterating:
```
it = mymap.begin();
while (it != mymap.end()) {
   if (something)
      it = mymap.erase(it);
   else
      it++;
}
```
Before C++11 you would have to manually advance the iterator to the next element before the deletion takes place, for example like this:
```
mymap.erase(it++);
```

This works because the post-increment side-effect of it++ happens before erase() deletes the element. Since this is maybe not immediately obvious, the C++11 variant above should be preferred.

## socket read/write segment
```
while (1) {
    // Read data into buffer.  We may not have enough to fill up buffer, so we
    // store how many bytes were actually read in bytes_read.
    int bytes_read = read(input_file, buffer, sizeof(buffer));
    if (bytes_read == 0) // We're done reading from the file
        break;

    if (bytes_read < 0) {
        // handle errors
    }

    // You need a loop for the write, because not all of the data may be written
    // in one call; write will return how many bytes were written. p keeps
    // track of where in the buffer we are, while we decrement bytes_read
    // to keep track of how many bytes are left to write.
    void *p = buffer;
    while (bytes_read > 0) {
        int bytes_written = write(output_socket, p, bytes_read);
        if (bytes_written <= 0) {
            // handle errors
        }
        bytes_read -= bytes_written;
        p += bytes_written;
    }
}
```
[refer](https://stackoverflow.com/questions/2014033/send-and-receive-a-file-in-socket-programming-in-linux-with-c-c-gcc-g)
