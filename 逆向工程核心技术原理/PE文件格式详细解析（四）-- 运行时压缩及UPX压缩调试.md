# PE文件格式详细解析（四）-- 运行时压缩及UPX压缩调试

## 一、数据压缩

1. 无损压缩（Lossless Data Compression）：经过压缩的文件能百分百恢复

   使用经过压缩的文件之前，需要点对文件进行解压缩（此过程需要保证数据完整性），常见的ZIP、RAR等是具有嗲表性的压缩文件格式，使用的压缩算法通常为Run-Length、Lepel-ZIV、Huffman等。

2. 有损压缩（Loss Data Compression）：经过压缩的文件不能恢复原状

   允许压缩文件（数据）时损失一定信息，以此换取高压缩率，多媒体文件多采用有损压缩方式，但不会影响人的视觉、听觉体验。

## 二、运行时压缩器

​	针对可执行文件，文件内部含有解压缩代码，文件在运行瞬间于内存中解压缩后执行。

1. 压缩器（Packer）：将普通PE文件创建成运行时压缩文件的应用程序
   - 目的：缩减PE文件大小；隐藏PE文件内部代码与资源
   - 种类：目的纯粹（UPX、ASPack等）、目的不纯粹（UPack、PESpin、NSAnti等）
2. 保护器（Protector）：经反逆向技术特别处理的压缩器
   - 目的：防止破解，隐藏OEP（Original Entry Point）；保护代码与资源
   - 种类：商用（ASProtect、Themida、SVKP等）、公用（UltraProtect、Morphine等）

## 三、运行时压缩测试（notepad.exe）

​	书上使用的是XP SP3的notepad.exe，此处使用的是win7 x64下的notepad.exe ，因此部分数据会产生不同。

	### 1. 压缩notepad.exe 

1. 下载UPX，地址http://upx.sourceforge.net，进行解压，并将notepad.exe拷贝到同级目录下

2. 进行压缩：`upx.exe -o notepad_upx.exe notepad.exe`

   第一个参数为输出的文件名，第二个参数为待压缩文件名（如果不在同级目录下，需要使用绝对路径）。

   压缩结果如下：

   ![upx1](https://i.imgur.com/UlwIF56.png)

   可以看到在文件大小上存在明显的尺寸减小（193536->151552）。这个压缩率比ZIP压缩要低一些，主要是因为PE文件压缩后要添加PE头，还要添加解压缩代码。

### 2. 比较notepad.exe与 notepad_upx.exe 

1. 下图(以书上版本为例)从PE文件视角比较2个文件，可以反映出UPX压缩器的特点：

![upx2](https://i.imgur.com/zO9Uv0X.png)

2. 细节比较：

   - PE头大小一致（0～400h）
   - 节区名发生变化（红框）
   - 第一个节区的RawDataSize = 0（文件中的大小为0）
   - EP文娱第二个节区，压缩前位于第一个节区
   - 资源节区（.rsrc）大小几乎无变化

3. 探讨UPX创建的空白节区，也就是RawDataSize=0的节区。使用PEView查看（此处为本机使用的notepad_upx.exe与书上不同）：

   ![upx3](https://i.imgur.com/ge4xdx5.png)

   

   查看第一个节区的相关数据，VirtualSize的大小为2C000，但是SizeOfRawData的大小为0。UPX为什么要创建一个这么大的空白节区呢？

   **原理是：经过UPX压缩的PE文件在运行时将首先将文件中的压缩代码解压到内存中的第一个节区，也就是说，解压缩代码与压缩代码的源代码都在第二个节区中，文件运行时首先执行解压缩代码，把处于压缩状态的源代码解压到第一个节区中，解压过程结束后即运行源文件的EP代码**。

   

   ## 四、总结

   这里开始初步进入调试阶段，需要好好掌握前面的知识，方便后续调试。下一节将开始od的动态调试。

   

   