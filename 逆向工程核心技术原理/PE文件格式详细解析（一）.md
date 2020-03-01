# PE文件格式详细解析（一）

## 一、PE文件基本介绍

PE文件是Windows操作系统下使用的一种可执行文件，由COFF（UNIX平台下的通用对象文件格式）格式文件发展而来。32位成为PE32，64位称为PE+或PE32+。

## 二、PE文件格式

1. PE文件种类如下表所示：

   | 种类         | 主扩展名           |
   | ------------ | ------------------ |
   | 可执行系列   | EXE, SCR           |
   | 库系列       | DLL, OCX, CPL, DRV |
   | 驱动程序系列 | SYS, VXD           |
   | 对象文件系列 | OBJ                |

2. 基本结构

   使用010editor（二进制文件查看工具）打开一个exe可以看到如下结构：

   ![PE_struc1](https://i.imgur.com/NHWe3JG.png)

   上图是该exe文件的起始部分，也是PE文件的头部，exe运行所需要的所有信息都存储在PE头中。

   

   ![PE_struc2](https://i.imgur.com/tnEWtgW.png)

​		从DOS头到节区头是PE头部分，其下的节区合称为PE体。文件中使用偏移（offset），内存中使用VA（Virtual Address，虚拟地址）来表示位置。文件加载到内存时，情况就会发生变化（节区大小、位置等）。文件的内容一般可分为代码（.text）、数据（.data）、资源（.rsrc）节，分别保存。PE头与各节区的尾部存在一个区域，成为NULL填充。文件/内存中节区的起始位置应该在各文件/内存最小单位的倍数上，空白区域使用NULL进行填充（如上图所示）。

3. VA&RVA

   VA指进程虚拟内存的绝对地址，RVA（Relative Virtual Address，相对虚拟地址）指从某个基准未知（ImageBase）开始的相对地址。VA与RVA的换算满足如下公式：

   ​	**RVA + IamgeBase = VA**

   PE头内部信息主要以RVA的形式进行存储，主要原因是PE文件（主要是DLL）加载到进程虚拟内存的特定位置时， 该位置可能已经加载了其他PE文件（DLL）。此时需要进行重定位将其加载到其他的空白位置，保证程序的正常运行。

## 三、PE头

1. DOS头

   主要为现代PE文件可以对早期的DOS文件进行良好兼容存在，其结构体为IMAGE_DOS_HEADER。

   大小为64字节，其中2个重要的成员分别是：

   -  e_magic:DOS签名（4D5A，MZ）
   -  e_lfanew：指示NT头的偏移（文件不同，值不同）

2. DOS存根

   stub，位于DOS头下方，可选，大小不固定，由代码与数据混合组成。

3. NT头

   结构体为IMAGE_NT_HEADERS，大小为F8，由3个成员组成：

   - 签名结构体，值为50450000h（“PE”00）
   - 文件头，表现文件大致属性，结构体为IMAGE_FILE_HEADER，重要成员有4个：
     - Machine：每个CPU都拥有的唯一的Machine码，兼容32位Intel x86芯片的Machine码为14C；
     - NumberOfSections：指出文件中存在的节区数量；
     - SizeOfOptionalHeader：指出结构体IMAGE_OPTIONAL_HEADER32（32位系统）的长度
     - Characteristics：标识文件属性，文件是否是可运行形态、是否为DLL等，以bit OR形式进行组合
   - 可选头，结构体为IMAGE_OPTIONAL_HEADER32，重要成员有9个：
     - Magic：IMAGE_OPTIONAL_HEADER32为10B，IMAGE_OPTIONAL_HEADER64为20B
     - **AddressOfEntryPoint**：持有EP的RVA值，指出程序最先执行的代码起始地址
     - ImageBase：指出文件的优先装入地址（32位进程虚拟内存范围为：0～7FFFFFFF）
     - SectionAlignment,FileAlignment：前者制定了节区在内存中的最小单位，后者制定了节区在磁盘文件中的最小单位
     - SizeOfImage：指定了PE Image在虚拟内存中所占空间的大小
     - SizeOfHeaders：指出整个PE头的大小
     - Subsystem：区分系统驱动文件和普通可执行文件
     - NumberOfRvaAndSize：指定DataDirectory数组的个数
     - DataDirectory：由IMAGE_DATA_DIRECTORY结构体组成的数组

4. 节区头

   节区头中定义了各节区的属性，包括不同的特性、访问权限等，结构体为IMAGE_SECTION_HEADER，重要成员有5个：

   - VirtualSize：内存中节区所占大小
   - VirtualAddress：内存中节区起始地址（RVA）
   - SizeOfRawData：磁盘文件中节区所占大小
   - Charateristics：节区属性（bit OR）

## 四、RVA To RAW

PE文件从磁盘到内存的映射：

1. 查找RVA所在节区

2. 使用简单的公式计算文件偏移：

   **RAW - PointerToRawData = RVA - ImageBase**

   **RAW = RVA - ImageBase + PointerToRawData**

example：ImageBase为0x10000000，节区为.text，文件中起始地址为0x00000400，内存中的起始地址为0x01001000，RVA = 5000，RAW = 5000 - 1000 + 400 = 4400。 