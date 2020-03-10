# PE文件格式详细解析（六）-- 基址重定位表（Base Relocation Table）

## 一、PE重定位

向进程的虚拟内存加载PE文件时，文件会被加载到PE头的ImageBase所指的地址处。如果是加载的DLL（SYS）文件，且在ImageBase位置处已经加载了DLL（SYS）文件，那么PE装载器就会将其加载到其他未被占用的空间。此时就会发生基址重定位。

**使用SDK或VC++创建PE文件，EXE默认的ImageBase为00400000，DLL默认的ImageBase为10000000，使用DDK创建的SYS文件默认的ImageBase为10000。**

创建好进程后，因为EXE文件会首先加载进内存，所以EXE文件中无需考虑基址重定位问题。但是需要考虑ASLR（地址随机化）。对于各OS的主要系统DLL，微软会根据不同版本分别赋予不同的ImageBase地址，例如同一系统的kernel32.dll和user32.dll等会被加载到自身固有的ImageBase，所以系统的DLL实际上也不会发生重定位问题。

## 二、PE重定位时发生了什么

以下以书上程序为例（书上是以exe文件举例，纯粹是举例，实际环境中基址重定位多发生在DLL文件中）。

1. 基本信息：

   如下图所示，其ImageBase为01000000

   ![16-1](https://i.imgur.com/gKZOCpO.png)

2. 使用OD运行，观察内存：

   下图是程序的EP代码部分，因为ASLR的原因，程序被加载到00270000处。

   ![16-2](https://i.imgur.com/3NpKJvE.png)

   从图中可以看出，红框内进程的内存地址是以硬编码的方式存在的，地址2710fc、271100是.text节区的IAT区域，地址27c0a4是.data节区的全局变量。因为ASLR的存在，每次在OD中重启程序，地址值就会随加载地址的不同而发生变化，这种使硬编码在程序中的内存地址随当前加载地址变化而改变的处理过程就是PE重定位。

   将以上两个图进行对比整理，数据如下表所示：

   | 文件（ImageBase：01000000） | 进程内存（加载地址：00270000） |
   | --------------------------- | ------------------------------ |
   | 0100010fc                   | 002710fc                       |
   | 01001100                    | 00271100                       |
   | 0100c0a4                    | 0028c0a4                       |

   即：因为程序无法预测会被加载到哪个地址，所以记录硬编码地址时以ImageBase为准；在程序运行书简，经过PE重定位，这些地址全部以加载地址为基准进行变换，从而保证程序的正常运行。

## 三、PE重定位操作原理

### 1. 基本操作原理

1. 在应用程序中查找硬编码的地址位置
2. 读取数值后，减去ImageBase（VA->RVA）
3. 加上实际加载地址（RVA->VA）

上面三个步骤即可完成PE重定位，其中最关键的是查找硬编码地址的位置，查找过程中会使用到PE文件内部的Relocation Tables（重定位表），它记录了硬编码地址便宜，是在PE文件构建中的编译/链接阶段提供的。通过重定位表查找，本质上就是根据PE头的“基址重定位表”项进行的查找。

![16-3](https://i.imgur.com/yQNLiDc.png)

如上图所示，红框内的硬编码的地址都需要经过重定位再加载到内存中。

### 2. 基址重定位表

位于PE头的DataDirectory数组的第六个元素，索引为5.如下图所示：

![16-4](https://i.imgur.com/Osux4RB.png)

上图中的基址重定位表的RVA为2f000，查看该地址处内容：

![16-5](https://i.imgur.com/HdnQ2IV.png)



![16-6](https://i.imgur.com/cOPe9t4.png)

### 3. IMAGE_BASE_RELOCATION结构体

上图中详细罗列了硬编码地址的偏移，读取该表就可以获得准确的硬编码地址偏移。基址重定位表是IMAGE_BASE_RELOCATION结构体数组。

其定义如下：

```c
typedefine struct _IMAGE_BASE_RELOCATION{
		DWORD		VirtualAddress;	//RVA值
		DOWRD		SizeOfBlock;		//重定位块的大小
		//WORD TypeOffset[1];		//以注释形式存在，非结构体成员，表示在该结构体下会出现WORD类型的数组，并且该数组元素的值就是硬编码在程序中的地址偏移。
}IMAGE_BASE_RELOCATION;

tydefine IMAGE_BASE_RELOCATION UNALIGEND * PIMAGE_BASE_RELOCATION;

```

### 4. 基地址重定位表的分析方法

下表列出上图中基址重定位表的部分内容：

| RVA   | 数据     | 注释           |
| ----- | -------- | -------------- |
| 2f000 | 00001000 | VirtualAddress |
| 2f004 | 00000150 | SizeOfBlock    |
| 2f008 | 3420     | TypeOffset     |
| 2f00a | 342d     | TypeOffset     |
| 2f00c | 3436     | TypeOffset     |

以VirtualAddress=00001000，SizeOfBlock=00000150，TypeOffset=3420为例。

TypeOffset值为2个字节，由4位的Type与12位的Offset合成：

| 类型（4位） | 偏移（12位） |
| ----------- | ------------ |
| 3           | 420          |

高4位指定Type，PE文件中常见的值为3（IMAGE_REL_BASED_HIGHLOW），64位的PE文件中常见值为A（IMAGE_REL_BASED_DIR64）。低12位位真正位移（最大地址为1000），改位移是基于VirtualAddress的位移，所以程序中硬编码地址的偏移使用以下公式进行计算：

`VirtualAddress(1000) + Offset(420) = 1420(RVA)`

下面我们在OD中看一下RVA 1420处是否实际存在要执行PE重定位操作的硬编码地址：

![16-7](https://i.imgur.com/576vC2C.png)

程序加载的基地址为270000，所以在271420处可以看到IAT的地址（VA，2710c4）。

### 5. 总结流程

1. 查找程序中硬编码地址的位置（通过基址重定位表查找）

   ![16-8](https://i.imgur.com/HdtKiCE.png)

   可以看到，RVA 1420处存在着程序的硬编码地址010010c4

2. 读取数值后，减去ImageBase值：

   010010c4 - 01000000 = 000010c4

3. 加上实际加载地址

   000010c4 + 00270000=002710c4

对于程序内硬编码的地址，PE装载器都做如上的处理，根据实际加载的内存地址修正后，将得到的值覆盖到同一位置上。对一个IMAGE_BASE_RELOCATION结构体的所有TypeOffset都做如上处理，且对RVA 1000～2000地址区域对应的所有硬编码地址都要进行PE重定位处理。如果TypeOffset值为0，说明一个IMAGE_BASE_RELOCATION结构体结束。至此，完成重定位流程。

## 四、参考

《逆向工程核心原理》

