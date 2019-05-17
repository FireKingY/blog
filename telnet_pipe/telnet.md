# 使用匿名管道控制telnet(32位)

---

## 工具：

- [detours](https://www.microsoft.com/en-us/download/details.aspx?id=52586)，一个微软发布的开源库，可以很简单的写出hook。
- VS2017
- IDA

首先使用IDA分析一下CreatPorcess创建telnet成功后，telnet马上退出的原因。使用IDA找到main函数，F5查看C语言代码。

## 分析出错原因

```c
void __cdecl __noreturn wmain(signed int a1, int a2)
{
  HANDLE v2; // eax
  HANDLE v3; // eax
  int v4; // eax
  MSG Msg; // [esp+Ch] [ebp-24h]
  DWORD ThreadId; // [esp+28h] [ebp-8h]
  DWORD pdwValue; // [esp+2Ch] [ebp-4h]

  pdwValue = 0;
  SetThreadPreferredUILanguages(256, 0, 0);
  HeapSetInformation(0, HeapEnableTerminationOnCorruption, 0, 0);
  _setlocale(0, byte_DE1FB2);
  if ( a1 <= 1 )
    RegisterApplicationRestart(0, 0);
  else
    RegisterApplicationRestart((PCWSTR)(a2 + 4), 0);
  dword_E0B73C = 0;
  v2 = GetStdHandle(0xFFFFFFF6);
  if ( v2 != (HANDLE)-1 )
  {
    hConsoleInput = v2;
    v3 = GetStdHandle(0xFFFFFFF5);
    if ( v3 != (HANDLE)-1 )
    {
      hConsoleOutput = v3;
      g_hSessionConsoleBuffer = v3;
      if ( GetConsoleScreenBufferInfo(v3, (PCONSOLE_SCREEN_BUFFER_INFO)&ConsoleScreenBufferInfo) )
      {
        word_E0B7A8 = 7;
        word_E0B7B8 = 7;
        byte_E0B7B6 = 32;
        if ( SLGetWindowsInformationDWORD(L"Telnet-Client-EnableTelnetClient", &pdwValue) >= 0 && pdwValue )
        {
          v4 = FInitApplication(a1, (WCHAR *)a2, (DWORD)gwi);
          if ( !v4 )
          {
            g_hControlHandlerEvent = CreateEventW(0, 0, 0, 0);
            g_hAsyncGetHostByNameEvent = CreateEventW(0, 1, 0, 0);
            g_hCaptureConsoleEvent = CreateEventW(0, 1, 1, 0);
            hObject = CreateThread(0, 0, DoTelnetCommands, gwi, 0, &ThreadId);
            while ( GetMessageW(&Msg, hWnd, 0, 0) )
            {
              TranslateMessage(&Msg);
              DispatchMessageW(&Msg);
            }
            g_exitClient = 1;
            SetConsoleCtrlHandler(ControlHandler, 0);
            DoProcessCleanup();
            SetUserSettings(&ui);
            ExitProcess(0);
          }
        }
        else
        {
          v4 = GetLastError();
        }
        _exit(v4);
      }
    }
  }
  _exit(-1);
}
```

首先是一些初始化工作，然后调用`GetStdHandle`获得hConsoleInput,hConsoleOutput并判断其是否等于-1，等于则直接exit(-1)。我们传入的pipe的HANDLE是不等于-1的，因此不会在这里退出。

然后可以看见这条语句

```c
if ( GetConsoleScreenBufferInfo(v3, (PCONSOLE_SCREEN_BUFFER_INFO)&ConsoleScreenBufferInfo))
```

这里v3就是刚刚获得的hConsoleOutput，由于我们传入的是pipe的HANDLE,而这里需要的是console screen buffer的HANDLE,所以铁定这里会返回false。因此程序就在这里退出了。

## 寻找解决方案

进一步使用IDA分析发现，telnet中输入输出都是直接使用的ReadConsole,WriteConsole，需要的是consoleInput和consoleScreenBuffer的HANDLE,而我们传入的匿名管道的HANDLE是行不通的。

- 思路1：既然telnet只认可标准的console的HANDLE,那就传标准的console的HANDLE就好了。

可在自己的程序里调用GetStdHandle将获得的HANDLE传入telnet，测试后发现这样不能达到像匿名管道那样的效果，只能用键盘输入命令，且结果会直接显示到控制台，程序无法得到结果。后来又尝试使用CreatFile函数创建新的console的HANDLE再传入，测试后发现依然不行，甚至会出现非常奇怪的现象。

- 思路2 传入pipe的HANDLE， 在telnet的初始时使用hook调用CreateFile创建console HANDLE供telnet使用。

由于hConsoleOutput和hConsoleInput都是全局变量，很方便直接读写。并且telnet一共调用了两次GetStdHandle,分别用于初始化hConsoleInput和hConsoleOutput。因此我们可以在初始化时(钩子函数中)使用CreatFile创建两个标准console HABNDLE初始化hConsoleOutput和hConsoleInput。当需要pipe的HANDLE时，就直接GetStdHandle就行了。

## 关闭重定位

telnetk开启了重定位，先关闭重定位。使用PEVIEW查看IMAGE_OPTIONAL_HEADER中的DLL Characteristics项，使用ultraedit定位到0x0136,将 40 81 修改为00 81。

```asm
pFile       Data        Descripiton
00000136    8140        DLL Characteristics
            0040        IMAGE_DLLCHARACTERISTICS_DYNAMIC_BASE
            0100        IMAGE_DLLCHARACTERISTICS_NT_COMPAT
            8000        IMAGE_DLLCHARACTERISTICS_TERMINAL_SERVER_AWARE
```

## 编写初始化钩子函数

首先可以通过IDA分析找出hConsoleOutput和hConsoleInput的地址。(由于我在做这一步时并没有关闭重定位，所以我是先拿到程序加载基址，再加上RVA来得到正确地址的) 然后将通过CreateFile返回的HANLDE赋给hConsoleOutput和hConsoleInput。
如果仅仅这样telnet还是会直接退出，分析telnet main函数中`if ( GetConsoleScreenBufferInfo(v3, (PCONSOLE_SCREEN_BUFFER_INFO)&ConsoleScreenBufferInfo) )`这条语句对应的汇编代码。

```asm
.text:00DECFF9 ; 22:     v3 = GetStdHandle(0xFFFFFFF5);
.text:00DECFF9                 call    ebx ; GetStdHandle(x) ; GetStdHandle(x)
.text:00DECFFB ; 23:     if ( v3 != (HANDLE)-1 )
.text:00DECFFB                 cmp     eax, edi
.text:00DECFFD                 jz      short loc_DECFEB ; exit point
.text:00DECFFF ; 27:       if ( GetConsoleScreenBufferInfo(v3, (PCONSOLE_SCREEN_BUFFER_INFO)&ConsoleScreenBufferInfo) )
.text:00DECFFF                 push    offset ConsoleScreenBufferInfo ; lpConsoleScreenBufferInfo
.text:00DED004                 push    eax             ; hConsoleOutput
.text:00DED005 ; 25:       hConsoleOutput = v3;
.text:00DED005                 mov     hConsoleOutput, eax
.text:00DED00A ; 26:       g_hSessionConsoleBuffer = v3;
.text:00DED00A                 mov     _g_hSessionConsoleBuffer, eax
.text:00DED00F                 call    ds:__imp__GetConsoleScreenBufferInfo@8 ; GetConsoleScreenBufferInfo(x,x)
```

GetConsoleScreenBufferInfo参数中的v3直接使用GetStdHandle的返回值,在汇编中表现为`push eax`语句。因此我们需要hookGetConsoleScreenBufferInfo函数，将第一个参数改为hConsoleOutput。
另外`GetStdHandle(STD_OUTPUT_HANDLE)`的返回值还赋给了g_hSessionConsoleBuffer变量，因此需要在初始化hConsoleOutput和hConsoleInput一并把g_hSessionConsoleBuffer初始化了。

```c
BOOL WINAPI newHeapSetInformation(_In_opt_ HANDLE HeapHandle, _In_ HEAP_INFORMATION_CLASS HeapInformationClass, _In_reads_bytes_opt_(HeapInformationLength) PVOID HeapInformation, _In_ SIZE_T HeapInformationLength)
{
  char16_t* basename;

  //这里最初是没有关闭重定位的，下面汇编是拿到随机加载的基址，并计算出正确的iHandle，oHandle的地址(基址+RVA)。
  _asm
  {
    push eax;
    push ecx;
    mov eax, fs:[30h];
    mov eax, [eax + 0ch];
    mov eax, [eax + 0ch];
    mov ecx, [eax + 0x18]; //ecx = BaseAddr
    add hConsoleInput, ecx;
    add hConsoleOutput, ecx;
    add _g_hSessionConsoleBuffer, ecx;
    mov ecx, [eax + 0x30];
    mov basename, ecx;
    pop ecx;
    pop eax;
  }

  //创建标准console HANDLE并赋给telnet的全局变量
  SECURITY_ATTRIBUTES sa = { 0 };
  sa.nLength = sizeof(sa);
  sa.lpSecurityDescriptor = NULL;
  sa.bInheritHandle = true;
  *(HANDLE*)hConsoleOutput = CreateFile(L"CONIN$", GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ | FILE_SHARE_WRITE, &sa, OPEN_EXISTING, NULL, NULL);
  *(HANDLE*)hConsoleInput = CreateFile(L"CONOUT$", GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ | FILE_SHARE_WRITE, &sa, OPEN_EXISTING, NULL, NULL);
  *(HANDLE*)_g_hSessionConsoleBuffer = *(HANDLE*)hConsoleInput;

  //调用原来的函数，完成初始化
  return oldHeapSetInformation(HeapHandle, HeapInformationClass, HeapInformation, HeapInformationLength);
}
```

```c
BOOL WINAPI newConsoleScreenBufferInfo(_In_ HANDLE hConsoleOutput, _Out_ PCONSOLE_SCREEN_BUFFER_INFO lpConsoleScreenBufferInfo)
{
  //修改传入参数
  return oldGetConsoleScreenBufferInfo(*(HANDLE*)_g_hSessionConsoleBuffer, lpConsoleScreenBufferInfo);
}
```

由于已经在初始化函数中为hConsoleInput，hConsoleOutput和g_hSessionConsoleBuffer赋值了，所以需要nop掉这三个变量在main函数中的复制语句。

## 编写输入输出钩子函数

首先需要找到输入输出对应的函数，我是直接查看telnet导入表中从kernel32.dll导入了哪些函数，猜测可能的相关函数。经测试在中文环境下输出函数为`WriteConsoleW`, 输入函数为`ReadConsoleW`。用对pipe的读写，替代原本的函数。

```c
BOOL WINAPI newWriteConsoleW(_In_ HANDLE hConsoleOutput, _In_reads_(nNumberOfCharsToWrite) CONST VOID * lpBuffer, _In_ DWORD nNumberOfCharsToWrite, _Out_opt_ LPDWORD lpNumberOfCharsWritten, _Reserved_ LPVOID lpReserved)
{
  //write output to pipe
  DWORD writed;
  WriteFile(GetStdHandle(STD_OUTPUT_HANDLE), lpBuffer, nNumberOfCharsToWrite * 2, &writed, NULL);
  return true;
}
```

```c
BOOL APIENTRY DllMain(HMODULE hModule,DWORD  ul_reason_for_call,LPVOID lpReserved)
{
  init();
  if (ul_reason_for_call == DLL_PROCESS_ATTACH)
    InstallHook();
  else if (ul_reason_for_call == DLL_PROCESS_DETACH)
    UnInstallHook();
  return true;
}
```