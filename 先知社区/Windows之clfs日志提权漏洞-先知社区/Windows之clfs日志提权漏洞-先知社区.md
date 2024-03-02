

# Windows 之 clfs 日志提权漏洞 - 先知社区

Windows 之 clfs 日志提权漏洞

- - -

本文是关于历届 Windows - clfs 漏洞的梳理，并不涉及到后续的 exp，因为 CLFS 驱动程序在很短的时间内，爆出过很多漏洞，而且其中很多漏洞牵扯到的函数，clfs 的结构体都是相关的还有针对上一个 CVE 补丁的别样思路的绕过。所以想对此进行一个好的总结和梳理，以便从中学习思路和分析绕过的心路历程。  
关于 clfs 漏洞 在进行梳理前，要认识如下几个点这也是每个 clfs 系列文章老生常谈的知识。

## **前置知识**

首先 Common Log File System(CLFS) API 提供了一个高性能、通用的日志文件子系统，专用的客户机应用程序可以使用该子系统，多个客户机可以共享该子系统来优化日志访问。这是微软对于 CLFS 的诠释，而且微软还提供了一系列的 api 来对日志进行操作，追加，访问等操作。这些 api 也在后续的漏洞利用过程中不可或缺。  
例如：

```plain
CLFSUSER_API HANDLE CreateLogFile(
  [in]           LPCWSTR               pszLogFileName,
  [in]           ACCESS_MASK           fDesiredAccess,
  [in]           DWORD                 dwShareMode,
  [in, optional] LPSECURITY_ATTRIBUTES psaLogFile,
  [in]           ULONG                 fCreateDisposition,
  [in]           ULONG                 fFlagsAndAttributes
);
CreateLogFile(日志文件名字，访问权限[读或写还有删除]，文件的共享模式，指向 SECURITY_ATTRIBUTES 结构的指针，打开/创建新文件，文件的属性和标志)
CreateLogFile(logFileName.c_str(), GENERIC_WRITE | GENERIC_READ, 0, NULL, OPEN_ALWAYS, 0);
```

任何需要日志记录或恢复支持的用户模式应用程序都可以使用 CLFS。CLFS 数据结构大致可以分为三种类型： 
1.存储在内核态的内存中并附加到对象 (例如附加到 File object 的 FCB)。2.临时存储在用户态或内核态调用者的内存中。3.持久化存储在磁盘上的基本日志文件 (BLF)，也就是用户可以通过 CreateLogFile 函数来创建日志文件，然后在本地创建一个同名的后缀为.blf 的日志文件，其中包含了日志存储所需的一些信息。  
其次 LOG\_BLOCK：每个日志块都以一个名为 CLFS\_LOG\_BLOCK\_HEADER 的块头开始的。

```plain
typedef struct _CLFS_LOG_BLOCK_HEADER
{
    UCHAR MajorVersion;
    UCHAR MinorVersion;
    UCHAR Usn;
    CLFS_CLIENT_ID ClientId;
    USHORT TotalSectorCount;
    USHORT ValidSectorCount;
    ULONG Padding;
    ULONG Checksum;
    ULONG Flags;
    CLFS_LSN CurrentLsn;
    CLFS_LSN NextLsn;
    ULONG RecordOffsets[16];
    ULONG SignaturesOffset;
} CLFS_LOG_BLOCK_HEADER, *PCLFS_LOG_BLOCK_HEADER;
by:https://github.com/ionescu007/clfs-docs/
```

其中 RecordOffsets 是日志块中记录的偏移量数组，CLFS 只处理指向 CLFS\_LOG\_BLOCK\_HEADER 末尾的第一个记录的偏移量（0x70)。

```plain
typedef struct _CLFS_LOG_BLOCK_HEADER  
{  
    UCHAR SECTOR_BLOCK_TYPE;  
    UCHAR Usn;  
};
```

当日志文件存储在磁盘上时候，则会对其日志块进行编码处理，在编码的过程中，还有一个由 SignaturesOffset 字段指向的数组。在编码的时候，每个扇区都有一个的签名，来保证一致性。还是就是 Checksum 是该日志块数据的校验和，用来对读取的数据进行校验，采用的是 CRC32 的校验，函数对应：CCrc32::ComputeCrc32

[![](assets/1704182823-74a707f76766273536f08df6c34649a4.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101225237-679cdc62-a8b5-1.png)

Base Record 定义为如下，它存储用于将基本日志文件与容器关联的元数据

```plain
typedef struct _CLFS_BASE_RECORD_HEADER  
{  
    CLFS_METADATA_RECORD_HEADER hdrBaseRecord;  
    CLFS_LOG_ID cidLog;  
    ULONGLONG rgClientSymTbl[CLIENT_SYMTBL_SIZE];  
    ULONGLONG rgContainerSymTbl[CONTAINER_SYMTBL_SIZE];  
    ULONGLONG rgSecuritySymTbl[SHARED_SECURITY_SYMTBL_SIZE];  
    ULONG cNextContainer;  
    CLFS_CLIENT_ID cNextClient;  
    ULONG cFreeContainers;  
    ULONG cActiveContainers;  
    ULONG cbFreeContainers;  
    ULONG cbBusyContainers;  
    ULONG rgClients[MAX_CLIENTS_DEFAULT];  
    ULONG rgContainers[MAX_CONTAINERS_DEFAULT];  
    ULONG cbSymbolZone;  
    ULONG cbSector;  
    USHORT bUnused;  
    CLFS_LOG_STATE eLogState;  
    UCHAR cUsn;  
    UCHAR cClients;  
} CLFS_BASE_RECORD_HEADER, *PCLFS_BASE_RECORD_HEADER;
```

然后其中 cActiveContainers 保存了当前活跃的容器数，rgContainers 数组则保存容器上下文的偏移值。用户可以使用 AddLogContainer 和 AddLogContainerSet 函数向日志中添加容器。容器上下文由以下结构表示： 
typedef struct \_CLFS\_CONTAINER\_CONTEXT

```plain
typedef struct _CLFS_CONTAINER_CONTEXT  
{  
    CLFS_NODE_ID cidNode;  
    ULONGLONG cbContainer;  
    CLFS_CONTAINER_ID cidContainer;  
    CLFS_CONTAINER_ID cidQueue;  
    union  
    {  
    CClfsContainer* pContainer;  
    ULONGLONG ullAlignment;  
    };  
    CLFS_USN usnCurrent;  
    CLFS_CONTAINER_STATE eState;  
    ULONG cbPrevOffset;  
    ULONG cbNextOffset;  
} CLFS_CONTAINER_CONTEXT, *PCLFS_CONTAINER_CONTEXT;  
* pContainer 实际上包含一个内核指针，指向在运行时描述容器的 CClfsContainer
* 当日志文件在磁盘上时，该字段必须设置为零。
```

BLF 文件，也是生成的日志文件。其中组成 blf 的不同元数据对应其不同的类型的数据，要关注一下 Control Block，其包含了有关布局，扩展区域以及截断，区域的信息。其结构体如下，其中而 rgBlocks 保存了日志块的大小

```plain
typedef struct _CLFS_CONTROL_RECORD
{
    CLFS_METADATA_RECORD_HEADER hdrControlRecord;
    ULONGLONG ullMagicValue;
    ULONG Version;
    CLFS_EXTEND_STATE eExtendState;
    USHORT iExtendBlock;
    USHORT iFlushBlock;
    ULONG cNewBlockSectors;
    ULONG cExtendStartSectors;
    ULONG cExtendSectors;
    CLFS_TRUNCATE_CONTEXT cxTruncate;
    ULONG cBlocks;
    ULONG cReserved;
    CLFS_METADATA_BLOCK rgBlocks[6];
} CLFS_CONTROL_RECORD;
```

其中 CLFS\_METADATA\_BLOCK 的结构体为

```plain
typedef struct _CLFS_METADATA_BLOCK
{
    ULONGLONG pbImage;  
    ULONG cbImage;        
    ULONG cbOffset;       
    CLFS_METADATA_BLOCK_TYPE eBlockType;
    ULONG Padding;
} CLFS_METADATA_BLOCK;
```

上述的 CLFS 数据结构，和一些对日志文件进行编码，添加容器等操作，在后续的 CVE 漏洞分析中都有所提及。是下边漏洞分析的关键。更好的梳理 clfs 内不同的数据结构了解其作用和功能，能更好的有助于复现和挖掘新的 clfs 驱动系列的问题。

## **漏洞分析**

### CVE-2022-24521

漏洞成因：CClfsBaseFilePersisted::LoadContainerQ 在此函数中有一处逻辑，先来正常梳理一下，如果 CLFS\_CONTAINER\_CONTEXT->cidQueue 的值为 -1，那么此时 CLFS\_CONTAINER\_CONTEXT->pContainer 的值就被设置成了 0，然后就会调用 CClfsBaseFilePersisted::RemoveContainer 函数。

[![](assets/1704182823-94dcad77d7d2f8d4fffc510fa2298883.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101225803-2a0820b8-a8b6-1.png)  
CClfsBaseFilePersisted::RemoveContainer 函数中，还调用了 CClfsBaseFilePersisted::FlushImage 函数，然后进行取值和校验后，将调用 pContainer 对象中存储的函数。

[![](assets/1704182823-1760d60d93930ce7651b11c0e1461690.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101225814-308f3638-a8b6-1.png)  
而通过 CClfsContainer::CClfsContainer 函数可知，\*pContainer 是存放在虚函数表中。

[![](assets/1704182823-e9baf09e27187c45b43204ff7c398d13.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101225828-3906f396-a8b6-1.png)

此时我们在看一下 CClfsBaseFilePersisted::FlushImage 函数

[![](assets/1704182823-7eaaf84ec38ae59824e526194979f9b2.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101225839-3f57db5c-a8b6-1.png)

其内部将执行 CClfsBaseFilePersisted::WriteMetadataBlock 函数，该函数会遍历每一个容器上下文，将 pContainer 先保存后置为 0。

[![](assets/1704182823-ba249c4eddfb59d87df3aaa75412e6b6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101225850-4621fe5e-a8b6-1.png)

然后其内部又调用 ClfsEncodeBlock 和 ClfsDecodeBlock 对数据进行编码和解码。直观的感受如图

[![](assets/1704182823-a252aac0d222323211e4416f1aac689c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101225900-4b9d4ef6-a8b6-1.png)

而上述过程中虽然代码有刻意的将 pContainer 进行保护，置其为 0，但在 ClfsEncodeBlock 编码的时候，会将每 0x200 字节的后两字节写入到 SignaturesOffset 中。那么如果 \_CLFS\_LOG\_BLOCK\_HEADER 中的 SignaturesOffset 字段被修改让它与\_CLFS\_CONTAINER\_CONTEXT 中 pContainer 指针相交，这样在函数编码的时候，其写入的内容就会覆盖掉 pContainer 执行。这样 CClfsBaseFilePersisted::FlushImage 函数在调用 pContainer 时候，其实执行的就是我们构造好的地址。从而导致 RIP，来实现函数执行任意函数的目的。

### CVE-2022-30220

此漏洞位于 CClfsBaseFilePersisted::ReadImage 函数中

[![](assets/1704182823-d0cbb987cef0b5c650063bb893cf07ab.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101225917-560f0348-a8b6-1.png)  
它没有上个漏洞较多的繁琐流程，此漏洞经补丁 diff 呈现的效果比较直观。主要是关于 \_CLFS\_CONTROL\_RECORD-> cBlocks(cBlocks 字段指示数组中的块数量。) 值大小的校验。如上图所示，if ( cBlocks > 6u ) 通过此判断后，随后又在在函数中使用 ExAllocatePoolWithTag 分配了两个池，分别为 24*和 2*大小的池空间。然后又对两个池进行 memmove 操作。之后将\_CLFS\_CONTROL\_RECORD -> rgBlocks 列表复制到 pool1 处。其实在看到此处分配的时候就有个猜想，会不会是分配的是大小为 0 的地址。也就是未初始化的空间。在读取日志块的函数后，随后继续往下看 CClfsBaseFileSnapshot::InitializeSnapshot 函数，同样使用了 ExAllocatePoolWithTag 函数去分配。而在这时候 cblocks 的值是比实际的大。这样就导致最后 ExAllocatePoolWithTag 分配的空间是 0，然后下边 memmove 函数复制的数据也是 0。

[![](assets/1704182823-dba394d7d7fb795fa86695c709c996f6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101225935-60c3e70e-a8b6-1.png)

最后执行到。ClfsEncodeBlockPrivate 函数时候，blockHeader 指向的就是一个未初始化的空间了，然后在下方数据的运算写入数据的时候，发送了错误导致崩溃。

### CVE-2022-37969

此漏洞主要的问题是缺乏对 clf.sys 中基本日志文件 (BLF) 的基本记录头中的字段 cbSymbolZone 的严格边界检查。在上述前言的介绍中，我们知道 BLF 文件在 0x200 字节扇区中写入/读取，后两个字节用于存储扇区签名。对于合法的 BLF 文件，具有原始字节的数组位于块的最后。ClfsEncodeBlock 和 ClfsDecodeBlock 函数负责编码解码还有将签名和原始字节写入它们各自的位置。（24521 中我们已经分析过的函数处理逻辑)。还写\_CLFS\_BASE\_RECORD\_HEADER 结构体中存在一个 cbSymbolZone。复习了一下之后，定位到关键处函数 CClfsBaseFilePersisted::AllocSymbol

[![](assets/1704182823-40f0c1bb3e848cf76565370e0b71b09d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101230016-78e6e46c-a8b6-1.png)  
从函数可知 cbSymbolZone 的值被修改，然后执行了 if 判断，cbSymbolZone 主要是用来计算新增的 Container 应该略过多少空间，也是此处漏洞的关键。此函数直接使用了 blf 文件中存储的 cbSymbolZone 的值，这样如果我们操作了 cbSymbolZone 的值，就能去改写 container，还能修改 container 的 pcontainer 指针。  
而函数中的 if 判断，就是下一步我们需要考虑的，不过与 cbSymbolZone 其比较的数值是 SignaturesOffset 字段，该字段我们可以在我们创建的 blf 文件，让被内存中的大量数据覆盖。这样，即使将 cbSymbolZone 字段设置为异常值，仍然可以绕过 cbSymbolZone 字段的验证。  
这样当函数调用 memset() 时候，就会导致在 v10 处发生越界写入，该偏移量属于日志中的 CLFS\_CONTAINER\_CONTEXT 结构。这样 CLFS\_CONTAINER\_CONTEXT 结构中偏移量的 CClfsContainer 指针被损坏就可以指向用户态可申请内存地址，然后再申请内存覆盖其地址。

### CVE-2023-23376

此漏洞主要涉及到了 CLFS\_CONTROL\_RECORD 结构体。此结构体用于保存 CLFS\_METADATA\_BLOCK，其中包含有关日志中存在的所有块的信息，并且还可以更改块大小的附加字段。其中块有关的是 eExtendState，iFlushBlock-正在写入的块的索引;cNewBlockSectors-新块的大小 (以扇区为单位);cExtendStartSectors -原始块小;cExtendSectors-添加的扇区数。cclfsbasefilepersist::OpenImage 函数检查中断的块扩展操作是否应该继续。

[![](assets/1704182823-c21a5f012e79a9189bc138f28828b3cb.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101230111-99fdba40-a8b6-1.png)

这个函数检查 iExtendBlock 和 iFlushBlock 索引，它们应该小于 6。否则，将在 CClfsBaseFilePersisted::ExtendMetadataBlock 函数中映射缓冲区 m\_rgBlocks 之外读取块指针。  
而传入 CClfsBaseFilePersisted::ExtendMetadataBlock 函数中的 iExtendBlock 和 iFlushBlock 并不被检查。  
所以此处就有一个问题。OpenImage 函数检查，而其他函数调用 ExtendMetadataBlock 函数存在问题，  
但是如果函数的调用流程 首先经过 OpenImage 函数的检查，那么关于索引的检查就会通过不了。但如果经过了此函数的检查，再去修改 CLFS\_CONTROL\_RECORD 或 索引，那么就可以利用这一点将地址作为指向指针传递。  
此处也是 Windows 中比较经典的逻辑漏洞。  
因此漏洞最后改变 CLFS\_CONTROL\_RECORD 结构中的 eExtendState, iExtendBlock, iFlushBlock 和其他字段的值  
然后执行 OpenImage 中的 ExtendMetadataBlock 函数时候，修改 CLFS\_CONTROL\_RECORD，让 CLFS\_CONTROL\_RECORD->DumpCount 递增，这样 ClfsEncodeBlock 函数（在 24521 中有过介绍）对块进行编码，然后将其更改的数据写入。这样就会覆盖 CLFS\_LOG\_BLOCK\_HEADER->RecordOffsets\[0\] 。最后被修改过的 CLFS\_CONTROL\_RECORD 就会执行，就可以在内存中执行任意的值。

### CVE-2023-28252

首先是 CClfsBaseFilePersisted::OpenImage 函数中调用 CClfsBaseFilePersisted::ReadImage 函数

[![](assets/1704182823-222641701bcd638b9dd00f9a26b2f88f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101230200-b707db7a-a8b6-1.png)  
然后在 CClfsBaseFilePersisted::ReadImage 函数中调用 CClfsBaseFile::GetControlRecord

[![](assets/1704182823-f0a74f8516e95363bd1e5b0aa9932209.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101230208-bc08ebaa-a8b6-1.png)

而在 CClfsBaseFilePersisted::OpenImage 函数中，函数调用的一个逻辑顺序 ReadImage->GetBaseLogRecord->条件判断（分别判断 eExtendState iExtendBlock iFlushBlock 的值）->ExtendMetadataBlock。

[![](assets/1704182823-b11560b142a864c874fc030408e99f37.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101230224-c5919a78-a8b6-1.png)

CClfsBaseFilePersisted::ExtendMetadataBlock 函数中会依靠 iExtendBlock 的值来加载元数据，然后获取指针，通过 eExtendState 的值来进行分支判断。

[![](assets/1704182823-b8c9f2d017224ce8681b38efcc31a7f2.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101230236-cc6ad47c-a8b6-1.png)

而后又调用了 CClfsBaseFilePersisted::FlushControlRecord 函数

[![](assets/1704182823-4214e47f8ef75fd650c09da9ad7c71be.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101230246-d2ca672e-a8b6-1.png)

而 CClfsBaseFilePersisted::FlushControlRecord 函数，对 GetControlRecord 获取日志控制块进行了分支判断，继而可以走到 CClfsBaseFilePersisted::WriteMetadataBlock 函数内。而后 WriteMetadataBlock 函数内获取和更新元数据块的地址，而更新的条件参数是 iFlushBlock 的值。随便内部又调用了 ClfsEncodeBlock 和 WriteSector 对数据进行编码和写入处理。

[![](assets/1704182823-0d469464101b2052d03be4313327561b.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240101230256-d85cfa58-a8b6-1.png)

在 WriteMetadataBlock 函数中而是直接调用了 WriteSector 函数写入数据。假设执行 ClfsEncodeBlock 时则首先将校验和置为 0，然后 ClfsEncodeBlockPrivate 检测到异常返回错误码，因此 ClfsEncodeBlock 不会更新校验和。然后元数据块的校验和变成为 0，这样就可以绕过 OpenImage 函数中的检测。  
在 CreateLogFile 函数会调用 CClfsBaseFilePersisted::CheckSecureAccess 函数。CheckSecureAccess 函数会遍历容器获取 pContainer 值，并调用虚函数。因此可以创建一个日志文件，其包含了精心构造的容器，然后利用该漏洞修改容器的偏移，使其再次调用 CreateLogFile 时，遍历的容器为构造的容器，然后 pContainer 值指向用户空间中构造虚函数布局，这样调用虚函数时便可以执行我们想要执行的函数。  
综上：WriteMetadataBlock 的 ClfsEncodeBlock 检测到异常触发错误码的时候，还是更新了 MetaBlockControl 块数据，这样就导致下次加载 MetaBlockControl 块时可以越界访问 MetaBlockControlShadow 块。  
在分析了多个 clfs 漏洞，我们可以发现这些漏洞基本上都集中于日志文件的写入操作时，对各个块头的处理也包括对偏移，指针的操作。

## **总结归纳**

整理 clfs 系列的漏洞，感觉最好还是通过 diff 出的结果再结合细节去分析，因为 clfs 系列的漏洞都是有利用其很多相关的函数。  
clfs 漏洞系列整体都是和 blf 文件内的结构还有对 blf 文件写入日志系统过程中所调用的函数有关，还有对用于存储实际数据的容器有关。还有对于 CLFS\_LOG\_BLOCK\_HEADER 等，驱动大多都会从此处开始来进行解析通过偏移量来完成流程，于是乎修改偏移等位置信息来造成内存破坏就是漏洞发现的一个关键点了。还有上述牵扯比较多的 CLFS\_CONTAINER\_CONTEXT 等等。

参考：  
[https://bbs.kanxue.com/thread-275566.htm#%E6%BC%8F%E6%B4%9E%E5%A4%8D%E7%8E%B0](https://bbs.kanxue.com/thread-275566.htm#%E6%BC%8F%E6%B4%9E%E5%A4%8D%E7%8E%B0)  
[https://github.com/ionescu007/clfs-docs/](https://github.com/ionescu007/clfs-docs/)  
[https://mp.weixin.qq.com/s/mkLjAyalo55PmdLbzMnX9g](https://mp.weixin.qq.com/s/mkLjAyalo55PmdLbzMnX9g)
