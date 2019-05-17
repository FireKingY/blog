---
title: _RTC_CheckEsp 与 函数调用约定
date: 2019-05-01 20:12:57
tags:
---
# _RTC_CheckEsp 与 函数调用约定
## _RTC_CheckEsp
```
#include<stdio.h>
#include<Windows.h>
typedef int(*fun_ype)(int, const char*, const char*, int);
int main()
{
	fun_type msgbox = (fun_type)MessageBoxA;
	msgbox(0, "test", "title", 0);
}
```
在vs中运行上述代码时，在debug模式下vs会报一个错误。这个错误有点奇怪，错误并不是由于哪一行代码出错，而是在main函数结束的花括号那行报的错。错误详信息为：
>Run-Time Check Failure #0 - The value of ESP was not properly saved across a function call.  This is usually a result of calling a function declared with one calling convention with a function pointer declared with a different calling convention.

大意就是说寄存器esp的值出错了。在`msgbox(0, "test", "title", 0);`打断点，调式进入反汇编模式，然后再逐过程调试，发现错误发生在`call        __RTC_CheckEsp (0E0111Dh)`这条语句中，而运行这条语句时，我们写的最后一行代码调用msgbox已经运行完了。
部分汇编代码：
```
	msgbox(0, "test", "title", 0);
00E016F6  mov         esi,esp  
00E016F8  push        0  
00E016FA  push        offset string "title" (0E06B30h)  
00E016FF  push        offset string "test" (0E06B30h)  
00E01704  push        0  
00E01706  call        dword ptr [msgbox]  
00E01709  add         esp,10h  
00E0170C  cmp         esi,esp  
00E0170E  call        __RTC_CheckEsp (0E0111Dh)  
}
```
`call __RTC_CheckEsp (0E0111Dh)` 在debug模式下由编译器生成，用于检查函数调用前后esp是否发生变化，如果发生变化则抛出一个错误。从上面的汇编代码可以看见在调用msgbox过程的第一条汇编语句就是在保存esp，而在`call __RTC_CheckEsp (0E0111Dh)`前的最后一条语句则是在比较调用完函数后的esp和调用函数前的esp，`__RTC_CheckEsp`就是通过比较结果来判断esp是否变化的。
注意在`00E01709  add         esp,10h`和`00E0170C  cmp         esi,esp`之间有一个`add esp, 10h`语句。通过vs反汇编调试，逐过程运行这几条汇编语句，并且跟踪esp的值。在执行完`call dword ptr [msgbox]`后，此时esp的值恰好等于传参前esp的值。因此可以认为就是`add esp, 10h`这条语句造成了esp前后不等的结果。
这里我是通过函数指针的方法来调用这个函数的，在遇到这个错误后我尝试直接调用`MessageBoxA(0,"test","title",0);`发现可以正常运行，并不会出现错误。查看反汇编代码发现果然没有`add esp, 10h`这条语句。因此可以确定这条语句是罪魁祸首了。
直接调用`MessageBoxA`反汇编代码：
```
	MessageBoxA(0, "test", "title", 0);
00B416F6  mov         esi,esp  
00B416F8  push        0  
00B416FA  push        offset string "title" (0B46B30h)  
00B416FF  push        offset string "test" (0B46B30h)  
00B41704  push        0  
00B41706  call        dword ptr [__imp__MessageBoxA@16 (0B4A098h)]  
00B4170C  cmp         esi,esp  
00B4170E  call        __RTC_CheckEsp (0B4111Dh)  
```

这两种调用方式有什么区别呢？两种调用方式都能成功弹出消息框，那么究竟是什么原因造成了其中的区别呢？我并没有找到原因，直到我看了函数调用约定后，回来思考这个问题就豁然开朗了。

## 函数调用约定
函数调用约定主要用于确定函数的传参方式和清栈责任。常见的函数调用约定有`_cdecl`, `_stdcall`,`_fastcall`。
在c/c++中，函数调用约定是函数构成的一部分，我们一般声明函数时包括：
`返回类型 函数名 参数列表 `
我们通常省略了调用约定：
`返回类型 调用约定 函数名 参数列表`
`_cdecl`时c/c++的默认调用约定，当没有直接给出时调用约定时，编译器会默认采用`_cdecl`。注意c++中类的成员函数的默认调用约定为`_thiscall`。

### _cdecl
`_cdecl`规定参数传递顺序为从右到左，由调用者负责清栈。
参数传递顺序指的是参数入栈的顺序，如`void _cdecl fun(int a. int b, int c)`, 参数入栈的顺序为：
```
push c
push b
push a
```
参数通过入栈而被函数使用，在函数执行完其功能后必然要有人将参数出栈，`_cdecl`规定由函数调用者将参数出栈。
编写如下代码验证。
```
#include<stdio.h>
void _cdecl fun(int a, int b, int c) {}
int main()
{
	fun(1, 2, 3);
}
```
在vs中查看反汇编代码：
函数调用部分代码：
```
	fun(1, 2, 3);
0085171E  push        3  
00851720  push        2  
00851722  push        1  
00851724  call        fun (085132Ah)  
00851729  add         esp,0Ch  
```
函数代码：
```
void _cdecl fun(int a, int b, int c) {}
008516D0  push        ebp  
008516D1  mov         ebp,esp  
008516D3  sub         esp,0C0h  
008516D9  push        ebx  
008516DA  push        esi  
008516DB  push        edi  
008516DC  lea         edi,[ebp-0C0h]  
008516E2  mov         ecx,30h  
008516E7  mov         eax,0CCCCCCCCh  
008516EC  rep stos    dword ptr es:[edi]  
008516EE  pop         edi  
008516EF  pop         esi  
008516F0  pop         ebx  
008516F1  mov         esp,ebp  
008516F3  pop         ebp  
008516F4  ret  
```

首先观察参数传递，发现参数传递顺序的确时从右到左。然后观察fun函数的返回语句`ret`，说明fun函数并没有将参数出栈。我们发现在`call fun`后的第一条语句为`add esp, 0ch`，而0ch恰好时fun函数的参数大小，这条语句也就是函数调用者在进行清栈操作。

### _stdcall
`_stdcall`规定函数参数传递顺序为从右到左，由被调函数自身负责清栈。
编写如下代码：
```
#include<stdio.h>
void _stdcall fun(int a, int b, int c) {}
int main()
{
	fun(1, 2, 3);
}
```
调用部分反汇编代码：
```
	fun(1, 2, 3);
001C171E  push        3  
001C1720  push        2  
001C1722  push        1  
001C1724  call        fun (01C1366h)  
}
```
函数反汇编代码：
```
void _stdcall fun(int a, int b, int c) {}
001C16D0  push        ebp  
001C16D1  mov         ebp,esp  
001C16D3  sub         esp,0C0h  
001C16D9  push        ebx  
001C16DA  push        esi  
001C16DB  push        edi  
001C16DC  lea         edi,[ebp-0C0h]  
001C16E2  mov         ecx,30h  
001C16E7  mov         eax,0CCCCCCCCh  
001C16EC  rep stos    dword ptr es:[edi]  
001C16EE  pop         edi  
001C16EF  pop         esi  
001C16F0  pop         ebx  
001C16F1  mov         esp,ebp  
001C16F3  pop         ebp  
001C16F4  ret         0Ch  
```
可以看见fun函数的返回语句为`ret 0ch`，并且在函数调用部分，`call fun`后面并没有`add esp, 0ch`操作。因此验证了参数的确是由被调函数出栈的。

## 回到最初的问题
在知道了`_cdecl`和`_stdcall`后，就能对问题有所思路了。由于在`typedef int(*fun_type)(int, const char*, const char*, int);`中并没有给出函数调用约定，编译器将采用默认的`_cdecl`。因此我们在调用msgbox时是用`_cdecl`的方式的调用的，这下我们就知道了那个恼人的`add esp, 10h`是调用者在进行清栈操作了。
这一以来我们可以猜测问题的根本原因是函数MessageBoxA的调用约定为`_stdcall`，而我们却用'_cdecl'的方式来调用msgbox了。
查看MessageBoxA的声明:
```
WINUSERAPI
int
WINAPI
MessageBoxA(
    _In_opt_ HWND hWnd,
    _In_opt_ LPCSTR lpText,
    _In_opt_ LPCSTR lpCaption,
    _In_ UINT uType);
```
这里WINAPI是一个宏，在minwindef中定义：
```
#define WINAPI      __stdcall
```
果然猜测是正确的。知道了问题的根本的原因后就很好解决了，只要将`typedef int(*fun_type)(int, const char*, const char*, int);`改为`typedef int(_stdcall *fun_type)(int, const char*, const char*, int);`就行了。
