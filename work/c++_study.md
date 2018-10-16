## Initializer List
[details](https://www.geeksforgeeks.org/when-do-we-use-initializer-list-in-c/)

## 虚函数和纯虚函数
**定义一个函数为虚函数，不代表函数为不被实现的函数。<br>**
**定义他为虚函数是为了允许用基类的指针来调用子类的这个函数。<br>**
**定义一个函数为纯虚函数，才代表函数没有被实现。<br>**
定义纯虚函数是为了实现一个接口，起到一个规范的作用，规范继承这个类的程序员必须实现这个函数。
在基类中实现纯虚函数的方法是在函数原型后加"=0"<br>
　virtual void funtion1()=0

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
