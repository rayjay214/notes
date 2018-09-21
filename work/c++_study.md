## Initializer List
[details](https://www.geeksforgeeks.org/when-do-we-use-initializer-list-in-c/)

## 虚函数和纯虚函数
**定义一个函数为虚函数，不代表函数为不被实现的函数。<br>**
**定义他为虚函数是为了允许用基类的指针来调用子类的这个函数。<br>**
**定义一个函数为纯虚函数，才代表函数没有被实现。<br>**
定义纯虚函数是为了实现一个接口，起到一个规范的作用，规范继承这个类的程序员必须实现这个函数。
在基类中实现纯虚函数的方法是在函数原型后加"=0"<br>
　virtual void funtion1()=0

