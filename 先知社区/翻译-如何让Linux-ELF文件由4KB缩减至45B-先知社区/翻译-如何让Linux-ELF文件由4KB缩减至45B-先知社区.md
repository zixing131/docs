

# 【翻译】如何让 Linux ELF 文件由 4KB 缩减至 45B！ - 先知社区

【翻译】如何让 Linux ELF 文件由 4KB 缩减至 45B！

- - -

A Whirlwind Tutorial on Creating Really Teensy ELF Executables for Linux  
[https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html](https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html)  
本文档更实际的目的是描述 ELF 文件格式和 Linux 操作系统的一些内部工作原理。  
使用汇编 Nasm 来写：[http://www.nasm.us/](http://www.nasm.us/)  
注意：国际赛 2024BraekerCTF 二进制安全题目采用了该文章，并出了很多有意思的考点，希望本篇文章对你有帮助

## 1, To Start

我们需要一个程序，越小越好，我们对如何让程序变得更小很感兴趣，而不是程序能干什么。

```plain
/* tiny.c */
  int main(void) { return 42; }
```

所以，这将是我们第一个程序，当然是第一版  
编译并运行它：

```plain
$ gcc -Wall tiny.c
  $ ./a.out ; echo $?
  42
```

所以，它有多大呢？

```plain
$ wc -c a.out
     3998 a.out
```

有点大了，进一步优化

```plain
$ gcc -Wall -s tiny.c
  $ ./a.out ; echo $?
  42
  $ wc -c a.out
     2632 a.out
```

确实有了提高

```plain
$ gcc -Wall -s -O3 tiny.c
  $ wc -c a.out
     2616 a.out
```

诚然，在高级语言层面不能再进行优化了，因此我们打算使用汇编来进行优化。  
第二版程序：  
我们需要做的就是从 main() 返回 42。在汇编语言中，这意味着函数应该将累加器 eax 设置为 42，然后返回

```plain
; tiny.asm
  BITS 32
  GLOBAL main
  SECTION .text
  main:
                mov     eax, 42
                ret
```

生成

```plain
$ nasm -f elf tiny.asm
  $ gcc -Wall -s tiny.o
  $ ./a.out ; echo $?
  42
```

so，它现在有多大呢？

```plain
$ wc -c a.out
     2604 a.out
```

看来我们只少了 12 个字节 C 语言自动产生的所有额外开销？  
问题是我们使用 main() 接口仍然会产生大量开销。链接器仍在为我们向操作系统添加一个接口，而正是该接口实际调用了 main()。那么如果我们不需要它，我们该如何解决这个问题呢？  
链接器默认使用的实际入口点是名为\_start 的符号。当我们与 gcc 链接时，它自动包含一个\_start 例程，该例程设置 argc 和 argv 等，然后调用 main()  
所以，让我们看看我们是否可以绕过它，并定义我们自己的\_start 例程：

```plain
; tiny.asm
  BITS 32
  GLOBAL _start
  SECTION .text
  _start:
                mov     eax, 42
                ret
```

gcc 会让我们那样做吗？

```plain
$ nasm -f elf tiny.asm
  $ gcc -Wall -s tiny.o
  tiny.o(.text+0x0): multiple definition of `_start'
  /usr/lib/crt1.o(.text+0x0): first defined here
  /usr/lib/crt1.o(.text+0x36): undefined reference to `main'
```

事实上，会的，但首先我们得学会如何来做。  
添加一个选项：-nostartfiles  
:::info  
\-nostartfiles  
Do not use the standard system startup files when linking. The standard libraries are used normally.  
:::  
但是结果却不正确

```plain
$ nasm -f elf tiny.asm
  $ gcc -Wall -s -nostartfiles tiny.o
  $ ./a.out ; echo $?
  Segmentation fault
  139
```

错误的地方在于我们把\_start 当作一个 C 函数，并试图从它返回。实际上，它根本不是一个函数。它只是目标文件中的一个符号，链接器使用它来定位程序的入口点。当我们的程序被调用时，它被直接调用。如果我们去看，我们会看到堆栈顶部的值是数字 1，这肯定是非常不像地址的。实际上，堆栈上的是我们程序的 argc 值。之后是 argv 数组的元素，包括终止 NULL 元素，后面是 envp 的元素。仅此而已堆栈上没有返回地址。

我们再来一次我们将调用\_exit()，这是一个接受单个整数参数的函数。所以我们需要做的就是将数字压入堆栈并调用函数。(还需要将\_exit() 声明为 external)

```plain
; tiny.asm
  BITS 32
  EXTERN _exit
  GLOBAL _start
  SECTION .text
  _start:
                push    dword 42
                call    _exit
```

编译，然后运行

```plain
$ nasm -f elf tiny.asm
  $ gcc -Wall -s -nostartfiles tiny.o
  $ ./a.out ; echo $?
  42
```

成功了！

```plain
$ wc -c a.out
     1340 a.out
```

那么 GCC 还有没有其他选项呢？  
:::info  
\-nostdlib  
Don't use the standard system libraries and startup files when linking. Only the files you specify will be passed to the linker.  
链接时不要使用标准的系统库和启动文件。只有您指定的文件才会传递给链接器。  
:::

```plain
$ gcc -Wall -s -nostdlib tiny.o
  tiny.o(.text+0x6): undefined reference to `_exit'
```

\_exit() 是系统函数！  
如果我们愿意放弃所有可移植性，我们可以让程序退出，而不必与其他任何东西链接。首先，我们需要知道如何在 Linux 下进行系统调用。

## 2, system call under Linux

与大多数操作系统一样，Linux 通过系统调用为其托管的程序提供基本的必需品。这包括打开文件、阅读和写入文件句柄 - 当然还有关闭进程。  
Linux 系统调用接口是一条指令：int 0x80。所有的系统调用都是通过这个中断完成的。要进行系统调用，eax 应该包含一个数字，指示调用的是哪个系统调用，如果有参数，其他寄存器用于保存参数。如果系统调用只带一个参数，那么它将在 ebx 中;带两个参数的系统调用将使用 ebx 和 ecx。同样，如果需要第三、第四或第五个参数，则分别使用 edx、esi 和 edi。从系统调用返回时，eax 将包含返回值。如果发生错误，eax 将包含一个负值，绝对值表示错误。  
不同系统调用的编号列在 /usr/include/asm/unistd.h中。快速浏览一下会告诉我们，exit系统调用被分配了数字1。像C函数一样，它有一个参数，返回给父进程的值，因此这将进入ebx。

```plain
; tiny.asm
  BITS 32
  GLOBAL _start
  SECTION .text
  _start:
                mov     eax, 1
                mov     ebx, 42  
                int     0x80
```

编译 and 运行

```plain
$ nasm -f elf tiny.asm
  $ gcc -Wall -s -nostdlib tiny.o
  $ ./a.out ; echo $?
  42
```

多大呢？

```plain
$ wc -c a.out
      372 a.out
```

已经缩小至 4 分之 1 了！  
还能不能再小呢？  
观察汇编代码：

```plain
00000000 B801000000        mov        eax, 1
  00000005 BB2A000000        mov        ebx, 42
  0000000A CD80              int        0x80
```

哎呀，我们不需要初始化所有的 ebx，因为操作系统只会使用最低的字节。单独设置 bl 就足够了，它将占用两个字节而不是五个字节。  
我们也可以通过将 eax 异或为零，然后使用一个字节的增量指令来将其设置为 1;这将多保存保存两个字节。

```plain
00000000 31C0              xor        eax, eax
  00000002 40                inc        eax
  00000003 B32A              mov        bl, 42
  00000005 CD80              int        0x80
```

顺便说一句，我们也可以停止使用 gcc 来链接我们的可执行文件，因为我们没有使用它的任何附加功能，而只是自己调用链接器 ld：

```plain
$ nasm -f elf tiny.asm
  $ ld -s tiny.o
  $ ./a.out ; echo $?
  42
  $ wc -c a.out
      368 a.out
```

小了四个字节。(Hey！我们不是少了五个字节吗？我们确实这样做了，但是 ELF 文件中的对齐考虑导致它需要额外的填充字节。）  
所以...我们到终点了吗？我们只能走这么小吗？嗯，嗯。我们的程序现在有 7 个字节长。ELF 文件真的需要 361 字节的开销吗？文件里到底有什么？我们可以使用 objdump 查看文件的内容：

```plain
$ objdump -x a.out | less
```

输出看起来像乱码

```plain
Sections:
  Idx Name          Size      VMA       LMA       File off  Algn
    0 .text         00000007  08048080  08048080  00000080  2**4
                    CONTENTS, ALLOC, LOAD, READONLY, CODE
    1 .comment      0000001c  00000000  00000000  00000087  2**0
                    CONTENTS, READONLY
```

完整的.text 部分被列出为七个字节长，正如我们指定的那样。因此，似乎可以得出结论，我们现在完全控制了程序的机器语言内容。  
但是还有一个部分叫做“.comment”。谁下的命令它甚至有 28 个字节长！  
.comment 节被列为位于文件偏移量 00000087（十六进制）。如果我们使用 hexdump 程序来查看文件的该区域，我们将看到：

```plain
00000080: 31C0 40B3 2ACD 8000 5468 6520 4E65 7477  1.@.*...The Netw
  00000090: 6964 6520 4173 7365 6D62 6C65 7220 302E  ide Assembler 0.
  000000A0: 3938 0000 2E73 796D 7461 6200 2E73 7472  98...symtab..str
```

难道是 Nasm！我们换个汇编器  
Well, well, well. Who'd've thought that Nasm would undermine our quest like this? Maybe we should switch to using gas, AT&T syntax notwithstanding....  
Alas, if we do:

```plain
; tiny.s
  .globl _start
  .text
  _start:
                xorl    %eax, %eax
                incl    %eax
                movb    $42, %bl
                int     $0x80
```

我们发现没有改变

```plain
$ gcc -s -nostdlib tiny.s
  $ ./a.out ; echo $?
  42
  $ wc -c a.out
      368 a.out
```

实际上，是有区别的，再次 objdump

```plain
Sections:
  Idx Name          Size      VMA       LMA       File off  Algn
    0 .text         00000007  08048074  08048074  00000074  2**2
                    CONTENTS, ALLOC, LOAD, READONLY, CODE
    1 .data         00000000  0804907c  0804907c  0000007c  2**2
                    CONTENTS, ALLOC, LOAD, DATA
    2 .bss          00000000  0804907c  0804907c  0000007c  2**2
                    ALLOC
```

没有 .comment 节，但现在我们有两个无用的部分来存储我们不存在的数据。即使这些部分的长度为零字节，它们也会产生开销，使我们的文件大小毫无理由地增加。  
好吧，那么这些开销到底是什么，我们如何摆脱它？  
为了回答这些问题，我们必须开始深入了解文件格式。我们需要了解 ELF 格式。

## 3, the ELF format

:::info  
The canonical document describing the ELF format for Intel-386 architectures can be found at [http://refspecs.linuxbase.org/elf/elf.pdf](http://refspecs.linuxbase.org/elf/elf.pdf). (You can also find a flat-text version of version 1.0 of the standard at [http://www.muppetlabs.com/~breadbox/software/ELF.txt](http://www.muppetlabs.com/~breadbox/software/ELF.txt).) This specification covers a lot of territory, so if you'd prefer to not read the whole thing yourself, I'll understand. Basically, here's what we need to know:  
:::  
关于详细内容，不是本文讨论的目的  
每个 ELF 文件都以一个称为 ELF 头的结构开始。这个结构有 52 个字节长，包含几条描述文件内容的信息。例如，前 16 个字节包含一个“标识符”，其中包括文件的魔术签名（7F 45 4C 46），以及一些指示内容是 32 位或 64 位、little-endian 或 big-endian 等的单字节标志。ELF 文件是可执行文件、目标文件还是共享对象库;程序的起始地址;以及**程序头表**和**节头表**在文件中的位置。  
the program header table（程序表）and the section header table（节头表）.  
这两个表可以出现在文件中的任何地方，但通常前者出现在 ELF 头之后，而后者出现在文件末尾或附近。这两个表的作用类似，因为它们标识文件的组成部分。然而，**节头表更侧重于识别程序的各个部分在文件中的位置**，**而程序头表描述了这些部分在哪里以及如何加载到内存中**。  
简而言之，**节头表供编译器和链接器使用，而程序头表供程序加载器使用**。  
程序头表对于目标文件是可选的，实际上是不存在的。同样，节头表对于可执行文件是可选的，、但几乎总是存在的！  
这就是我们第一个问题的答案在我们的程序中，一个相当大的开销是一个完全不必要的节头表，也许还有一些同样无用的节，它们对我们程序的内存映像没有贡献。  
所以，我们转向第二个问题：我们如何摆脱这一切？  
我们只能靠自己了。没有一个标准工具会屈尊使一个可执行文件没有某种类型的节头表。如果我们想要这样的东西，我们必须自己做。  
不过，这并不意味着我们必须拿出一个二进制编辑器，手工编写十六进制值。  
Nasm 有一个 flat binary output format，这将很好地为我们服务。  
我们现在所需要的是一个空的 ELF 可执行文件的图像，我们可以用我们的程序填充它。  
我们可以查看 ELF 规范和/usr/include/linux/elf.h，以及由标准工具创建的可执行文件，以确定空的 ELF 可执行文件应该是什么样子。

```plain
BITS 32

                org     0x08048000

  ehdr:                                                 ; Elf32_Ehdr
                db      0x7F, "ELF", 1, 1, 1, 0         ;   e_ident
        times 8 db      0
                dw      2                               ;   e_type
                dw      3                               ;   e_machine
                dd      1                               ;   e_version
                dd      _start                          ;   e_entry
                dd      phdr - $$                       ;   e_phoff
                dd      0                               ;   e_shoff
                dd      0                               ;   e_flags
                dw      ehdrsize                        ;   e_ehsize
                dw      phdrsize                        ;   e_phentsize
                dw      1                               ;   e_phnum
                dw      0                               ;   e_shentsize
                dw      0                               ;   e_shnum
                dw      0                               ;   e_shstrndx

  ehdrsize      equ     $ - ehdr

  phdr:                                                 ; Elf32_Phdr
                dd      1                               ;   p_type
                dd      0                               ;   p_offset
                dd      $$                              ;   p_vaddr
                dd      $$                              ;   p_paddr
                dd      filesize                        ;   p_filesz
                dd      filesize                        ;   p_memsz
                dd      5                               ;   p_flags
                dd      0x1000                          ;   p_align

  phdrsize      equ     $ - phdr

  _start:

  ; your program here

  filesize      equ     $ - $$
```

此映像包含一个 ELF 头，将文件标识为 Intel 386 可执行文件，没有节头表，程序头表包含一个条目。所述条目指示程序加载器从存储器地址 0x08048000（这是可执行文件加载的默认地址）开始将整个文件加载到存储器中（程序的正常行为是将其 ELF 头和程序头表包括在其存储器映像中），并在\_start 处开始执行代码，该代码立即出现在程序头表之后。没有数据段，没有数据段，没有.comment  
加入我们的小汇编代码

```plain
; tiny.asm
                org     0x08048000

  ;
  ; (as above)
  ;


  _start:
                mov     bl, 42
                xor     eax, eax
                inc     eax
                int     0x80

  filesize      equ     $ - $$
```

并尝试运行

```plain
$ nasm -f bin -o a.out tiny.asm
  $ chmod +x a.out
  $ ./a.out ; echo $?
  42
```

观察程序的大小

```plain
$ wc -c a.out
       91 a.out
```

91 个字节。不到我们上次尝试的四分之一，不到我们第一次尝试的四十分之一！  
更重要的是，这次我们可以解释每一个字节。我们确切地知道可执行文件中有什么，以及为什么需要它。这就是极限。不能再小了。  
事实的如此吗？

## 4、更进一步

如果您真的停下来阅读 ELF 规范，您可能会注意到一些事实。  
1)ELF 文件的不同部分允许位于任何地方（除了 ELF 头，它必须在文件的顶部），它们甚至可以相互重叠。  
2) 头文件中的某些字段实际上并没有被使用。  
特别是，我想到的是 16 字节标识字段末尾的零字符串。它们是纯粹的填充，为 ELF 标准的未来扩展腾出空间。所以操作系统根本不应该关心里面有什么。我们已经把所有东西都加载到内存中了，我们的程序只有 7 个字节长。  
我们可以把代码放在 ELF 头里面。

```plain
; tiny.asm

  BITS 32

                org     0x08048000

  ehdr:                                                 ; Elf32_Ehdr
                db      0x7F, "ELF"                     ;   e_ident
                db      1, 1, 1, 0, 0
  _start:       mov     bl, 42
                xor     eax, eax
                inc     eax
                int     0x80
                dw      2                               ;   e_type
                dw      3                               ;   e_machine
                dd      1                               ;   e_version
                dd      _start                          ;   e_entry
                dd      phdr - $$                       ;   e_phoff
                dd      0                               ;   e_shoff
                dd      0                               ;   e_flags
                dw      ehdrsize                        ;   e_ehsize
                dw      phdrsize                        ;   e_phentsize
                dw      1                               ;   e_phnum
                dw      0                               ;   e_shentsize
                dw      0                               ;   e_shnum
                dw      0                               ;   e_shstrndx

  ehdrsize      equ     $ - ehdr

  phdr:                                                 ; Elf32_Phdr
                dd      1                               ;   p_type
                dd      0                               ;   p_offset
                dd      $$                              ;   p_vaddr
                dd      $$                              ;   p_paddr
                dd      filesize                        ;   p_filesz
                dd      filesize                        ;   p_memsz
                dd      5                               ;   p_flags
                dd      0x1000                          ;   p_align

  phdrsize      equ     $ - phdr

  filesize      equ     $ - $$
```

查看大小

```plain
$ nasm -f bin -o a.out tiny.asm
  $ chmod +x a.out
  $ ./a.out ; echo $?
  42
  $ wc -c a.out
       84 a.out
```

现在我们真的已经尽可能低了。我们的文件正好与一个 ELF 头和一个程序头表条目一样长，这两个都是我们绝对需要的，以便加载到内存中并运行。所以现在没什么可减的了！  
那么，如果我们可以对**程序表**与刚才对 ELF 头部执行的相同的操作呢？让它与 ELF 头重叠，有可能吗？  
请注意，ELF 头中的最后八个字节与程序头表中的前八个字节具有某种相似性。一种可以被描述为“相同”的相似性。

```plain
; tiny.asm

  BITS 32

                org     0x08048000

  ehdr:
                db      0x7F, "ELF"             ; e_ident
                db      1, 1, 1, 0, 0
  _start:       mov     bl, 42
                xor     eax, eax
                inc     eax
                int     0x80
                dw      2                       ; e_type
                dw      3                       ; e_machine
                dd      1                       ; e_version
                dd      _start                  ; e_entry
                dd      phdr - $$               ; e_phoff
                dd      0                       ; e_shoff
                dd      0                       ; e_flags
                dw      ehdrsize                ; e_ehsize
                dw      phdrsize                ; e_phentsize
  phdr:         dd      1                       ; e_phnum       ; p_type
                                                ; e_shentsize
                dd      0                       ; e_shnum       ; p_offset
                                                ; e_shstrndx
  ehdrsize      equ     $ - ehdr
                dd      $$                                      ; p_vaddr
                dd      $$                                      ; p_paddr
                dd      filesize                                ; p_filesz
                dd      filesize                                ; p_memsz
                dd      5                                       ; p_flags
                dd      0x1000                                  ; p_align
  phdrsize      equ     $ - phdr

  filesize      equ     $ - $$
```

可以肯定的是，Linux 一点也不介意我们的吝啬：

```plain
$ nasm -f bin -o a.out tiny.asm
  $ chmod +x a.out
  $ ./a.out ; echo $?
  42
  $ wc -c a.out
       76 a.out
```

所以：这里是什么是和不是基本的 ELF 头。前四个字节必须包含幻数，否则 Linux 不会处理它。然而，e\_ident 字段中的其他三个字节没有检查，这意味着我们有不少于 12 个连续的字节可以设置为任何值。e\_type 必须设置为 2，以指示可执行文件，而 e\_machine 必须设置为 3，正如刚才提到的。与 e\_ident 中的版本号一样，e\_version 完全被忽略。（这是可以理解的，因为目前只有一个版本的 ELF 标准。e\_entry 自然必须是有效的，因为它指向程序的开始。显然，e\_phoff 需要包含文件中程序头表的正确偏移量，e\_phnum 需要包含该表中正确数量的条目。然而，e\_flags 被记录为目前未被 Intel 使用，因此我们应该可以免费重用它。e\_ehsize 应该用于验证 ELF 头是否具有预期的大小，但 Linux 对此并不在意。e\_phentsize 同样用于验证程序标题表条目的大小。在旧的内核中，这一项是未检查的，但现在需要正确设置。ELF 头中的其他内容都是关于节头表的，它不适用于可执行文件。  
总而言之，仔细检查一下就会发现，ELF 头中的大多数必要字段都在前半部分 - 后半部分几乎完全可以自由使用。考虑到这一点，我们可以比以前更多地对这两个结构进行优化：

```plain
; tiny.asm

  BITS 32

                org     0x00200000

                db      0x7F, "ELF"             ; e_ident
                db      1, 1, 1, 0, 0
  _start:
                mov     bl, 42
                xor     eax, eax
                inc     eax
                int     0x80
                dw      2                       ; e_type
                dw      3                       ; e_machine
                dd      1                       ; e_version
                dd      _start                  ; e_entry
                dd      phdr - $$               ; e_phoff
  phdr:         dd      1                       ; e_shoff       ; p_type
                dd      0                       ; e_flags       ; p_offset
                dd      $$                      ; e_ehsize      ; p_vaddr
                                                ; e_phentsize
                dw      1                       ; e_phnum       ; p_paddr
                dw      0                       ; e_shentsize
                dd      filesize                ; e_shnum       ; p_filesz
                                                ; e_shstrndx
                dd      filesize                                ; p_memsz
                dd      5                                       ; p_flags
                dd      0x1000                                  ; p_align

  filesize      equ     $ - $$
```

正如您（希望）看到的那样，程序头表的前 20 个字节现在与 ELF 头的最后 20 个字节重叠。实际上，这两者非常吻合。在重叠区域内，ELF 头只有两个部分很重要。第一个是 e\_phnum 字段，它恰好与 p\_paddr 字段重合，p\_paddr 字段是程序头表中绝对被忽略的少数字段之一。另一个是 e\_phentsize 字段，它与 p\_vaddr 字段的上半部分重合。这些是通过为我们的程序选择一个非标准加载地址来匹配的，上半部分等于 0x0020。  
它有多大呢？

```plain
$ nasm -f bin -o a.out tiny.asm
  $ chmod +x a.out
  $ ./a.out ; echo $?
  42
  $ wc -c a.out
       64 a.out
```

程序短了 12 个字节，与预测完全一样。  
我们不能在不遇到无望的障碍的情况下再向上移动 12 个字节，试图协调两个结构中的几个字段。唯一的另一种可能性是让它在前四个字节之后立即开始。这使得程序标题表的第一部分舒适地在 e\_ident 区域内，但仍然给其余部分留下问题。经过一些实验，看起来这是不可能的。  
事实证明，程序头表中还有几个字段我们可以修改。

我们注意到，p\_memsz 表示为内存段分配多少内存。显然，它至少需要和 p\_filesz 一样大，但是如果它更大也不会有任何坏处。毕竟，仅仅因为我们要求内存并不意味着我们必须使用它。  
其次，事实证明，与我所有的预期相反，可执行位可以从 p\_flags 字段中删除。事实证明，可读和可执行位是多余的：任何一个都会包含另一个。  
因此，考虑到这些事实，我们可以将文件重新组织成这个程序：

```plain
; tiny.asm

  BITS 32

                org     0x00010000

                db      0x7F, "ELF"             ; e_ident
                dd      1                                       ; p_type
                dd      0                                       ; p_offset
                dd      $$                                      ; p_vaddr 
                dw      2                       ; e_type        ; p_paddr
                dw      3                       ; e_machine
                dd      _start                  ; e_version     ; p_filesz
                dd      _start                  ; e_entry       ; p_memsz
                dd      4                       ; e_phoff       ; p_flags
  _start:
                mov     bl, 42                  ; e_shoff       ; p_align
                xor     eax, eax
                inc     eax                     ; e_flags
                int     0x80
                db      0
                dw      0x34                    ; e_ehsize
                dw      0x20                    ; e_phentsize
                dw      1                       ; e_phnum
                dw      0                       ; e_shentsize
                dw      0                       ; e_shnum
                dw      0                       ; e_shstrndx

  filesize      equ     $ - $$
```

p\_flags 字段已从 5 更改为 4，正如我们注意到的那样，我们可以侥幸逃脱。这个 4 也是 e\_phoff 字段的值，它给出了程序头表文件的偏移量，这正是我们找到它的位置。程序（还记得吗？）已下移到 ELF 头的较低部分，从 e\_shoff 字段开始，在 e\_flags 字段内结束。  
请注意，加载地址已被更改为更小的数字——实际上是尽可能的低。这使得 e\_entry 字段中的值保持在一个相当小的数字，这很好，因为它也是 p\_memsz 的数字。（实际上，对于虚拟内存来说，这并不重要——我们可以让它保持原值，它也可以正常工作。但是礼貌一点没有坏处。）  
对 p\_filesz 的更改可能需要解释。因为我们没有在 p\_flags 字段中设置写入位，Linux 不允许我们定义大于 p\_filesz 的 p\_memsz 值，因为如果这些额外的字节不可写，它就不能对它们进行零初始化。由于我们不能在不移动程序头表的对齐的情况下更改 p\_flags 字段，您可能认为唯一的解决方案是将 p\_memsz 值降低到等于 p\_filesz（这将使其无法与 e\_entry 共享）。然而，还有另一种解决方案，即将 p\_filesz 增加到等于 p\_memsz。这意味着它们都比实际文件大——实际上要大得多——但它免除了加载程序必须写入只读内存的责任，这是它所关心的。

```plain
$ nasm -f bin -o a.out tiny.asm
  $ chmod +x a.out
  $ ./a.out ; echo $?
  42
  $ wc -c a.out
       52 a.out
```

因此，程序头表和程序本身都完全嵌入在 ELF 头文件中，我们的可执行文件现在与 ELF 头文件完全一样大！不多不少。并且仍然运行，没有来自 Linux 的任何错误！  
现在，终于，我们确实达到了绝对的最低限度。这是毫无疑问的，对吧？毕竟，我们必须有一个完整的 ELF 结构（即使它被严重损坏）  
错。我们还有最后一个肮脏的把戏。  
似乎是这样的，如果文件不是一个完整的 ELF 头的大小，Linux 用零填充缺失的字节。我们在文件末尾有不少于七个零，如果我们从文件图像中删除它们：

```plain
; tiny.asm

  BITS 32

                org     0x00010000

                db      0x7F, "ELF"             ; e_ident
                dd      1                                       ; p_type
                dd      0                                       ; p_offset
                dd      $$                                      ; p_vaddr 
                dw      2                       ; e_type        ; p_paddr
                dw      3                       ; e_machine
                dd      _start                  ; e_version     ; p_filesz
                dd      _start                  ; e_entry       ; p_memsz
                dd      4                       ; e_phoff       ; p_flags
  _start:
                mov     bl, 42                  ; e_shoff       ; p_align
                xor     eax, eax
                inc     eax                     ; e_flags
                int     0x80
                db      0
                dw      0x34                    ; e_ehsize
                dw      0x20                    ; e_phentsize
                db      1                       ; e_phnum
                                                ; e_shentsize
                                                ; e_shnum
                                                ; e_shstrndx

  filesize      equ     $ - $$
```

令人难以置信的是，我们仍然可以生成一个工作的可执行文件：

```plain
$ nasm -f bin -o a.out tiny.asm
  $ chmod +x a.out
  $ ./a.out ; echo $?
  42
  $ wc -c a.out
       45 a.out
```

在这里无法回避这样一个事实，即文件中指定程序头表中条目数的第 45 个字节需要非零，需要存在，并且需要从 ELF 头开始的第 45 个位置。我们被迫得出结论，没有什么可以做的了。

## 5、结论

这个 45 字节的文件不到我们可以使用标准工具创建的最小 ELF 可执行文件的八分之一，也不到我们可以使用纯 C 代码创建的最小文件的五十分之一。我们已经从文件中删除了所有我们能删除的东西，并将大部分我们不能使用的东西用于双重目的。  
我们发现构造的这样的小怪物，Linux 居然承认它的存在，还给进程 ID。另一方面，经过这样的学习，我们发现 Linux 真的太有意思了。
