---
layout:     post
title:      "C++经典面试题"
subtitle:   ""
date:       2018-03-20 12:00:00
author:     "Tango"
header-img: "img/in-post/post-2017-10-11/post-bg-universe.jpg"
catalog: true
tags:   
    - 面试
    - C++对象内存模型
---

最近看了《深入理解C++对象模型》，综合学习C++对象的内存布局，本文基于个人理解和程序测试绘制C++对象模型内存布局，如有错误，还请批评指正。

### 1.前言
首先,相较于C语言，C++语言并没有额外增加内存消耗(确切说，在**没有虚函数情况下**)。
对于一个C++类对象,每个对象有独立的数据成员(**非static**),但是内存中**成员函数只有一份**,该类的所有对象共享成员函数。

static数据成员属于类,该类的所有对象共享static数据成员，static数据成员存储在 **静态存储区**。
对象数据成员依据创建对象的方式不同，可能存在于**栈上**或者**堆上**。
成员函数存储在**代码段**。

当调用对象的成员函数,怎么识别是哪个对象？
`编译器在编译阶段,进行函数的重构，即将成员函数进行非成员化。通过将this指针作为函数的第一个参数,通过this指针即可以找到对象的数据成员; `
例如： 

```
Class Example {
public :
   int a;
   Example,a(a)(int a);
   void print();
}

================编译前=============
int main(){
    Example e = new Example(10);
    e.print();
    return 0;
}

==============编译后=============
int main(){

    Example e = new Example(10);
    print(&e);
    return 0;
}

```

上述是理解C++对象模型的前提。下面详细图示对象内存布局。

下文叙述的前提是:  
 
- 类都存在虚函数
- 子类实现了基类的虚函数  


--- 

### 2. 无继承状态

```C
Class Base {

public:
   int a;
   int b;
   virtual void function();

}

```

![](/img/in-post/cpp/post-no-inherit.png)

如上图所示，对于无继承状态的C++布局:   

```
1）首先是虚函数表指针,该指针是由编译器 定义和初始化(编译阶段，编译器在构造函数内增加代码实现)
3）成员函数代码存储在 代码段,堆上构造虚函数表，将虚成员函数的地址存储在虚函数内。
2）数据成员按照声明的顺序布局;
```
### 3. 单继承

```C
Class Base {

public:
   int a;
   int b;
   virtual void function();
}


Class Derive : public Base {

public:
    int c;
    virtual void function_2();
}

```

![](/img/in-post/cpp/post-cpp-single-inherit.png)

如上图所示，对于无继承状态的C++布局:  

```
1) 首先是基类虚函数表指针
2) 基类数据成员
3) 子类数据成员
4) 子类实现基类的虚函数,并覆盖基类虚函数表中相应的函数的地址
5) 子类扩展基类的虚函数表,将子类的虚函数地址存储在基类虚函数表中
6) 内存中只存在一张虚函数表

```

### 4.多继承
```C
Class Base1 {

public:
   int a;
   int b;
   virtual void function_1();
}

Class Base2 {

public:
   int c;
   virtual void function_2();
}

Class Derive : public Base1,public Base2 {

public:
    int d;
    virtual void function_3();
}

```

![](/img/in-post/cpp/post-cpp-multi-inherit.png)

如上图所示，对于无继承状态的C++布局:  

```
1) 首先是基类1虚函数表指针
2) 基类1数据成员
3) 基类2虚函数表指针
4) 基类2数据成员
5) 子类数据成员
6) 子类实现基类的虚函数,并覆盖基类虚函数表中相应的函数的地址
6) 子类扩展第一个基类的虚函数表,将子类的虚函数地址存储在基类虚函数表中
6) 内存中存在2张虚函数表

```

### 5.虚继承
```C
Class Base {

public:
   int a;
   int b;
   virtual void function_1();
}


Class Derive : public Base {

public:
    int e;
    virtual void function_2();
}

```
![](/img/in-post/cpp/post-cpp-virtual-inherit.png)  

如上图所示，对于无继承状态的C++布局:  

```
1) 首先是基类虚函数表指针
2) 子类数据成员
3) 基类数据成员
4) 子类实现基类的虚函数,并覆盖基类虚函数表中相应的函数的地址
6) 子类扩展第基类的虚函数表,将子类的虚函数地址存储在基类虚函数表中
6) 内存中存在1张虚函数表

```

### 6.菱形继承
```C
Class BaseBase {

public:
   int a;
   int b;
   virtual void function_1();
}

Class Base1:public virtual BaseBase {

public:
   int c;
   virtual void function_2();
}


Class Base2 : public virtual BaseBase {

public:
   int d;
   virtual void function_3();
}

Class Derive : public Base1,public Base2 {

public:
    int e;
    virtual void function_4();
}

```

![](/img/in-post/cpp/post-cpp-multi-virtual-inherit.png)

如上图所示，对于无继承状态的C++布局:  

```
1) 基类1的虚函数表指针vptr
2) 指向基类的基类的 数据成员的指针(实际是在内存中的偏移量)
3) 基类1数据成员
4) 基类2的虚函数表指针
5) 指向基类的基类的 数据成员的指针(实际是在内存中的偏移量)
6) 基类2的数据成员
7) 子类的数据成员
8) 0x0000的空隔
9) 基类的基类的虚函数表指针
10) 基类的基类的 数据成员

```

