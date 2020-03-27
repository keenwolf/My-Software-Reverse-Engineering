# Windows消息钩取

## 一、钩子和消息钩子

钩子，英文Hook，泛指偷看或截取信息时所用的手段或工具。

Windows操作系统向用户提供GUI，它是以事件驱动（Event Driven）方式工作。事件发生后，OS将事先定义好的消息发送给相应的应用程序，应用程序分析收到的消息后执行相应动作。以敲击键盘为例，

常规Windows消息流：

1. 发生键盘输入事件，WM_KEYDOWN消息被添加到OS消息队列；
2. OS判断哪个应用程序发生了事件，从OS消息队列中取出消息，添加到相应应用程序的app消息队列；
3. 应用程序监视自身的消息队列，发现新添加的WM_KEYDOWN消息，调用相应的事件处理程序进行处理。

附带钩子的信息流：

1. 发生键盘输入事件，WM_KEYDOWN消息被添加到OS消息队列；
2. OS判断哪个应用程序发生了事件，从OS消息队列中取出消息，发送给应用程序；
3. 钩子程序截取信息，对消息采取一定的动作（因钩子目的而定）；
4. 如钩子程序不拦截消息，消息最终传输给应用程序，此时的消息可能经过了钩子程序的修改。

## 二、SetWindowsHookEx()

这是一个实现消息钩子的API，其定义如下：

```
HHOOK SetWindowsHookEx(
	int idHook,						// hook type
	HOOKpROC lpfn,				// hook procedure
	HINSTANCE hMod,				//hook procedure所属的DLL句柄
	DWORD dwThreadId			//需要挂钩的线程ID，为0时表示为全局钩子（Global Hook）
);
```

hook proceduce是由操作系统调用的回调函数；安装消息钩子时，钩子过程需要存在于某个DLL内部，且该DLL的示例句柄即为hMod。

使用SetWindowsHookEx()设置好钩子后，在某个进程中生成指定消息时，OS就会将相关的DLL文件强制注入（injection）相应进程，然后调用注册的钩子程序。

## 三、键盘消息钩取

以下以书上例子进行练习，首先过程原理图如下：

![21-1](https://i.imgur.com/KIzGnzh.png)

KeyHook.dll文件是一个含有钩子过程（KeyboardProc）的DLL文件，HookMain.exe是最先加载KeyHook.dll并安装键盘钩子的程序。HookMain.exe加载KeyHook.dll后使用SetWindowsHookEx()安装键盘钩子；若其他进程（如图中所示）发生键盘输入事件，OS就会强制将KeyHook.dll加载到像一个进程的内存，然后调用KeyboardProc()函数。

**实验：HookMain.exe**

关于实验操作部分建议跟随书上走一遍流程，体验Hook的魅力。

## 四、源代码分析

### 1. HookMain.cpp

HookMain程序的主要源代码如下所示：

```C++
#include "stdio.h"
#include "conio.h"
#include "windows.h"

#define	DEF_DLL_NAME		"KeyHook.dll"
#define	DEF_HOOKSTART		"HookStart"
#define	DEF_HOOKSTOP		"HookStop"

typedef void (*PFN_HOOKSTART)();
typedef void (*PFN_HOOKSTOP)();

void main()
{
	HMODULE	hDll = NULL;
	PFN_HOOKSTART	HookStart = NULL;
	PFN_HOOKSTOP	HookStop = NULL;
	char	ch = 0;

  // 加载KeyHook.dll
	hDll = LoadLibraryA(DEF_DLL_NAME);
    if( hDll == NULL )
    {
        printf("LoadLibrary(%s) failed!!! [%d]", DEF_DLL_NAME, GetLastError());
        return;
    }

  // 获取导出函数地址
	HookStart = (PFN_HOOKSTART)GetProcAddress(hDll, DEF_HOOKSTART);
	HookStop = (PFN_HOOKSTOP)GetProcAddress(hDll, DEF_HOOKSTOP);

  // 开始钩取
	HookStart();

  // 等待，直到用户输入“q”
	printf("press 'q' to quit!\n");
	while( _getch() != 'q' )	;

  // 终止钩子
	HookStop();
	
  // 卸载KeyHook.dll
	FreeLibrary(hDll);
}
```

### 2. KeyHook.dll

KeyHook.dll源代码：

```c++
#include "stdio.h"
#include "windows.h"

#define DEF_PROCESS_NAME		"notepad.exe"

HINSTANCE g_hInstance = NULL;
HHOOK g_hHook = NULL;
HWND g_hWnd = NULL;

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD dwReason, LPVOID lpvReserved)
{
	switch( dwReason )
	{
        case DLL_PROCESS_ATTACH:
			g_hInstance = hinstDLL;
			break;

        case DLL_PROCESS_DETACH:
			break;	
	}

	return TRUE;
}

LRESULT CALLBACK KeyboardProc(int nCode, WPARAM wParam, LPARAM lParam)
{
	char szPath[MAX_PATH] = {0,};
	char *p = NULL;

	if( nCode >= 0 )
	{
		// bit 31 : 0 => press, 1 => release
		if( !(lParam & 0x80000000) )	//释放键盘按键时
		{
			GetModuleFileNameA(NULL, szPath, MAX_PATH);
			p = strrchr(szPath, '\\');

      //比较当前进程名称是否为notepad.exe，成立则消息不传递给应用程
			if( !_stricmp(p + 1, DEF_PROCESS_NAME) )
				return 1;
		}
	}

  //如果不是notepad.exe，则调用CallNextHookEx()函数，将消息传递给应用程序
	return CallNextHookEx(g_hHook, nCode, wParam, lParam);
}

#ifdef __cplusplus
extern "C" {
#endif
	__declspec(dllexport) void HookStart()
	{
		g_hHook = SetWindowsHookEx(WH_KEYBOARD, KeyboardProc, g_hInstance, 0);
	}

	__declspec(dllexport) void HookStop()
	{
		if( g_hHook )
		{
			UnhookWindowsHookEx(g_hHook);
			g_hHook = NULL;
		}
	}
#ifdef __cplusplus
}
#endif
```

总体上代码相对简单，调用导出函数HookStart()时，SetWindowsHookEx()函数就会将KetyboardProc()添加到键盘钩链。

### 3. 代码执行流程分析

安装好键盘钩子后，无论在哪个进程中，只要发生了键盘输入事件，OS就会强制将KeyHook.dll注入到进程中，加载了KeyHook.dll的进程，发生键盘事件时会首先调用执行KeyHook.KetyboardProc()。

KetyboardProc()函数中发生键盘输入事件时，会比较当前进程的名称与“notepad.exe”是否相同，相同返回1，终止KetyboardProc()函数，意味着截获并删除了消息，这样键盘消息就不会传递到notepad.exe程序的消息队列。



## 五、调试

使用OD打开HookMain.exe文件：

![21-2](https://i.imgur.com/qGGQTjV.png)

###1. 查找核心代码

我们关心的是核心的键盘钩取部分的代码，如何查找核心代码？

1. 逐步跟踪（除非迫不得已！）
2. 检索相关API
3. 检索相关字符串

我们已经知道程序的功能，会在控制台显示字符串“press ‘q’ to quit!”，所以先检查程序导入的字符串（Search for -All referencen text strings）：

![21-3](https://i.imgur.com/RtuvCod.png)

地址40104d处引用了要查找的字符串，双击跳转：

![21-4](https://i.imgur.com/7GpBjWg.png)

来到main函数处。

### 2. 调试main函数

在401000处下断，开始调试，了解main函数中主要的代码流。401006地址处调用LoadLibraryA(Keyhook.dll)，然后由40104b地址处的CALL EBX指令调用KeyHook.HookStart()函数。跟进查看：

![21-5](https://i.imgur.com/0CcNyhq.png)

这里的代码是被加载到HookMain.exe进程中的KeyHook.dll的HookStart()函数，第一句就是调用SetWindowsHookExW()函数，在进行参数入栈操作后，我们可以在栈中看到函数的4个参数值。

## 参考

《逆向工程核心原理》

