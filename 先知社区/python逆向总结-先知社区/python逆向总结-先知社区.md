

# python 逆向总结 - 先知社区

python 逆向总结



# python 类逆向

## Python 运行原理

Python 是解释型语言，没有严格意义上的编译和汇编过程。但是一般可以认为编写好的 python 源文件，由 python 解释器翻译成以.pyc 为结尾的字节码文件。pyc 文件是二进制文件，可以由 python 虚拟机直接运行。

Python 在执行 import 语句时，将会到已设定的 path 中寻找对应的模块。并且把对应的模块编译成相应的 PyCodeObject 中间结果，然后创建 pyc 文件，并将中间结果写入该文件。然后，Python 会 import 这个 pyc 文件，实际上也就是将 pyc 文件中的 PyCodeObject 重新复制到内存中。而被直接运行的 python 代码一般不会生成 pyc 文件。

加载模块时，如果同时存在.py 和.pyc，Python 会尝试使用.pyc，如果.pyc 的编译时间早于.py 的修改时间，则重新编译.py 并更新.pyc。  
[![](assets/1701612121-99249a1b548637be4947388fdbae0b58.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824211237-e5562876-427f-1.png)  
1.把原始代码编译成字节码

编译后的字节码是特定于 Python 的一种表现形式，它不是二进制的机器码，需要进一步编译才能被机器执行。如果 Python 进程在机器上拥有写入权限，那么它将把程序的字节码保存为一个以.pyc 为扩展名的文件，如果 Python 无法在机器上写入字节码，那么字节码将会在内存中生成并在程序结束时自动丢弃。

2.把编译好的字节码转发到 Python 虚拟机（PVM）中进行执行

PVM(Python Virtual Machine) 是 Python 的运行引擎，是 Python 系统的一部分，它是迭代运行字节码指令的一个大循环、一个接一个地完成操作。

### Python 分发的文件类型

#### python 打包原理

Python 代码的基本运行过程：

1.  Python.exe 调用 XX.py(源码)，解释并运行。
2.  Python.exe 调用 XX.pyc(字节码)，解释并运行。
3.  Python.exe 调用 XX.pyd(机器码)，调用运行。
4.  如果有依赖的库，根据上面三种情况调用运行。

##### PyInstaller

原理：分析脚本文件，递归找到所有依赖的模块。如果依赖模块有.pyd 文件，即将其复制到 disk 目录。如果没有.pyd 文件，则生成.pyc 文件拷贝到 disk 目录，并压缩为.zip 保存。制作一个 exe，导入 PythonXX.dll(解析器库)，并添加 exe 运行需要的相关运行时库。这就构成了一个不用安装 Python 的运行包。

如果要生成一个 exe，则利用 PE 结构将整个包加载内存中再来运行。

##### Nuitka

Nuitka 是一个 Python 编写的 Python 解释器，所以 Nuitka 可以作为 Python 的扩展库使用。Nuitka 会将所有没有生成 pyd 的模块的.py 代码转换成 C 代码，然后调用 gcc/MSVC 编译成.pyd。

#### .pyc

.pyc：python 编译后的二进制文件，是.py 文件经过编译产生的字节码，pyc 文件是可以由 python 虚拟机直接执行的程序（PyCodeObject 对象在硬盘上的保存形式）。

pyc 文件**仅**在由另一个.py 文件或模块导入时从.py 文件创建（**import**）。触发 pyc 文件生成不仅可以通过 import，还可以通过 py\_compile 模块手动生成。

pyc 文件会加快程序的**加载速度**，而不会加快程序的实际执行速度。

##### pyc 文件格式

[![](assets/1701612121-a41f755acdbc610b8677233dbbd82b1d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824211402-1834ca68-4280-1.png)  
pyc 文件一般由 3 个部分组成：

1.  Magic num：标识此 pyc 的版本信息，不同的版本的 Magic 都在 `Python/import.c` 内定义
2.  文件创建时间：UNIX 时间戳（从 1970.1.1 开始计数秒数）
3.  序列化了的 PyCodeObject：此结构在 `Include/code.h` 内定义，序列化方法在 `Python/marshal.c` 内定义

Python 在不同的版本，pyc 的头部长度和内容是不同的：

PEP 3147:  
[![](assets/1701612121-05622b00bcdb6c0ee2fa61c6467b7dcd.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824211455-37c1232c-4280-1.png)  
包含两个 32 位大端数字，后面跟的序列化的 PyCodeObject。32 位数字表示一个 Magic Num 和 Timestamp。每当 Python 改变字节码格式时，Magic Num 会改变，例如，通过向其虚拟机添加新的字节码。这确保为以前版本的 VM 构建的 pyc 文件不会造成问题。Timestamp 用于确保 pyc 文件与用于创建它的 py 文件匹配。当 Magic Num 或 Timestamp 不匹配时，将重新编译 py 文件并写入新的 pyc 文件。

PEP 552:  
[![](assets/1701612121-0d195fc3fa934ae7096ced60ea470afb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824212211-3ba42088-4281-1.png)  
pyc 头文件目前由 3 个 32 位的字组成。我们将把它扩大到 4 个。第一个单词将继续是 magic number，对字节码和 pyc 格式进行版本控制。第二个 4byte 新增加的字段，将是一个位字段 (bit field)，对报头其余部分的解释和 pyc 的失效行为取决于位字段的内容。

如果位字段 (bit field) 为 0，则 pyc 是传统的基于时间戳的 pyc。即第三个和第四个 4 字节内容分别是时间戳和文件大小，通过比较源文件的元数据和头文件中的元数据来进行无效判断。

如果位字段的最低位被设置，则 pyc 是基于哈希的 pyc。我们将第二个最低位称为 check\_source 标志。位字段之后是源文件的 64 位散列。我们将使用带有源文件内容硬编码密钥。

对于 Magic 值，它的逻辑为：后 2bytes 为 0D0A，前面的值满足： \[min, max\]范围，版本信息定义参考结构内容，示例分析代码如下：

```plain
typedef struct {
    unsigned short min;
    unsigned short max;

    wchar_t version[MAX_VERSION_SIZE];
} PYC_MAGIC;

static PYC_MAGIC magic_values[] = {
    { 50823, 50823, L"2.0" },
    { 60202, 60202, L"2.1" },
    { 60717, 60717, L"2.2" },
    { 62011, 62021, L"2.3" },
    { 62041, 62061, L"2.4" },
    { 62071, 62131, L"2.5" },
    { 62151, 62161, L"2.6" },
    { 62171, 62211, L"2.7" },
    { 3000, 3131, L"3.0" },
    { 3141, 3151, L"3.1" },
    { 3160, 3180, L"3.2" },
    { 3190, 3230, L"3.3" },
    { 3250, 3310, L"3.4" },
    { 3320, 3351, L"3.5" },
    { 3360, 3379, L"3.6" },
    { 3390, 3399, L"3.7" },
    { 3400, 3419, L"3.8" },
    { 0 }
};
```

###### 序列化

**PyObject**：PyObject 如何序列化，哪些内容被参与了序列化，参看 `Python/marshal.c` 内的函数 `w_object` 函数

**PyCodeObject**：

结构体 PyCodeObject 在 `Include/code.h` 中定义如下：

```plain
typedef struct {
    PyObject_HEAD
    int co_argcount; /* 位置参数个数 */
    int co_nlocals; /* 局部变量个数 */
    int co_stacksize; /* 栈大小 */
    int co_flags;
    PyObject *co_code; /* 字节码指令序列 */
    PyObject *co_consts; /* 所有常量集合 */
    PyObject *co_names; /* 所有符号名称集合 */
    PyObject *co_varnames; /* 局部变量名称集合 */
    PyObject *co_freevars; /* 闭包用的的变量名集合 */
    PyObject *co_cellvars; /* 内部嵌套函数引用的变量名集合 */
    /* The rest doesn’t count for hash/cmp */
    PyObject *co_filename; /* 代码所在文件名 */
    PyObject *co_name; /* 模块名 | 函数名 | 类名 */
    int co_firstlineno; /* 代码块在文件中的起始行号 */
{
```

在 Python 代码中，每个**作用域**(或者叫 block 或者名字空间) 对应一个 PyCodeObject 对象，所以会出现嵌套：比如 一个 module 类 定义了 N 个 class, 每个 class 内又定义了 M 个方法。每个 子作用域 对应的 PyCodeObject 会出现在它的 父作用域 对应的 PyCodeObject 的 co\_consts 字段里。

#### .pyo

pyo 文件是源代码文件经过优化编译后生成的文件，是 pyc 文件的优化版本，由解释器在导入模块时创建。当我们调用 Python 解释器时，通过添加“-O”标志来启用优化器，生成.pyo 文件。在 Python3.5 之后，不再使用.pyo 文件名，而是生成文件名类似“test.opt-n.pyc 的文件。

#### .pyd

.pyd 文件类型是特定于 Windows 操作系统类平台的，是一个动态链接库，它包含一个或一组 Python 模块，由其他 Python 代码调用。要创建.pyd 文件，需要创建一个名为 example.pyd 的模块。在这个模块中，需要创建一个名为 PyInit\_example() 的函数。当程序调用这个库时，它们需要调用 import foo, PyInit\_example() 函数将运行。

#### .pyz

executable python zip archives

具体内容参见下：ZlibArchive

### PyInstaller 打包

PyInstaller 中使用了两种存档。一个是 ZlibArchive，它能高效地存储 Python 模块，并通过一些导入钩子直接导入。另一个是 CArchive，类似于.zip 文件，这是一种打包 (或压缩) 任意数据块的通用方法。

#### ZlibArchive

ZlibArchive 包含压缩的.pyc 或.pyo 文件。spec 文件中的 PYZ 类调用创建了一个 ZlibArchive。

ZlibArchive 中的目录是一个 Python 字典，它的 Key(import 语句中给定的成员名) 与 ZlibArchive 中的查找位置和长度相关联。ZlibArchive 的所有部分都以编组格式存储，因此与平台无关。

ZlibArchive 在运行时用于导入绑定的 python 模块，即使使用最大压缩，也比正常导入快。  
[![](assets/1701612121-0c9e394bdbeb925c7925bee1ac149c99.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824212544-ba903fc6-4281-1.png)

#### CArchive

CArchive 很像一个.zip 文件，可以包含任何类型的文件。可以 python 创建，也可以从 C 代码中解包。CArchive 可以附加到另一个文件，比如 ELF 和 COFF 可执行文件，存档是在文件的末尾用它的目录创建的，后面只跟一个 cookie，它告诉目录从哪里开始以及存档本身从哪里开始。CArchive 可以嵌入到另一个 CArchive 中，内部存档可以在适当的地方打开使用，而不需要提取。

每个目录条目都有可变的长度。条目中的第一个字段给出了条目的长度。最后一个字段是相应打包文件的名称。名称以空内容结尾，每一个成员都可以进行压缩。还有一个与每个成员相关联的类型代码供程序使用。

Carchive 结构  
[![](assets/1701612121-6d52004aebe4bda133ecc46a3dc715b3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824212627-d446926c-4281-1.png)

自解压执行文件结构  
[![](assets/1701612121-5ee5a092fc2cb4f182a54cc3f3cab39b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824212704-ea44fea0-4281-1.png)

## pyc 反编译

_在 cmd 里输入`python pycname.pyc`即可让.pyc 文件在相应的 python 版本环境下运行_

#### uncompyle6

Uncompyle6，是一个 Python 原生的跨版本反编译器，是 decompyle, uncompyle, uncompyle2 的后继版本。Uncompyle6 能够将 Python 二进制文件 pyc, pyo 反编译为对应的源代码。支持 **Python 1.0-3.8** 版本的反编译，拥有较好的逻辑处理和解析能力。

**安装**：要在 python3.8 及之前的版本下载。要想不破坏我现有的 3.11 版本环境，需要搭建虚拟环境后安装。

虚拟环境安装详细过程见最后

**使用**：

在已经搭建好虚拟环境的前提下

```plain
.\py38\Scripts\activate
#进入虚拟环境

uncompyle6 -o test.py test.pyc
#将 test.pyc 文件反编译，并输出到 test.py 文件中
```

生成的文件会存放在 C:\\Users\\admin 目录下，剪切出来即可

[![](assets/1701612121-430f70df3107268b992eacfed5ccd80a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824212821-17f6d9cc-4282-1.png)

如果遇到错误，可能因为 pyc 文件生成时，头部的 magic number 被清理，需要另外补上

#### pycdc

pycdc 是一个用于 pyc 反编译的工具，适用于 **Python 3.9** 及更高版本。

在 GitHub 下载 pycdc [https://github.com/zrax/pycdc](https://github.com/zrax/pycdc)

**Linux**下下载使用

```plain
apt-get install cmake
```

在下载目录下输入 `cmake CMakeLists.txt`，会在当前目录下生成 Makefile 文件，然后输入 `make` 即可进行编译安装

将文件和 pycdc 文件放在一个目录下

```plain
./pycdc 文件名.pyc
```

终端就会输出对应的 Python 源代码了

**Windows**下

下载安装 cmake [Download | CMake](https://cmake.org/download/)

[![](assets/1701612121-30113a05375f3d8e71de43b508687ee9.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824212941-4808f6ae-4282-1.png)

之前 make 不了，在安装 visual studio 之后实验了一下

[![](assets/1701612121-926de7698230658dac74a824f417bc19.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213013-5b2d45a0-4282-1.png)

打开 CMakeLists.txt 点击运行即可

在文件夹内的路径是`"\pycdc-master\out\build\x64-Debug\pycdc.exe"`，使用的时候拷出来，与 pyc 文件放在同一路径下

```plain
./pycdc 文件名.pyc
```

[![](assets/1701612121-835b6a91e1a844a77ffd414e83ec9ba0.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213051-71d5663e-4282-1.png)

在线工具的内核也是 pycdc [python 反编译 - 在线工具 (tool.lu)](https://tool.lu/pyc/)

同时编译的 pycdas.exe 是显示 python 字节码的工具

使用方式与 pycdc 相同

[![](assets/1701612121-21df8f6215580c7c923c66e4455a3a82.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213126-86baddcc-4282-1.png)

### 题目练习

#### \[watevrCTF 2019\]Repyc

pyc 文件。反编译一下是乱码

[![](assets/1701612121-b77ffcb686ffe37dc1c54a8bbfa6708c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213158-99d3833c-4282-1.png)

函数里都是 if 语句，还有±\*/&|这些操作，很像是一个 VM，那么下面的函数调用的列表参数就是这个 VM 的代码。乱码可以进行替换操作

wp 里修改好的

```plain
a = 0
b = 1
c = 2

def fun1 (arg):
    A= a
    M= [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
    C= [a] * 100
    K= []
    while arg[A][a] != '12':
        #print(M)
        opcode= arg[A][a].lower()
        opn = arg[A][b:]
        #print(opn)
        if opcode== '4':
            M [opn[a]] = M [opn[b]] + M[opn[c]]
        elif opcode== '3':
            M [opn[a]] = M [opn[b]] * M[opn[c]]
        elif opcode== '5':
            M [opn[a]] = M [opn[a]]
        elif opcode== '1':
            M[opn[a]] = opn[b]
        elif opcode== '11':
            M[opn[a]] = C[opn[b]]
        elif opcode== '6':
            M[opn[a]] = a
        elif opcode== '2':
            C[opn[a]] = input(M[opn[b]])
        elif opcode== '10':
            print(M[opn[a]])
        elif opcode== '9':
            M[7] = a
            for i in range(len(M[opn[a]])):
                print( M[opn[a]], M[opn[b]])
                if M[opn[a]] != M[opn[b]]:
                    M[7] = b
                    A= M[opn[c]]
                    K.append(A)

        elif opcode== '7':
            s= ''
            for i in range(len(M[opn[a]])):
                s+= chr(ord(M[opn[a]][i]) ^ M[opn[b]])
            print(s)
            M[opn[a]] = s
        elif opcode== '8':
            s= ''
            for i in range(len(M[opn[a]])):
                #print(M[opn[a]][i],M[opn[b]])
                s+= chr(ord(M[opn[a]][i]) - M[opn[b]])
            M[opn[a]] = s

        A+= b


fun1([
 [
  '1', 0, 'Authentication token: '],
 [
  '2', 0, 0],
 [
  '1', 6, 'á×äÓâæíäàßåÉÛãåäÉÖÓÉäàÓÉÖÓåäÉÓÚÕæïèäßÙÚÉÛÓäàÙÔÉÓâæÉàÓÚÕÓÒÙæäàÉäàßåÉßåÉäàÓÉÚÓáÉ·Ôâ×ÚÕÓÔÉ³ÚÕæïèäßÙÚÉÅä×ÚÔ×æÔÉ×Úïá×ïåÉßÉÔÙÚäÉæÓ×ÜÜïÉà×âÓÉ×ÉÑÙÙÔÉâßÔÉÖãäÉßÉæÓ×ÜÜïÉÓÚÞÙïÉäàßåÉåÙÚÑÉßÉàÙèÓÉïÙãÉáßÜÜÉÓÚÞÙïÉßäÉ×åáÓÜÜ\x97ÉïÙãäãÖÓ\x9aÕÙÛ\x99á×äÕà©â«³£ï²ÕÔÈ·±â¨ë'],
 [
  '1', 2, c ** (3 * c + b) - c ** (c + b)],
 [
  '1', 4, 15],
 [
  '1', 3, b],
 [
  '3', 2, 2, 3],
 [
  '4', c, c, 4],
 [
  '5', a, c],
 [
  '6', 3],
 [
  '7', 6, 3],
 [
  '1', 0, 'Thanks.'],
 [
  '1', 1, 'Authorizing access...'],
 [
  '10', 0],
 [
  '11', a, a],
 [
  '7', a, 2],
 [
  '8', a, 4],
 [
  '1', 5, 19],
 [
  '9', 0, 6, 5], 
 [
  '10', 1],
 [
  '12'],
 [
  '1', b, 'Access denied!'],
 [
  '10', 1],
 [
  '12']])
```

读上面的 elif 推测 opcode

```plain
opcode    operation
1         mov M[op1] op2
2         read M[op1] stdin
3         M[op1]=M[op2]+M[op3]
4         M[op1]=M[op2]*M[op3]
5         M[op1]=M[op1]
6         M[op1]=0
7         M[op1]中每个字符异或 M[op2]
8         M[op1]中每个字符减去 M[op2]
9         jnp M[op1] M[op2]
10        write M[op1] stdout
11        mov M[op1] C[op2]
12        exit
```

对 enc 进行异或和减去 15

```plain
aaa='á×äÓâæíäàßåÉÛãåäÉÖÓÉäàÓÉÖÓåäÉÓÚÕæïèäßÙÚÉÛÓäàÙÔÉÓâæÉàÓÚÕÓÒÙæäàÉäàßåÉßåÉäàÓÉÚÓáÉ·Ôâ×ÚÕÓÔÉ³ÚÕæïèäßÙÚÉÅä×ÚÔ×æÔÉ×Úïá×ïåÉßÉÔÙÚäÉæÓ×ÜÜïÉà×âÓÉ×ÉÑÙÙÔÉâßÔÉÖãäÉßÉæÓ×ÜÜïÉÓÚÞÙïÉäàßåÉåÙÚÑÉßÉàÙèÓÉïÙãÉáßÜÜÉÓÚÞÙïÉßäÉ×åáÓÜÜ\x97ÉïÙãäãÖÓ\x9aÕÙÛ\x99á×äÕà©â«³£ï²ÕÔÈ·±â¨ë'
flag=''
for i in range(len(aaa)):
    flag+=chr((ord(aaa[i])+15)^135)
print(flag)
```

watevr{this\_must\_be\_the\_best\_encryption\_method\_evr\_henceforth\_this\_is\_the\_new\_Advanced\_Encryption\_Standard\_anyways\_i\_dont\_really\_have\_a\_good\_vid\_but\_i\_really\_enjoy\_this\_song\_i\_hope\_you\_will\_enjoy\_it\_aswell!\_youtube.com/watch?v=E5yFcdPAGv0}

#### \[SUCTF2018\]HelloPython

[![](assets/1701612121-5054a202d70ca854086aba7561061d09.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213343-d81b05f2-4282-1.png)

pyc 文件

```plain
(lambda __operator, __print, __g, __contextlib, __y: [ (lambda __mod: [ [ [ (lambda __items, __after, __sentinel: __y((lambda __this: (lambda : (lambda __i: [ (lambda __out: (lambda __ctx: [__ctx.__enter__(), __ctx.__exit__(None, None, None), __out[0]((lambda : __this()))][2])(__contextlib.nested(type('except', (), {'__enter__': lambda self: None, '__exit__': lambda __self, __exctype, __value, __traceback: __exctype is not None and [ True for __out[0] in [(sys.exit(0), (lambda after: after()))[1]] ][0]})(), type('try', (), {'__enter__': lambda self: None, '__exit__': lambda __self, __exctype, __value, __traceback: [ False for __out[0] in [(v.append(int(word, 16)), (lambda __after: __after()))[1]] ][0]})())))([None]) for __g['word'] in [__i] ][0] if __i is not __sentinel else __after())(next(__items, __sentinel)))))())(iter(p_text.split('_')), (lambda : [ [ [ [ [ [ [ (lambda __after: __y((lambda __this: (lambda : (lambda __target: [ (lambda __target: [ (lambda __target: [ [ __this() for __g['n'] in [__operator.isub(__g['n'], 1)] ][0] for __target.value in [__operator.iadd(__target.value, (y.value << 4) + k[2] ^ y.value + x.value ^ (y.value >> 5) + k[3])] ][0])(z) for __target.value in [__operator.iadd(__target.value, (z.value << 4) + k[0] ^ z.value + x.value ^ (z.value >> 5) + k[1])] ][0])(y) for __target.value in [__operator.iadd(__target.value, u)] ][0])(x) if n > 0 else __after())))())((lambda : [ [ (__print(('').join(map(hex, w)).replace('0x', '').replace('L', '')), None)[1] for w[1] in [z.value] ][0] for w[0] in [y.value] ][0])) for __g['w'] in [[0, 0]] ][0] for __g['n'] in [32] ][0] for __g['u'] in [2654435769] ][0] for __g['x'] in [c_uint32(0)] ][0] for __g['z'] in [c_uint32(v[1])] ][0] for __g['y'] in [c_uint32(v[0])] ][0] for __g['k'] in [[3735928559, 590558003, 19088743, 4275878552]] ][0]), [])
 for __g['v'] in [[]] ][0]
 for __g['p_text'] in [raw_input('plain text:\n> ')] ][0]
 for __g['c_uint32'] in [__mod.c_uint32] ][0])(__import__('ctypes', __g, __g, ('c_uint32', ), 0))
 for __g['sys'] in [__import__('sys', __g, __g)] ][0])(__import__('operator', level=0), __import__('__builtin__', level=0).__dict__['print'], globals(), __import__('contextlib', level=0), (lambda f: (lambda x: x(x))((lambda y: f((lambda : y(y)()))))))
```

反编译之后是混淆过的乱码

虽然很难看，但是可以硬猜，通过 2654435769 确定的 TEA 加密

通过`(y.value << 4) + k[2] ^ y.value + x.value ^ (y.value >> 5) + k[3])]`之类的也能猜到

[![](assets/1701612121-64637a0c5cf1bb7be9b9f764110ad21e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213504-084db472-4283-1.png)

套到 tea 解密的脚本

[![](assets/1701612121-b668f886d670396d2425e02fd100e0e3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213538-1c890b26-4283-1.png)

## python exe 反编译

### PyInstaller：将.py 编译成.exe

更多操作选项：[python：pyinstaller 使用指南\_python pyinstall](https://blog.csdn.net/weixin_41916986/article/details/122343057)

pyinstaller 可以将 python 项目在不同平台上打包为可执行文件。

打包流程：读取编写好的 python 项目–>分析其中调用的模块和库，并收集其文件副本（包括 python 的解释器）–>将副本和 python 项目文件封装在一个可执行文件中

使用 pip 安装：`pip install pyinstaller`,安装后会在 Scripts 文件夹下生成 pyinstaller.exe`"C:\Python311\Scripts\pyinstaller.exe"`,接下来就可以使用该工具将 Python 程序生成 exe 程序了。

创建一个新文件夹，写一个非常简单的 helloworld 并保存到新文件夹中

```plain
def main():
    print('hello')
    print('world')
# 增加调用 main() 函数
if __name__ == '__main__':
    main()
```

在文件的目录下执行`pyinstaller -F NAME.py`

[![](assets/1701612121-58b0a771ff1fc4cf4825c60e95226049.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213614-322c3458-4283-1.png)

之后在新生成的 dist 目录下就有打包好的 exe 文件

[![](assets/1701612121-6fbcd7adb214ba910273ee61e15cd671.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213646-4588bf8a-4283-1.png)

同时会生成如下三个文件

[![](assets/1701612121-9d0a21aeafefa2a4a91db1f26505e10a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213708-5261b284-4283-1.png)

##### .spec 配置文件

```plain
block_cipher = None
#以 py 文件为输入，分析 py 文件的依赖模块，并生成相应的信息
a = Analysis(['xxx.py'], # 要打包.py 文件名列表，和 xxx.py 同级可以不同添加
       pathex=['D:\\abc\\def\\project_v1.0'], # 项目路径
       binaries=[], # 程序调用外部 pyd、dll 文件（二进制文件路径）以数组形式传入;例:('D:\\pro\\text.dll', 'pro'),将'pdftotext.dll'pro，与原项目结构一致即可
       datas=[], # 存放的资源文件（图片、文本等静态文件）以数组形成传入;例：('D:\\static\\c.ioc','static'),将'cc.ioc'打包之后放在 static 目录，与原项目结构一致即可
       hiddenimports=[], # pyinstaller 解析模块时可能会遗漏某些模块（not visible to the analysis phase），造成打包后执行程序时出现类似 No Module named xxx;这时就需要在 hiddenimports 中加入遗漏的模块
       hookspath=[],
       runtime_hooks=[],
       excludes=[], # 去除不必要的模块 import，写在 excludes 中添加此模块
       win_no_prefer_redirects=False,
       win_private_assemblies=False,
       cipher=block_cipher)

# .pyz 的压缩包，包含程序运行需要的所有依赖
pyz = PYZ(a.pure, a.zipped_data, 
       cipher=block_cipher)

# 根据 Analysis 和 PYZ 生成单个 exe 程序所需要的属性及其配置
exe = EXE(pyz,
     a.scripts,
     exclude_binaries=True,
     name='xxx', # 生成 exe 文件的名字
     debug=False, # debug 模式
     strip=False,
     upx=True,
     console=False, # 是否在打开 exe 文件时打开 cmd 命令框
     icon='C:\Users\xx.ico' ) # 设置 exe 程序图标，ico 格式文件（16*16）

# 收集前三个部分的内容进行整合，生成程序所需要的依赖包，及资源文件和配置
coll = COLLECT(exe,
        a.binaries,
        a.zipfiles,
        a.datas,
        strip=False,
        upx=True,
        name='fastplot')
```

### pyinstxtractor

下载 pyinstxtractor.py

将 pyinstxtractor.py 工具与要反编译的 main.exe 文件放入同一个工作目录下，在当前目录的终端界面输入`python pyinstxtractor.py main.exe`，得到 main.exe\_extracted 文件夹。

[![](assets/1701612121-9745be7ebea49e160ad9f823af09dad7.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213825-805499ea-4283-1.png)

找到 main 文件，将其后缀改为 pyc 并用 winhex 打开

[![](assets/1701612121-d7011e23e9a4634e91357358513fb976.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213850-8f07cc3c-4283-1.png)

要补全文件头才能正常使用

从同文件夹下的 struct 文件中找到该 python 版本的 magic number，将其补在 E3 之前，得到完整的 pyc 文件

```plain
在 Python3.7 及以上版本的编译后二进制文件中，头部除了四字节 Magic Number，还有四个字节的空位和八个字节的时间戳 + 大小信息，后者对文件反编译没有影响，全部填充 0 即可；
Python3.3 - Python3.7（包含 3.3）版本中，只需要 Magic Number 和八位时间戳 + 大小信息
Python3.3 以下的版本中，只有 Magic Number 和四位时间戳
```

[![](assets/1701612121-a747d70a8881526f31dc0ba7e8491308.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824213929-a621c760-4283-1.png)

之后使用前面的方法将 pyc 文件反编译成 py 即可

### pyi-archive\_viewer

pyi-archive\_viewer 是 PyInstaller 自己提供的工具，它可以直接提取打包结果 exe 中的 pyc 文件。

[Advanced Topics — PyInstaller 5.13.0 documentation](https://pyinstaller.org/en/stable/advanced-topics.html#using-pyi-archive-viewer)

查看 exe 内部的文件结构：

```plain
pyi-archive_viewer name.exe
```

[![](assets/1701612121-f3faceab66675d86b877a16762f99956.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214031-cb50e89a-4283-1.png)

操作指令

```plain
U: go Up one level
O <name>: open embedded archive name
X <name>: extract name
Q: quit
```

提取操作如下

[![](assets/1701612121-8ae5dc9d75e5ba30b05f8743a9abc45c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214054-d8ecc0dc-4283-1.png)

### pyz

[Python 逆向——Pyinstaller 逆向 - 软件逆向 - 看雪 - 安全社区](https://bbs.kanxue.com/thread-271253.htm)

Pyinstaller.py 文件有一个-key 参数，可以对一些 pyc 文件进行加密，加密后的 pyc 文件称为 pyz 文件

使用如下指令加密（要先`pip install tinyaes`）

```plain
pyinstaller --key=4444 -F xxx.py
```

写一个实验程序

```plain
def main():
    print("hellopyz")

if __name__ == '__main__':
    main()
```

打包

[![](assets/1701612121-3cb54033f9405732211017b52f2f061f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214131-eee0c4d8-4283-1.png)

打包后我们对其进行解包，发现加密并没有什么用

[![](assets/1701612121-40845c6967d180c97a29d63ba0dae9d8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214158-ff04b554-4283-1.png)

原因是 pyinstaller 只对依赖库进行了加密，并没有对主程序做加密处理。

### 将.py 编译成.pyd 来防止反编译

可以将模块 py 文件编译为动态链接库，这样破解难度将大大增加，pyd 使用 cython 即可编译。

pyd 文件类似于 DLL, 一般用 C/C++ 语言编译而成，可用作**模块**导入 Python 程序中。pyd 文件仅适用于特定版本的 Python, **不同版本间互不兼容**, 如 Python3.8 不支持 3.7 版本的文件。pyd 文件用 C/C++ 语言编译而成，难以被反编译，在保护 Python 程序源码上有很好的效果。而且由于使用了 C/C++ 等低级语言，代码执行效率较高。

```plain
>>> import _tkinter # 导入 pyd 文件
>>> _tkinter.__file__
'...\\_tkinter.pyd'
```

python 有的可能不带 cython，安装即可：

```plain
pip install Cython
```

编译 pyd 需要 c 编译器，可以使用 Visual Studio 的 C 编译器或 MinGW 编译器，这里我选择 Visual Studio。

Visual Studio 环境要通过 VisualStudioSetup.exe 安装好 C++ 开发和 Python 开发两个选项。

##### 从 py 文件生成 pyd 文件

通过已有的.py 文件生成 pyd 文件，需要使用 distutils 模块和 Cython 库。

用一个例子

```plain
#test.py
def main():
    print("Hello world pyd")

#main.py
import test
test.main()

#setup.py:生成 pyd 文件的主程序
from distutils.core import setup
from Cython.Build import cythonize
import sys,os,traceback

setup(
    name = 'test',
    ext_modules = cythonize('test.py'),
)
```

运行后程序目录下会生成一个 test.c 文件，包含了 pyd 文件所用到的 C 语言代码。将生成的 pyd 文件重命名为 test.pyd（也可以不重命名），运行 main.py，就可以看到 pyd 文件中的结果了。  
使用 PyInstaller 等库打包 exe 的时候，先打包 main.py 作为主程序。即使 exe 被某个库反编译，得到的只是主程序 main.py, 而 pyd 文件中的代码是很难被反编译的。

[![](assets/1701612121-273763a74e43bcb1f489ce5e68efaa7e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214257-228b5848-4284-1.png)

##### 编写 C/C++ 代码，编译成 pyd 文件

打开 Visual Studio 创建新项目，选择 Python 扩展模块为项目类型，输入项目名称，并选择项目所在的文件夹。

[![](assets/1701612121-dfabcc477b4e7308c65bcbf374b5675a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214322-317e4c0c-4284-1.png)

等 Visual Studio 加载好后，会自动生成 pyd 文件的初始 C 语言代码。编写完成后，打开菜单“生成” -> “生成 xxxx”，就可以在项目所在文件夹的\\Debug 目录下找到生成的 pyd 文件。使用"调试"菜单中的调试命令，或者按 F5 键，就可以直接调试 pyd 文件。

```plain
#include <Python.h>

/*
 * Implements an example function.
 */
PyDoc_STRVAR(pyddd_example_doc, "example(obj, number)\
\
Example function");

PyObject *pyddd_example(PyObject *self, PyObject *args, PyObject *kwargs) {
    /* Shared references that do not need Py_DECREF before returning. */
    PyObject *obj = NULL;
    int number = 0;

    /* Parse positional and keyword arguments */
    static char* keywords[] = { "obj", "number", NULL };
    if (!PyArg_ParseTupleAndKeywords(args, kwargs, "Oi", keywords, &obj, &number)) {
        return NULL;
    }

    /* Function implementation starts here */

    if (number < 0) {
        PyErr_SetObject(PyExc_ValueError, obj);
        return NULL;    /* return NULL indicates error */
    }

    Py_RETURN_NONE;
}

/*
 * List of functions to add to pyddd in exec_pyddd().
 */
static PyMethodDef pyddd_functions[] = {
    { "example", (PyCFunction)pyddd_example, METH_VARARGS | METH_KEYWORDS, pyddd_example_doc },
    { NULL, NULL, 0, NULL } /* marks end of array */
};

/*
 * Initialize pyddd. May be called multiple times, so avoid
 * using static state.
 */
int exec_pyddd(PyObject *module) {
    PyModule_AddFunctions(module, pyddd_functions);

    PyModule_AddStringConstant(module, "__author__", "admin");
    PyModule_AddStringConstant(module, "__version__", "1.0.0");
    PyModule_AddIntConstant(module, "year", 2023);

    return 0; /* success */
}

/*
 * Documentation for pyddd.
 */
PyDoc_STRVAR(pyddd_doc, "The pyddd module");


static PyModuleDef_Slot pyddd_slots[] = {
    { Py_mod_exec, exec_pyddd },
    { 0, NULL }
};

static PyModuleDef pyddd_def = {
    PyModuleDef_HEAD_INIT,
    "pyddd",
    pyddd_doc,
    0,              /* m_size */
    NULL,           /* m_methods */
    pyddd_slots,
    NULL,           /* m_traverse */
    NULL,           /* m_clear */
    NULL,           /* m_free */
};

PyMODINIT_FUNC PyInit_pyddd() {
    return PyModuleDef_Init(&pyddd_def);
}
```

注：当同时存在 mylib.pyd 和 mylib.py 时，引入优先级是 pyd>py，所以不用移除 py 文件，默认引入时就是 pyd。

## python 字节码

### 基本知识

#### 机器码

机器码 (machine code)，学名机器语言指令，有时也被称为原生码（Native Code），是电脑 CPU 直接读取运行的机器指令 (计算机只认识 0 和 1)，运行速度最快。

#### 字节码

字节码（byte code）是一种包含执行程序、由一序列 OP 代码 (操作码)/数据对 组成的二进制文件。字节码是一种中间码，它比机器码更抽象，需要直译器转译后才能成为机器码的中间代码。

通常情况下它是已经经过编译，但与特定机器码无关。字节码通常不像源码一样可以让人阅读，而是编码后的数值常量、引用、指令等构成的序列。字节码主要为了实现特定软件运行和软件环境、与硬件环境无关。字节码的实现方式是通过编译器和虚拟机器。编译器将源码编译成字节码，特定平台上的虚拟机器将字节码转译为可以直接执行的指令。

### dis

Python 代码是编译成字节码再放到 CPython 编译器中执行，而 dis 模块可以将 python 代码分解为字节码。

[dis --- Python 字节码反汇编器 — Python 3.11.4 文档](https://docs.python.org/zh-cn/3/library/dis.html)

使用 dis 输出 python 字节码

[![](assets/1701612121-39830716a022cb451946ac6e4f5164c8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214401-486907e0-4284-1.png)

输出格式如下

```plain
源码行号 | 指令在函数中的偏移 | 指令符号 | 指令参数 | 实际参数值
```

\*python2 的偏移是 3 的倍数，python3 是 2 的倍数

对于 pyc 文件，也可以通过如下代码将其输出为字节码

```plain
import dis
import marshal
f=open("printname.pyc","rb")
b_data=f.read()
f.close()
PyCodeObjectData=b_data[8:]#反序列化
#python3 版本的需要去除前 16 个字节，python2 版本的需要去除前 8 个字节
Pyobj=marshal.loads(PyCodeObjectData)
dis.dis(Pyobj)
```

#### 解题方法

就是阅读题

可以通过查询文档的方法人工编译

[![](assets/1701612121-4e00e6a1380b285d3da1d7a831b85471.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214435-5c836432-4284-1.png)

或者利用人工智能（chatgptyyds）

[![](assets/1701612121-6daa0f6d0078070ce88b24b4fae7d3d8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214501-6c7dedbc-4284-1.png)

### pyc 加花

**原理**：在 opcode 中添加一句加载超过范围的变量指令，而后再添加一句跳转指令跳过，达成不影响执行而且能欺骗反编译工具的目的

#### 读取 co\_code 长度

co\_code：字节码指令序列，字节码都由操作码 opcode 和参数 opatg 组成的序列。记录着指令数量，指令的增加和减少都会影响该值。

```plain
根据文档，文件由一个 16 字节的标头（4 个 32 位字）和一个可变大小的有效负载组成。pyc

4B 幻数：用于区分哪个版本的 Python 生成了文件。见 https://docs.python.org/3/library/importlib.html#importlib.util.MAGIC_NUMBER
4B 位字段（PEP0552）。至少在 3.7.3 上，这是空的。
4B 修改日期：源文件时间戳
4B 文件大小：源文件大小
从字节 16 开始，有效负载存储编组代码对象（https://docs.python.org/3/c-api/code.html）。

代码对象提供以下属性（以及更多属性）：

co_argcount 参数数（不包括 * 或 ** 参数）
co_code 原始编译字节码的字符串
co_consts 字节码中使用的常量元组
co_filename 在其中创建此代码对象的文件的名称
co_firstlineno Python 源代码中第一行的数量
co_flags 位图：1=优化 |2=新本地 |4=*参数 |8=**参数
co_lnotab 行号到字节码索引的编码映射
定义此代码对象所用的 co_name 名称
co_names 局部变量名称元组
co_nlocals 局部变量的数量
需要 co_stacksize 虚拟机堆栈空间
co_varnames 参数和局部变量名称的元组
co_consts 是可以包含代码对象实例的嵌套数据结构
```

使用`len(code.co_code)`指令即可打印

[![](assets/1701612121-d8f9c88739f30e3c0080a550b12a4e2d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214526-7b5b6940-4284-1.png)

写一个简单的测试程序，用 python3.8 编译成 pyc 文件

```plain
def main():
    a = 40
    b = 57
    b = b + 3
    c = a + b
    print(chr(c))

if __name__ == '__main__':
    main()
```

使用如下指令打印字节码

```plain
import marshal,dis
f=open("kkkk.pyc","rb").read()#注意如果不是在当前目录下时要用\\分隔，如"C:\\Users\\admin\\Desktop\\kkkk.pyc"
f
code = marshal.loads(f[16:])
dis.dis(code)
  1           0 LOAD_CONST               0 (<code object main at 0x0000018C4525A0E0, file "kkkk.py", line 1>)
              2 LOAD_CONST               1 ('main')
              4 MAKE_FUNCTION            0
              6 STORE_NAME               0 (main)

  8           8 LOAD_NAME                1 (__name__)
             10 LOAD_CONST               2 ('__main__')
             12 COMPARE_OP               2 (==)
             14 POP_JUMP_IF_FALSE       22

  9          16 LOAD_NAME                0 (main)
             18 CALL_FUNCTION            0
             20 POP_TOP
        >>   22 LOAD_CONST               3 (None)
             24 RETURN_VALUE

Disassembly of <code object main at 0x0000018C4525A0E0, file "kkkk.py", line 1>:
  2           0 LOAD_CONST               1 (40)
              2 STORE_FAST               0 (a)

  3           4 LOAD_CONST               2 (57)
              6 STORE_FAST               1 (b)

  4           8 LOAD_FAST                1 (b)
             10 LOAD_CONST               3 (3)
             12 BINARY_ADD
             14 STORE_FAST               1 (b)

  5          16 LOAD_FAST                0 (a)
             18 LOAD_FAST                1 (b)
             20 BINARY_ADD
             22 STORE_FAST               2 (c)

  6          24 LOAD_GLOBAL              0 (print)
             26 LOAD_GLOBAL              1 (chr)
             28 LOAD_FAST                2 (c)
             30 CALL_FUNCTION            1
             32 CALL_FUNCTION            1
             34 POP_TOP
             36 LOAD_CONST               0 (None)
             38 RETURN_VALUE
>>> len(code.co_code)
26
```

#### 去花并修改 co\_code 长度

接下来尝试去花，使用 010editor 或者 winhex 打开 pyc 文件

要在二进制格式中定位指令，我们通过查询对应 opcode.h 中的指令编号来进行，比如我的地址是`"C:\Python311\include\opcode.h"`

### 下面是一个失败的示例：

_我想通过删除`b = b + 3`来改变输出结果，但是出现了错误。可能是 pyc 文件的结构没有想象的那么简单，不能轻松修改。不过操作和方法是可以用来去花的_

[![](assets/1701612121-53da13d0a0d935825f125330e9ff5119.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214618-9a1805f0-4284-1.png)

在打印出一部分字节码后，通过查询花指令对应的十进制编号以及操作数在二进制文件中搜索定位（都要转成 16 进制）。比如：

```plain
4           8 LOAD_FAST                1 (b)
             10 LOAD_CONST               3 (3)
             12 BINARY_ADD
             14 STORE_FAST               1 (b)
在 opcode 中查找到指令
#define BINARY_ADD               23
```

要查询最后的`23 = 0x17`

通过指令（124 = 0x7C,100 = 0x64,23 = 0x17,125 = 0x7D）和操作数（0x01,0x03）定位到它在二进制文件中的位置

[![](assets/1701612121-fad43674c4167df97494932e829508ef.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214855-f78f2cf4-4284-1.png)

定位后删除花指令，注意 python2 的一条指令占 3 位，python3 的一条指令占 2 位，在 010editor 中直接按退格键删除

删除后保存并用 uncompyle6 反编译，发现会失败

[![](assets/1701612121-8902ed27cea950d3c65ba82de80c6c7f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824214957-1c748910-4285-1.png)

删除花指令并不能完成去花，之后我们需要修改 co\_code 的值使其符合修改后的文件内容

搜索 len(code.co\_code) = 26 = 0x1A

[![](assets/1701612121-5cb9264b3b060de1876c57c1ed670ec3.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215034-328c35f4-4285-1.png)

我们删去了 4 个指令，即 2\*4 = 8 位，将其改为 26 - 8 = 18 = 0x12 后保存

_理论上现在可以反编译了，但实操不行_

[![](assets/1701612121-036df90950c8571d959453a1af8b23c6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215103-443c1382-4285-1.png)

成功的操作见题目练习中的\[VNCTF2022\] BabyMaze

## 其他混淆

还有高手？

#### Oxyry Python

[Oxyry Python Obfuscator - 世界上最可靠的 Python 混淆器](https://pyob.oxyry.com/)

仅对源码进行了混淆，查看字节码还是能够看到代码逻辑

#### Pyarmor

[GitHub - dashingsoft/pyarmor: A tool used to obfuscate python scripts, bind obfuscated scripts to fixed machine or expire obfuscated scripts.](https://github.com/dashingsoft/pyarmor)

原理：[PyArmor 原理调研 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/609709232?utm_id=0)

#### pyminfier

[GitHub - liftoff/pyminifier: Pyminifier is a Python code minifier, obfuscator, and compressor.](https://github.com/liftoff/pyminifier)

github 开源混淆器，对类名 / 函数名 / 变量名进行重新命名，甚至能够对部分 Python 常量进行扰乱（仅支持 python3）

#### pyc\_obscure

[GitHub - marryjianjian/pyc\_obscure: a simple obfuscator for pyc](https://github.com/marryjianjian/pyc_obscure)

将垃圾数据插入 PyCodeObject 的 co\_code 以混淆像 uncompyle6 这样的反编译器

### 题目练习

#### 陕西省赛 2023 babypython

有两千多行，其中大多数是无用的干扰代码

[![](assets/1701612121-de1c7bc1ebfccdc1772cd941442bd94a.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215139-59b64ce6-4285-1.png)

图中的部分是有用的，其中有替换和倒序，这样处理一下最后的密文 =1nb0A3b7AUQwB3b84mQ/E0MvJUb+EXbx5TQwF3bt52bAZncsd9c

base64 解一下得到 qglrv@onmlqpA>qmq>mBo3A?Bn<lppA@;lp4nx

```plain
aaa = ['f','l','a','g','{']
bbb = "qglrv@onmlqpA>qmq>mBo3A?Bn<lppA@;lp4nx"
for i in range(len(bbb)):
    print(chr(ord(bbb[i])-3^8),end="")
```

尝试"flag{"为答案开头得到加密逻辑为`(bbb[i])-3^8`

flag{5dcbafe63fbf3b7d8647c1aee650ae9c}

#### \[VNCTF2022\] BabyMaze

下载题目，是一个 pyc 文件

使用 uncompyle6 反编译失败

[![](assets/1701612121-c050d4ce24c5151b0863c98d38212fab.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215255-86feb9f4-4285-1.png)

反编译一开头就是 JUMP\_ABSOLUTE 的花指令

[![](assets/1701612121-3df7fb7545b7eed435a2c6f361eaf5db.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215322-96e10048-4285-1.png)

在 python3.8 的 opcode.h 里找到 JUMP\_ABSOLUTE

```plain
#define JUMP_ABSOLUTE           113
113 = 0x71
```

在 010editor 里找到 0x71，选中 710471067102 退格键删掉

[![](assets/1701612121-ed038332f8a406268e75e3ca508c90bc.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215355-aa8a8934-4285-1.png)

获取 co\_code 的值，在 010editor 里搜索并减去 2\*3 = 6

[![](assets/1701612121-4b145abc6589de21b6609832e4d0464d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215421-ba009eb2-4285-1.png)

这样操作之后保存，就可以正常打开了

[![](assets/1701612121-3aaa7cbae912b1810d1244a690328ea4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215459-d0ad1c08-4285-1.png)

反编译一下

```plain
#!/usr/bin/env python
# visit https://tool.lu/pyc/ for more information
# Version: Python 3.8

_map = 
        ......

def maze():
    x = 1
    y = 1
    step = input()
    for i in range(len(step)):
        if step[i] == 'w':
            x -= 1
        elif step[i] == 's':
            x += 1
        elif step[i] == 'a':
            y -= 1
        elif step[i] == 'd':
            y += 1
        else:
            return False
        if None[x][y] == 1:
            return False
        if None == 29 and y == 29:
            return True
        return None


def main():
    print('Welcome To VNCTF2022!!!')
    print('Hello Mr. X, this time your mission is to get out of this maze this time.(FIND THAT 7!)')
    print('you are still doing the mission alone, this tape will self-destruct in five seconds.')
    if maze():
        print('Congratulation! flag: VNCTF{md5(your input)}')
    else:
        print("Sorry, we won't acknowledge the existence of your squad.")

if __name__ == '__main__':
    main()
```

使用 DFS 或者手撕均可

```plain
#出题人的脚本：
map = [[1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1], [1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1], [1, 0, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1], [1, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1, 0, 1], [1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 0, 1, 0, 1, 0, 1], [1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1], [1, 1, 1, 0, 1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1], [1, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1], [1, 0, 1, 1, 1, 0, 1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 0, 1, 1, 1, 0, 1, 0, 1, 0, 1], [1, 0, 0, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1], [1, 1, 1, 1, 1, 1, 1, 0, 1, 0, 1, 1, 1, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 0, 1, 0, 1, 1, 1, 0, 1], [1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1], [1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 0, 1, 0, 1, 1, 1, 0, 1, 0, 1, 1, 1], [1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1], [1, 0, 1, 1, 1, 1, 1, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1], [1, 0, 1, 0, 0, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1], [1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 1, 0, 1], [1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0, 1], [1, 0, 1, 0, 1, 1, 1, 0, 1, 0, 1, 1, 1, 0, 1, 1, 1, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 0, 1, 1, 1], [1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0, 0, 0, 0, 0, 1, 0, 1, 0, 1, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1], [1, 0, 1, 1, 1, 0, 1, 0, 1, 0, 1, 0, 1, 1, 1, 1, 1, 0, 1, 0, 1, 0, 1, 1, 1, 0, 1, 0, 1, 0, 1], [1, 0, 1, 0, 0, 0, 1, 0, 1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1], [1, 0, 1, 0, 1, 1, 1, 0, 1, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1], [1, 0, 1, 0, 0, 0, 1, 0, 1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1], [1, 0, 1, 1, 1, 0, 1, 0, 1, 0, 1, 1, 1, 1, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1, 0, 1], [1, 0, 0, 0, 1, 0, 1, 0, 1, 0, 0, 0, 0, 0, 1, 0, 1, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1, 0, 1, 0, 1], [1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 0, 1], [1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1], [1, 0, 1, 1, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 1, 0, 1, 1, 1, 1, 1, 1, 1, 0, 1, 1, 1, 1, 1, 0, 1], [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1], [1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1]]
usedmap = [[0 for i in range(len(map))] for i in range(len(map)) ]
flag = ''

def DFS(x, y):
    global flag
    if x == len(map) - 2 and y == len(map) - 2:         # 31x31 的迷宫 终点也就是数组的 (29, 29)
        print(flag)
        return

    if map[x + 1][y] == 0 and  usedmap[x + 1][y] == 0:  # 如果下一步不是墙 且没走过
        usedmap[x][y] = 1                               # 标记当前坐标走过（不是下一步）
        flag += 's'
        DFS(x + 1, y)                                   # 尝试向下走
        flag = flag[:-1]                                # 回溯到这说明这条路不可行 所以去掉's' 
        usedmap[x][y] = 0                               # 再设置当前坐标为0 重新找路

    if map[x - 1][y] == 0 and usedmap[x - 1][y] == 0:
        usedmap[x][y] = 1
        flag += 'w'
        DFS(x - 1, y)
        flag = flag[:-1]
        usedmap[x][y] = 0

    if map[x][y + 1] == 0 and usedmap[x][y + 1] == 0:
        usedmap[x][y] = 1
        flag += 'd'
        DFS(x, y + 1)
        flag = flag[:-1]
        usedmap[x][y] = 0

    if map[x][y - 1] == 0 and usedmap[x][y - 1] == 0:
        usedmap[x][y] = 1
        flag += 'a'
        DFS(x, y - 1)
        flag = flag[:-1]
        usedmap[x][y] = 0

print("path:")
x = 1       # 设置起始坐标
y = 1
DFS(x, y)
```

[![](assets/1701612121-0abde89bb3f109699f18922524fae952.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215535-e67aefd8-4285-1.png)

flag{801f190737434100e7d2790bd5b0732e}

### 搭建 python 虚拟环境

##### virtualenv

使用 Python 的第三方库**virtualenv**创建虚拟环境

```plain
pip install virtualenv
```

使用 virtualenv 中的命令来创建虚拟环境的话，会默认同时在 C 盘用户目录中也创建一个环境，采用 virtualenvwrapper-win 可以更好的自定义安装

virtualenvwrapper-win 是一个基于 virtualenv 开发的工具包。

```plain
pip install virtualenvwrapper-win
#需要先安装 virtualenv
```

##### 为虚拟环境配置环境变量

[![](assets/1701612121-d7825ea8637e8682976213d9412c9f5b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215607-f9697dee-4285-1.png)

新建系统变量**WORKON\_HOME**，变量值为想要安装环境的位置

##### 新建虚拟环境

```plain
mkvirtualenv -p python_path env_name
```

\-p: 指定根据哪个 python 创建新的虚拟环境，一般是期望 python 的 exe 可执行文件的路径 env\_name: 是创建虚拟环境的名字

记得先授予权限（上面的报错就是没授予权限导致的）

[![](assets/1701612121-215f3b0eca535ea41f0be0c51d7db3de.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215634-094493a2-4286-1.png)

创建好虚拟环境后，会自动跳转到该环境目录下，路径前面有`(py38)`。而且系统会默认为新建的虚拟环境配置电脑中安装好的 Python 环境（只包括部分执行命令，没有相关第三方模块），当然我们也可以在新虚拟环境中重新安装 Python。

##### 查看虚拟环境配置情况

cmd 中运行`workon`或者`lsvirtualenv`即可查看当前的虚拟环境配置情况

##### 激活虚拟环境

```plain
先 cd 转换
activate
```

##### 退出虚拟环境

```plain
deactivate
```

退出后路径前的（）就没有了

##### 删除虚拟环境

```plain
rmvirtualenv
```

进入当前虚拟环境的包目录 (site-packages)

```plain
cdsitepackages
```

直接显示当前虚拟环境下所有包

```plain
lssitepackages
```

#### 直接使用 virtualenv

指定 python 版本并生成环境：`virtualenv [name] --python=pythonx.x.x`

进入此虚拟环境（win)：`.\[name]\Scripts\activate`

退出虚拟环境：`deactivate`

[![](assets/1701612121-7eddda1847cad1a1ba49549ff3e8fe70.png)](https://xzfile.aliyuncs.com/media/upload/picture/20230824215712-202b3652-4286-1.png)

（菜鸡一只，有什么问题还望各位大牛指导 Orz）
