# PE文件格式详细解析（五）-- 调试UPX压缩的notepad程序

## 一、未经过UPX压缩的notepad的EP代码

首先看一下未经过UPX压缩的notepad的相关信息：

1. PEView查看基本结构信息：

   ![upx4](https://i.imgur.com/fY8Yn5t.png)

   RVA = 1000，且SizeOfRawData是有大小的。

3. OD查看EP代码：

   首先简单看一下汇编代码，程序在010073b2处调用kernel32.dll中的GetModuleHandleA()函数，然后可以得到程序的ImageBase，存放在EAX中：

   

   ![upx8](https://i.imgur.com/vufFLNd.png)

   

   然后，进行PE文件格式的验证，比较MZ和PE签名。

   ![upx7](https://i.imgur.com/ld5iEM5.png)

   以上代码可以简单记录一下，方便后续与经过UPX压缩的程序进行比较。

## 二、经过UPX压缩的notepad_upx.exe的EP代码

1. PEView查看下信息（上一节已经介绍过）：

   第一个图为第一个节区UPX0的信息，第二个图为第二个节区UPX1的信息。

   ![upx5](https://i.imgur.com/Vq7yMDR.png)

   ![upx6](https://i.imgur.com/oKq8RjB.png)

2. OD进行EP代码查看：

   ![upx9](https://i.imgur.com/Cj3XgN4.png)

   可以发现经过UPX压缩的EP代码发生了明显的改变，入口地址变为了01014410，该地址其实为第二个节区UPX1的末尾地址（使用PEView可以确认），实际压缩的源代码位于该地址的上方。

   然后我们看一下代码开始部分：

   ```asm
   01014410		60						pushad
   01014411		BE 00000101		mov esi, notepad_.01010000
   01014416		8DBE 0010FFFF	lea esi, dword ptr ds:[esi-0xf000]
   ```

   首先看第一句，pushad，其主要作用将eax～edi寄存器的值保存到栈中：

   

   ![upx10](https://i.imgur.com/RZEKfiQ.png)

   结合上面的图，发现在执行完pushad指令后，eax～edi的值确实都保存到了栈中。

   后面两句分别把第二个节区的起始地址（01010000）与第一个节区的起始地址（01001000）存放到esi与edi寄存器中。UPX文件第一节区仅存在于内存中，该处即是解压缩后保存源文件代码的地方。

   需要注意的是，在调试时同时设置esi与edi，大概率是发生了esi所指缓冲区到edi所指缓冲区的内存复制。此时从Source（esi）读取数据，解压缩后保存到Destination（edi）。

## 三、跟踪UPX文件

**掌握基本信息后，开始正式跟踪UPX文件，需要遵循的一个原则是，遇到循环（loop）时，先了解作用再跳出，然后决定是否需要再循环内部单步调试。**

备注：此处开始使用书上的例子，因为我个人的反汇编的代码会跟书上不一致，不建议新手使用。

### 1. 第一个循环

在EP代码处执行Animate Over（Ctrl+F8）命令，开始跟踪代码：

![upx11](https://i.imgur.com/1mAnhGN.png)

跟踪到这里后发现第一个关键循环，涉及到edi的反复变化，循环次数为36b，主要作用是从edx（01001000）中读取一个字节写入edi（01001001）。edi所指的地址即是第一个节区UPX0的起始地址（PEView已经验证过），仅存于内存中，数据全部被填充为NULL，主要是清空区域，防止有其他数据。这样的循环我们跳出即可，在010153e6处下断点，然后F9跳出。

### 2. 第二个循环

在断点处继续Animate Over跟踪代码，遇到下图的循环结构：

 ![upx12](https://i.imgur.com/3dXfJ3O.png)

该村换是正式的解压缩循环。

先从esi所指的第二个节区（UPX1）地址中依次读取数据，然后经过一系列运算解压缩后，将数据放入edi所指的第一个节区（UPX0）地址。关键指令解释：

```asm
0101534B   .  8807          mov byte ptr ds:[edi],al
0101534D   .  47            inc edi                                  ;  notepad_.0100136C
...
010153E0   .  8807          mov byte ptr ds:[edi],al
010153E2   .  47            inc edi                                  ;  notepad_.0100136C
...
010153F1   .  8907          mov dword ptr ds:[edi],eax
010153F3   .  83C7 04       add edi,0x4
* 解压缩后的数据放在AL（eax）中，edi指向第一个节区的地址
```

在01015402地址处下断，跳出循环（暂不考虑内部压缩过程）。在转储窗口查看解压缩后的代码：

![upx13](https://i.imgur.com/XvxJMgB.png)

### 3. 第三个循环

重新跟踪代码，遇到如下循环：

![upx14](https://i.imgur.com/bfkEWMI.png)

 这部分代码主要是恢复源代码的CALL/JMP指令（机器码：E8/E9）的destination地址。

到此为止，基本恢复了所有的压缩的源代码，最后设置下IAT即可成功。

### 4. 第四个循环

01015436处下断：

![upx15](https://i.imgur.com/gYkkmzn.png)

此处edi被设置为01014000，指向第二个节区（UPX1）区域，该区域中保存着原程调用的API函数名称的字符串。

![upx16](https://i.imgur.com/6pzePhH.png)

UPX在进行压缩时，会分析IAT，提取出原程序中调用的额API名称列表，形成api函数名称字符串。

使用这些API名称字符串调用01015467地址处的GetProcAddress()函数，获取API的起始地址，然后把API地址输入ebx寄存器所指的原程序的IAT区域，循环进行，直到完全恢复IAT。

然后，到01054bb的jmp指令处，跳转到OEP（原始EP）代码处：

![upx17](https://i.imgur.com/lA3D82p.png)

至此，UPX的解压缩全部完成，后续进行notepad.exe的正常执行。



## 五、快速查找UPX OEP的方法

### 1. 在POPAD指令后的JMP指令处设置断点

UPX压缩的特征之一是其EP代码被包含在PUSHAD/POPAD指令之间，并且在POPAD指令之后紧跟着的JMP指令会跳转到OEP代码处，所以可以在此处下断点，直接跳转到OEP地址处。

### 2. 在栈中设置硬件断点

本质上也是利用 PUSHAD/POPAD指令的特点。因为eax～edi的值依次被保存到栈中，不管中间做了什么操作，想要运行OEP的代码就需要从栈中读取这些寄存器的值来恢复程序的原始运行状态，所以我们只要设置硬件断点监视栈中寄存器的值的变化就可以快速定位到OEP。

F8执行完pushad后，在od的dump窗口进入栈地址：

![upx18](https://i.imgur.com/rahwARf.png)

然后选中下硬件读断点：

![upx19](https://i.imgur.com/XsUTvtL.png)

直接F9，你会发现很快就来到PUSHAD后的JMP指令处。

最后，补充硬件断点的几个知识：硬件断点是CPU支持的断点，最多设置4个；执行完指令后再停止。

