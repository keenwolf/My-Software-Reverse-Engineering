# PE文件格式详细解析（三）--EAT

​	**Windows操作系统中，库是为了方便其他程序调用而集中包含相关函数的文件（DLL、SYS）。Win32 API是最具有代表性的库，其中kernel32.dll文件被称为最核心的库文件。**

## 一、基础知识

​	EAT是一种核心机制，使不同的应用程序可以调用库文件中提供的函数，只有通过EAT才能准确求得从相应库中到处函数的起始地址。PE文件内的IMAGE_EXPORT_DIRECTORY保存着导出信息，且PE文件中**仅有一个**用来说明EAT的IMAGE_EXPORT_DIRECTORY结构体。

 `备注：IAT的 IMAGE_IMPORT_DESCRIPTOR结构体以数组形式存在，且有多个成员，这主要是因为PE文件可以同时导入多个库。`

## 二、IMAGE_EXPORT_DIRECTORY结构体

### 1. 在PE头中的位置

​	在PE头中，IMAGE_OPTIONAL_HEADER32.DataDirectory[0].VirtualAddress的值几十IMAGE_EXPORT_DIRECTORY结构体数组的起始地址（RVA）。下图显示的是kernel32.dll文件的IMAGE_OPTIONAL_HEADER32.DataDirectory[0]:

![eat1](https://i.imgur.com/99pVCoQ.png)

其中第一个4字节为VirtualAddress，第二个4字节为Size。

### 2. 详细的结构代码

详细的结构代码如下：

![eat2](https://i.imgur.com/8hDcGoF.png)



下面对结构体中的部分重要成员进行解释（全部地址均为RVA）：

| 项目                  | 含义                                                |
| --------------------- | --------------------------------------------------- |
| NumberOfFuctions      | 实际Export函数的个数                                |
| NumberOFNames         | Export函数中具名的函数个数                          |
| AddressOfFunctions    | Export函数地址数组（数组元素个数=NumberOfFuctions） |
| AddrssOfNames         | 函数名称地址数组（数组元素个数=NumberOfNames）      |
| AddressOfNameOrdinals | Ordinal地址数组（元素个数=NumberOfNames）           |

### 3. kernel32.dll文件的IMAGE_EXPORT_DIRECTORY结构体

下图中描述的是kernel32.dll 文件的IMAGE_EXPORT_DIRECTORY结构体与整个的EAT结构：

![eat3](https://i.imgur.com/cwYjKQg.png)

从库中获得函数地址的API为GetProcAddress()函数，该API引用EAT来获取指定API的地址。其过程大致如下：

1. 利用AddressOfName成员转到“函数名称数组”
2. “函数名称数组”中存储着字符串地址，通过比较（strcmp）字符串，查找指定的函数名称（此时数组的索引称为name_index）
3. 利用AddressOfNameOrdinals成员，转到ordinal数组
4. 在ordinal数组中通过name_index查找相应ordinal值
5. 利用AddressOfFunctionis成员转到“函数地址数组”（EAT）
6. 在“函数地址数组”中将刚刚求得的ordinal用作数组索引，获得指定函数的起始地址

kernel32.dll中所有到处函数均有相应名称，AddressOfNameOrdinals数组的值以index=ordinal的形式存在。但存在一部分dll中的导出函数没有名称，所以仅通过ordinal导出，从Ordinal值中减去IMAGE_EXPORT_DIRECTORY.Base 成员后得到一个值，使用该值作为“函数地址数组”的索引即可查找到相应函数的地址。

## 三、完整的kernel32.dll的EAT的解析过程

以下以查找kernel32.dll中的AddAtomW函数为例，串联整个过程：

1. 由前面第一个图的VirtualAddress和Size可以获得IMAGE_EXPORT_DIRECTORY结构体的RAW为1A2C，计算过程如下：

   **RAW = RVA - VA + PTR  = 262C - 1000 + 400 = 1A2C**(此处仅以书上地址为例，每个人地址会不同)

2. 根据IMAGE_EXPORT_DIRECTORY结构的详细代码可以获得AddressOfNames成员的值为RVA =353C，RAW=293C。使用二进制查看软件查看该地址：

   

   ![eat4](https://i.imgur.com/aj6NeY2.png)

   此处为4字节RVA组成的数组，数组元素个数为NumberOfNames（3BA）。

3. 查找指定函数名称

   函数名称为“ AddAtomW”，在上图中找到RVA数组的第三个元素的值RVA:4BBD -> RAW:3FBD，进入相应地址即可看到该字符串，函数名为数组的第三个元素，数组索引为2.

   ![eat5](https://i.imgur.com/cLexE3I.png)

4. Ordinal数组

   AddressOfNameOrdinals成员的值为RVA:4424 -> RAW:3824:

   ![eat6](https://i.imgur.com/AHbKA51.png)

   oridinal数组中各元素大小为2字节。

5. ordinal

   将4中确定的index值2应用到数组即可求得Ordinal(2)

   `AddressOfNameOrdinals[index] = ordinal(index = 2, ordinal = 2)`

6. 函数地址数组 - EAT

   AddressOfFunctions成员的值为RVA:2654 -> RVA:1A54：

   ![eat7](https://i.imgur.com/9qI40MB.png)

7. AddAtomW函数地址

   将5中求得的Ordinal用于上图数组的索引，求得RVA = 00326F1

   `AddressOfFunctionis[ordinal] = RVA(ordinal = 2,RVA = 326F1)`

   书中kernel32.dll 的ImageBase为7C7D0000，所以AddAtomW函数的实际地址VA = 7C7D0000 + 326F1 = 7C8026F1

   以上地址可以使用od进行验证，此处不多赘述。











