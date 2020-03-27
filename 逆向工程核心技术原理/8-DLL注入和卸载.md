# DLL注入和卸载

## 一、DLL注入

DLL注入：向运行中的其他进程强制插入特定的DLL文件，主要是命令其他进程自行调用LoadLibrary() API，加载用户指定的DLL文件。

DLL注入与一般DLL加载的主要区别是加载的目标进程是其自身或其他进程。

### 1. DLL

DLL（Dynamic Linked Library，动态链接库），DLL被加载到进程后会自动运行DllMain函数，用户可以把想要执行的额代码放到DllMain函数，每当加载DLL时，添加的代码就会自动得到执行。利用该特性可以修复程序BUG，或向程序添加新功能。

```c++
// DllMain()函数 
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD dwReason, LPVOID lpvRserved)
{
  switch(dwReason)
  {
    case DLL_PROCESS_ATTACH:
      // 添加想要执行的代码
      break;
      
    case DLL_THREAD_ATTACH:
      break;
      
    case DLL_THREAD_DETACH:
      break;
      
    case DLL_PROCESS_DETACH:
      break;
  }
  
  return TRUE;
}
```

### 2. DLL注入实例

使用LoadLibrary()加载某个DLL时，该DLL中的DllMain()函数会被调用执行。DLL注入的原理就是从外部促使目标进程调用LoadLibrary() API。

1. 改善功能与修复BUG
2. 消息钩取--Windows 默认提供的消息钩取功能本质上应用的就是一种DLL注入技术
3. API钩取--先创建DLL形态的钩取函数，然后注入要钩取的目标进程，主要是应用了被注入的DLL拥有目标进程内存访问权限这一特性
4. 其他应用程序--监视、管理PC用户的应用程序
5. 恶意代码--非法注入，进行代码隐藏

### 3. DLL注入的实现方法

#### 1. 创建远程线程（CreateRemoteThread() API）

此处主要记录一下书上的源码分析，操作部分请自行实践。

```c++
// myhack.cpp
#include "windows.h"
#include "tchar.h"

#pragma comment(lib, "urlmon.lib")

#define DEF_URL     	(L"http://www.naver.com/index.html")
#define DEF_FILE_NAME   (L"index.html")

HMODULE g_hMod = NULL;

DWORD WINAPI ThreadProc(LPVOID lParam)
{
    TCHAR szPath[_MAX_PATH] = {0,};

    if( !GetModuleFileName( g_hMod, szPath, MAX_PATH ) )
        return FALSE;
	
    TCHAR *p = _tcsrchr( szPath, '\\' );
    if( !p )
        return FALSE;

    _tcscpy_s(p+1, _MAX_PATH, DEF_FILE_NAME); //参数准备

    URLDownloadToFile(NULL, DEF_URL, szPath, 0, NULL); //调用函数进行URL下载

    return 0;
}

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
    HANDLE hThread = NULL;

    g_hMod = (HMODULE)hinstDLL;

    switch( fdwReason )
    {
    case DLL_PROCESS_ATTACH : 
        OutputDebugString(L"<myhack.dll> Injection!!!");
        
        //创建远程线程进行download
        hThread = CreateThread(NULL, 0, ThreadProc, NULL, 0, NULL);
        
        // 需要注意，切记随手关闭句柄，保持好习惯
        CloseHandle(hThread);
        break;
    }

    return TRUE;
}
```

```C++
// InjectDll.cpp
#include "windows.h"
#include "tchar.h"

BOOL InjectDll(DWORD dwPID, LPCTSTR szDllPath)
{
    HANDLE hProcess = NULL, hThread = NULL;
    HMODULE hMod = NULL;
    LPVOID pRemoteBuf = NULL;
  	
  	//确定路径需要占用的缓冲区大小
    DWORD dwBufSize = (DWORD)(_tcslen(szDllPath) + 1) * sizeof(TCHAR); 
    LPTHREAD_START_ROUTINE pThreadProc;

    // #1. 使用OpenProcess函数获取目标进程句柄（PROCESS_ALL_ACCESS权限）
    if ( !(hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPID)) )
    {
        _tprintf(L"OpenProcess(%d) failed!!! [%d]\n", dwPID, GetLastError());
        return FALSE;
    }

    // #2. 使用VirtualAllocEx函数在目标进程中分配内存，大小为szDllName
  	// VirtualAllocEx函数返回的是hProcess指向的目标进程的分配所得缓冲区的内存地址
    pRemoteBuf = VirtualAllocEx(hProcess, NULL, dwBufSize, MEM_COMMIT, PAGE_READWRITE);

    // #3.  将myhack.dll路径 ("c:\\myhack.dll")写入目标进程中分配到的内存
    WriteProcessMemory(hProcess, pRemoteBuf, (LPVOID)szDllPath, dwBufSize, NULL);

    // #4. 获取LoadLibraryA() API的地址
  	// 这里主要利用来了kernel32.dll文件在每个进程中的加载地址都相同这一特点，所以不管是获取加载到	
  	// InjectDll.exe还是notepad.exe进程的kernel32.dll中的LoadLibraryW函数的地址都是一样的。这里的加载地
  	// 址相同指的是在同一次系统运行中，如果再次启动系统kernel32.dll的加载地址会变，但是每个进程的
  	// kernerl32.dll的加载地址还是一样的。
  	hMod = GetModuleHandle(L"kernel32.dll");
    pThreadProc = (LPTHREAD_START_ROUTINE)GetProcAddress(hMod, "LoadLibraryW");
	
    // #5. 在目标进程notepad.exe中运行远程线程
  	// pThreadProc = notepad.exe进程内存中的LoadLibraryW()地址
  	// pRemoteBuf = notepad.exe进程内存中待加载注入dll的路径字符串的地址
    hThread = CreateRemoteThread(hProcess, NULL, 0, pThreadProc, pRemoteBuf, 0, NULL);
    WaitForSingleObject(hThread, INFINITE);	
		
  	//同样，记得关闭句柄
    CloseHandle(hThread);
    CloseHandle(hProcess);

    return TRUE;
}

int _tmain(int argc, TCHAR *argv[])
{
    if( argc != 3)
    {
        _tprintf(L"USAGE : %s <pid> <dll_path>\n", argv[0]);
        return 1;
    }

    // change privilege
    if( !SetPrivilege(SE_DEBUG_NAME, TRUE) )
        return 1;

    // inject dll
    if( InjectDll((DWORD)_tstol(argv[1]), argv[2]) )
        _tprintf(L"InjectDll(\"%s\") success!!!\n", argv[2]);
    else
        _tprintf(L"InjectDll(\"%s\") failed!!!\n", argv[2]);

    return 0;
}
```

main()函数主要检查输入程序的参数，然后调用InjectDll函数。InjectDll函数是实施DLL注入的核心函数，功能是命令目标进程自行调用LoadLibrary API。



重点介绍一下CreateRemoteThread()函数，该函数在进行DLL注入时会经常用到，其函数原型如下：

```c++
CreateRemoteThread()
HANDLE WINAPI CreateRemoteThread(
  __in HANDLE    hProcess,	//目标进程句柄
  __in LPSECURITY_ATTRIBUTES   lpThreadAttributes,
  __in SIZE_T dwStackSize,
  __in LPTHREAD_START_ROUTNE	lpStartAddress,	//线程函数地址
  __in LPVOID dwCreationFlags,	//线程参数地址
  __out LPDOWRD lpThreadId
);
```

#### 2. AppInit_DLLs

第二种方法是操作注册表，Windows的注册表中默认提供了AppInit_DLLs与LoadAppInit_DLLs两个注册表项，只要将要注入DLL的路径字符串写入AppInit_DLLs项目，并在LoadAppInit_DLLs中设置值为1，重启时，系统就会将指定的DLL注入到所有运行进程中。主要原理是User32.dll被加载到进程时，会读取AppInit_DLLs注册表项，若值为1，就调用LoadLibrary()函数加载用户DLL。所以严格来说，是将注入DLL加载到使用user32.dll的进程中。

```c++
// myhack2.cpp
// 主要作用是以隐藏模式运行IE，连接到指定网站

#include "windows.h"
#include "tchar.h"

#define DEF_CMD  L"c:\\Program Files\\Internet Explorer\\iexplore.exe" 
#define DEF_ADDR L"http://www.naver.com"
#define DEF_DST_PROC L"notepad.exe"

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
    TCHAR szCmd[MAX_PATH]  = {0,};
    TCHAR szPath[MAX_PATH] = {0,};
    TCHAR *p = NULL;
    STARTUPINFO si = {0,};
    PROCESS_INFORMATION pi = {0,};

    si.cb = sizeof(STARTUPINFO);
    si.dwFlags = STARTF_USESHOWWINDOW;
    si.wShowWindow = SW_HIDE;

    switch( fdwReason )
    {
    case DLL_PROCESS_ATTACH : 
        if( !GetModuleFileName( NULL, szPath, MAX_PATH ) )
            break;
   
        if( !(p = _tcsrchr(szPath, '\\')) )
            break;

        if( _tcsicmp(p+1, DEF_DST_PROC) )
            break;

        wsprintf(szCmd, L"%s %s", DEF_CMD, DEF_ADDR);
        if( !CreateProcess(NULL, (LPTSTR)(LPCTSTR)szCmd, 
                            NULL, NULL, FALSE, 
                            NORMAL_PRIORITY_CLASS, 
                            NULL, NULL, &si, &pi) )
            break;

        if( pi.hProcess != NULL )
            CloseHandle(pi.hProcess);

        break;
    }
   
    return TRUE;
}
```

将上述dll文件复制到某个位置，修改注册表项`HKEY_LOCAL_MACHINE\SOFTWARE\Microdoft\Windows NT\CurrentVersion\Windows`,将AppInit_DLLs项的值修改为待注入DLL的绝对路径，然后修改LoadAppInit_DLLs注册表项的值为1，重启，运行notepad.exe，就会看到DLL已经被注入。

#### 3. 使用SetWindowsHookEx()函数

第三个方法就是消息钩取，使用SetWindowsHookEx安装钩子，将指定DLL强制注入进程。

## 二、DLL卸载

DLL卸载原理：驱使目标进程调用FreeLibrary()函数，即将FreeLibrary()函数的地址传递给CreateRemoteThread()函数的lpStartAddress参数，并把待卸载的DLL句柄传递给lpParameter参数。

需要注意的一点是：引用计数问题。调用一次FreeLibrary()函数，引用计数就会-1。引用计数表示的是内核对象被使用的次数。

```c++
// EjectDll.exe

#include "windows.h"
#include "tlhelp32.h"
#include "tchar.h"

#define DEF_PROC_NAME	(L"notepad.exe")
#define DEF_DLL_NAME	(L"myhack.dll")

DWORD FindProcessID(LPCTSTR szProcessName)
{
    DWORD dwPID = 0xFFFFFFFF;
    HANDLE hSnapShot = INVALID_HANDLE_VALUE;
    PROCESSENTRY32 pe;

    // 获取系统快照
    pe.dwSize = sizeof( PROCESSENTRY32 );
    hSnapShot = CreateToolhelp32Snapshot( TH32CS_SNAPALL, NULL );

    // 查找进程
    Process32First(hSnapShot, &pe);
    do
    {
        if(!_tcsicmp(szProcessName, (LPCTSTR)pe.szExeFile))
        {
            dwPID = pe.th32ProcessID;
            break;
        }
    }
    while(Process32Next(hSnapShot, &pe));

    CloseHandle(hSnapShot);

    return dwPID;
}

BOOL SetPrivilege(LPCTSTR lpszPrivilege, BOOL bEnablePrivilege) 
{
    TOKEN_PRIVILEGES tp;
    HANDLE hToken;
    LUID luid;

    if( !OpenProcessToken(GetCurrentProcess(),
                          TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY, 
			              &hToken) )
    {
        _tprintf(L"OpenProcessToken error: %u\n", GetLastError());
        return FALSE;
    }

    if( !LookupPrivilegeValue(NULL,           // lookup privilege on local system
                              lpszPrivilege,  // privilege to lookup 
                              &luid) )        // receives LUID of privilege
    {
        _tprintf(L"LookupPrivilegeValue error: %u\n", GetLastError() ); 
        return FALSE; 
    }

    tp.PrivilegeCount = 1;
    tp.Privileges[0].Luid = luid;
    if( bEnablePrivilege )
        tp.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;
    else
        tp.Privileges[0].Attributes = 0;

    // Enable the privilege or disable all privileges.
    if( !AdjustTokenPrivileges(hToken, 
                               FALSE, 
                               &tp, 
                               sizeof(TOKEN_PRIVILEGES), 
                               (PTOKEN_PRIVILEGES) NULL, 
                               (PDWORD) NULL) )
    { 
        _tprintf(L"AdjustTokenPrivileges error: %u\n", GetLastError() ); 
        return FALSE; 
    } 

    if( GetLastError() == ERROR_NOT_ALL_ASSIGNED )
    {
        _tprintf(L"The token does not have the specified privilege. \n");
        return FALSE;
    } 

    return TRUE;
}

BOOL EjectDll(DWORD dwPID, LPCTSTR szDllName)
{
    BOOL bMore = FALSE, bFound = FALSE;
    HANDLE hSnapshot, hProcess, hThread;
    HMODULE hModule = NULL;
    MODULEENTRY32 me = { sizeof(me) };
    LPTHREAD_START_ROUTINE pThreadProc;

    // dwPID = notepad 进程ID
    // 使用TH32CS_SNAPMODULE参数，获取加载到notepad进程的DLL名称
    hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, dwPID);

    bMore = Module32First(hSnapshot, &me);
    for( ; bMore ; bMore = Module32Next(hSnapshot, &me) )
    {
        if( !_tcsicmp((LPCTSTR)me.szModule, szDllName) || 
            !_tcsicmp((LPCTSTR)me.szExePath, szDllName) )
        {
            bFound = TRUE;
            break;
        }
    }

    if( !bFound )
    {
        CloseHandle(hSnapshot);
        return FALSE;
    }

    if ( !(hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, dwPID)) )
    {
        _tprintf(L"OpenProcess(%d) failed!!! [%d]\n", dwPID, GetLastError());
        return FALSE;
    }

    hModule = GetModuleHandle(L"kernel32.dll");
  	// 获取FreeLibrary函数加载地址，并使用CreateRemoteThread进行调用
    pThreadProc = (LPTHREAD_START_ROUTINE)GetProcAddress(hModule, "FreeLibrary");
    hThread = CreateRemoteThread(hProcess, NULL, 0, 
                                 pThreadProc, me.modBaseAddr, 
                                 0, NULL);
    WaitForSingleObject(hThread, INFINITE);	

    CloseHandle(hThread);
    CloseHandle(hProcess);
    CloseHandle(hSnapshot);

    return TRUE;
}

int _tmain(int argc, TCHAR* argv[])
{
    DWORD dwPID = 0xFFFFFFFF;
 
    // 查找process
    dwPID = FindProcessID(DEF_PROC_NAME);
    if( dwPID == 0xFFFFFFFF )
    {
        _tprintf(L"There is no <%s> process!\n", DEF_PROC_NAME);
        return 1;
    }

    _tprintf(L"PID of \"%s\" is %d\n", DEF_PROC_NAME, dwPID);

    // 更改 privilege
    if( !SetPrivilege(SE_DEBUG_NAME, TRUE) )
        return 1;

    // 注入 dll
    if( EjectDll(dwPID, DEF_DLL_NAME) )
        _tprintf(L"EjectDll(%d, \"%s\") success!!!\n", dwPID, DEF_DLL_NAME);
    else
        _tprintf(L"EjectDll(%d, \"%s\") failed!!!\n", dwPID, DEF_DLL_NAME);

    return 0;
}

```

CreateToolhelp32Snapshot()函数主要用来获取加载到进程的模块信息，将获取的hSnapshot句柄传递给Module32First()/Module32Next()函数后，即可设置与MODULEENTRY32结构相关的模块信息，以下为该结构的详细定义：

```C++
typedef sturc tagMODULEENTRY32
{
	DWORD dwSize;
  DWORD th32ModuleID;		// 该模块
  DWORD th32ProcessID;	// 模块拥有的进程
  DWORD GlbcntUsage;		//模块中的global usage计数
  DWORD ProcessUsage;		
  BYTE * modBaseAddr;		//在进程的上下文中的模块的基地址
  DWORD modBaseSize;		// 在modBaseAddr开始位置的模块的大小（字节为单位）
  HMODULE hModule;
  char szModule[MAX_MODULE_NAME32+1];	//DLL名称
  char szExePath[MAX_PATH];
}MODULEENTRY32;
```



## 参考

《逆向工程核心原理》

