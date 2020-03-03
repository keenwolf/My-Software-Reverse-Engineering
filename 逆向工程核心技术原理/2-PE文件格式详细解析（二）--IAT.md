# PE文件格式详细解析（二）--IAT

**IAT，导入地址表（Import Address Table），保存了与windows操作系统核心进程、内存、DLL结构等相关的信息。只要了理解了IAT，就掌握了Windows操作系统的根基。IAT是一种表格，用来记录程序正在使用哪些库中的哪些函数。**

## 一、DLL

DLL，动态链接库（Dynamic Linked Library）

### 1. 来源

   在16位的DOS环境中，不存在DLL的概念，例如在C中使用printf函数时，编译器会先从C库中读取相应函数的二进制代码，然后插入到应用程序中。但是Windows支持多任务，采用这种包含库的方式会没有效率，因为如果每个程序在运行时都将Windows库中的函数加载进来，将造成严重的内存浪费，因此引入了DLL的概念。

### 2. 设计理念

   1. 不把函数库包含进应用程序中，单独组成DLL文件，在需要使用时再进行调用。
   2. 使用内存映射技术将加载后的DLL代码、资源在多个进程中实现共享。
   3. 在对函数库进行更新时，只更新DLL文件即可。

### 3. 加载方式

   DLL加载方式有两种：**显式链接（Explicit Linking）** 和 **隐式链接（Implicit Linking）**

- 显示链接：程序在使用DLL时进行加载，使用完毕后释放内存
- 隐式链接：程序在开始时即一同加载DLL，程序终止时再释放占用的内存

   **IAT提供的机制与DLL的隐式链接有关。**

## 二、DLL调用的简单理解

在OD中查看程序的反汇编代码如下所示:

![iat](https://i.imgur.com/4ZZual0.png)

在调用ThunRTMain()函数时，并非是直接调用函数，而是通过获取0x00405164地址处的值-0x7400A1B0，该值是加载到待分析应用程序进程内存中的ThunRTMain()函数的地址。  

需要注意的是，此处之所以编译器不直接进行jmp 7400A1B0主要是因为以下两点：

- DLL版本不同，由于操作系统的版本存在差异，DLL文件版本也会存在差异
- DLL重定位，DLL文件的ImageBase一般为0x10000000，如果应用程序同时有两个DLL文件需要加载--a.dll和b.dll，在运行时a.dll首先加载进内存，占到了0x10000000，此时b.dll如果再加载到0x10000000，就会发生冲突，所以需要加载到其他的空白内存空间处。

## 三、IMAGE_IMPORT_DESCRIPTOR结构体

## 1. 结构介绍

   该结构体中记录着PE文件要导入哪些库文件，因为在执行一个程序时需要导入多个库，所以导入了多少库，就会存在多少IMAGE_IMPORT_DESCRIPTOR结构体，这些结构体组成数组，数组最后以NULL结构体结束。部分重要成员如下所示：

| 成员          | 含义                                        |
| ------------- | ------------------------------------------- |
| OriginalThunk | INT的地址（RVA），4字节长整型数组，NULL结束 |
| Name          | 库名称字符串的地址（RVA）                   |
| FirstThunk    | IAT的地址（RVA），4字节长整型数组，NULL结束 |

下图描述了notepad.exe之kernel32.dll的IMAGE_IMPORT_DESCRIPTOR结构：

![iat1](https://i.imgur.com/t65HBV4.png)



## 2. PE装载器把导入函数输入至IAT的顺序

1. 读取IID的Name成员，获取库名称字符串（eg：kernel32.dll）

2. 装载相应库：

   LoadLibrary("kernel32.dll")

3. 读取IID的OriginalFirstThunk成员，获取INT地址

4. 逐一读取INT中数组的值，获取相应IMAGE_IMPORT_BY_NAME地址（RVA）

5. 使用IMAGE_IMPORT_BY_NAME的Hint（ordinal）或Name项，获取相应函数的起始地址：

   GetProcAddress("GetCurrentThreadld")

6. 读取IID的FirstThunk（IAT）成员，获得IAT地址

7. 将上面获得的函数地址输入相应IAT数组值

8. 重复以上步骤4～7，知道INT结束（遇到NULL）

## 四、总结

IAT是在学习PE文件格式中重要的一部分，也是比较难的一部分，需要仔细学习，一定要熟练掌握。建议根据实际的PE文件结合前面的分析步骤，亲自动手多加分析，不断熟悉分析流程。



