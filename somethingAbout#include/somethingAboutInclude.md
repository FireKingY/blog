---
title: 关于C/C++中的#include，以及变量的声明与定义
date: 2019-05-07 00:37:00
tags: c,c++,#include,extern
---
# 关于C/C++中的#include
---
## 编译器对#include的处理

编译器在遇见`#include "a.h"`时,会将a.h就地展开,简单来说就是把文件a.h中的内容复制一份到`#include`处.这一动作在预处理阶段完成.对于g++可以参数`-E`只进行预处理.

```c++
//a.h
#define RUN 1
int cur;
```
```c++
//b.h
void funB()
```
```c++
//main.cpp
#include "a.h"
#include "b.h"

int main()
{
    cur = RUN;
}

funB(){}
```

使用命令`g++ main.cpp -E > main.i`得到:
```c++
# 1 "main.cpp"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "main.cpp"
# 1 "a.h" 1

int cur;
# 2 "main.cpp" 2
# 1 "b.h" 1
void funB();
# 3 "main.cpp" 2

int main()
{
    cur = 1;
}

void funB(){}
```
再如:
```c++
//a.h
int a = 1;
int b = 1;
```
```c++
//main.cpp
int main()
{
    #include "a.h"
    int c = a + b;
}
```
得到的main.i
```c++
# 1 "main.cpp"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "main.cpp"
int main()
{
# 1 "a.h" 1
int a = 1;
int b = 1;
# 4 "main.cpp" 2
    int c = a + b;
}
```
## 可能引发的错误
### 多重定义 redefinition
此类错误本质上是在头文件中定义而非声明变量/函数所造成的。
```c++
//a.h
int a;
int b;
``` 
```c++
//main.cpp
#include "a.h"

int b;

int main()
{
    a = 1;
    b = 1;
}
```
使用`g++ main.cpp`编译会报错:
```c++
main.cpp:3:5: error: redefinition of 'int b'
 int b;
     ^
In file included from main.cpp:1:
a.h:2:5: note: 'int b' previously declared here
 int b;
```
原因是变量`b`重定义了。运行`g++ main.cpp -E　> main.i`，查看得到的main.i
```c++
# 1 "main.cpp"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "main.cpp"
# 1 "a.h" 1
int a;
int b;
# 2 "main.cpp" 2

int b;

int main()
{
    a = 1;
    b = 1;
}
```
`int b;`定义了两次。
再如：
```c++
//a.h
int a;
```
```c++
//b.h
#include "a.h"
int b;
```
```c++
//c.h
#include "a.h"
int c;
```
```c++
//main.cpp
#include "b.h"
#include "c.h"

int main()
{
}
```
编译报错:
```c++
In file included from c.h:1,
                 from main.cpp:2:
a.h:1:5: error: redefinition of 'int a'
 int a;
     ^
In file included from b.h:1,
                 from main.cpp:1:
a.h:1:5: note: 'int a' previously declared here
 int a;
```
查看预处理结果
```c++
# 1 "main.cpp"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "main.cpp"
# 1 "b.h" 1
# 1 "a.h" 1
int a;
# 2 "b.h" 2
int b;
# 2 "main.cpp" 2
# 1 "c.h" 1
# 1 "a.h" 1
int a;
# 2 "c.h" 2
int c;
# 3 "main.cpp" 2

int main()
{
}
```
a.h分别在b.h和c.h中引用了两次,编译器将a.h中的内容复制了两份到main.cpp中。若在头文件中只有声明，没有定义，则无论如何包含都不会出现多重定义错误。

### 相互引用
```c++
//a.h
#pragma once
#include "b.h"
typedef int myint;
mychar a;
```
```c++
//b.h
#pragma once
#include "a.h"
typedef char mychar;
myint b;
```
```c++
//main.cpp
#include "a.h"

int main()
{
}
```
编译报错:
```c++
In file included from a.h:2,
                 from main.cpp:1:
b.h:4:1: error: 'myint' does not name a type; did you mean 'int'?
 myint b;
 ^~~~~
 int
```
查看预处理结果:
```c++
# 1 "main.cpp"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "main.cpp"
# 1 "a.h" 1
       
# 1 "b.h" 1
       

typedef char mychar;
myint b;
# 3 "a.h" 2
typedef int myint;
mychar a;
# 2 "main.cpp" 2


int main()
{
}
```
可见`myint b;`在`typedef in myint;`之前。编译器首先遇见`#include "a.h"`，便把a.h展开为
```c++
//#include "a.h"
#include "b.h"
typedef int myint;
mychar a;
```
然后扫描到`#include "b.h"`,展开得到:
```c++
//#include "a.h"
//#include "b.h"
#include "a.h"
typedef char mychar;
myint b;
typedef int myint;
mychar a;
```
这时又扫描到`#include "a.h"`，由于在a.h中写了`#pragma once`，故这里不会再展开，最后结果
```c++
typedef char mychar;
myint b;
typedef int myint;
mychar a;
```
造成了`myint b;`在`typedef in myint;`之前出现。解决方法将a.h和b.h共有的类型定义放到额外的头文件中，然后再a.h和b.h中包含它。
```c++
//common.h
#pragma once
typedef int myint;
typedef char mychar
```
```c++
//a.h
#pragma once
#include "common.h"
mychar a;
```
```c++
//b.h
#pragma once
#include "common.h"
myint b;
```
预处理结果
```c++
# 1 "main.cpp"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "main.cpp"
# 1 "a.h" 1
       
# 1 "common.h" 1
       
typedef int myint;
typedef char mychar;
# 3 "a.h" 2
mychar a;
# 2 "main.cpp" 2
# 1 "b.h" 1
       

myint b;
# 3 "main.cpp" 2

int main()
{
}
```
---
## 声明与定义
两者区别在于编译器遇见定义（全局变量）时，会为其分配一块空间用处存储该变量，而引用只是告诉编译器有这么一个变量，（编译器会将该变量的信息加到符号表中:未证实）。一个变量可以有多次声明，但只能有一次定义,多次声明必须一致。无法在仅声明变量未定义变量的情况下使用变量，因为没有对应的存储空间。
如果一个变量在多个文件中均有定义，各个文件能够编译成二进制代码，但在链接阶段时，由于有多个定义，也就有多个对应的空间，编译器不知道该怎么处理便会报多重定义错误。

### 变量的声明与定义
```c++
extern int a; //声明
extern int a = 1; //定义
int a; //定义
int a = 1; //定义
```
考虑下列代码:
```c++
//main.cpp
extern int a;
int main()
{
    a = 1;
    return 0;
}
```
```c++
//def.cpp
int a = 1;
```
使用`g++ main.cpp`编译会报`undefined reference to 'a'`,这是因为在main.cpp中只进行了声明，未进行定义。该变量的定义在def.cpp中。使用`g++ main.cpp def.cpp`即可编译成功。

### 函数的声明与定义
```c++
void fun(int); // 声明
void fun(int a){} //定义，分配对应的存储空间存储代码
```
