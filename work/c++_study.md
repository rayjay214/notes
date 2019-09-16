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
