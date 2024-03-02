
# [Linux - 文本处理](https://www.raingray.com/archives/1581.html)

## 目录

-   [目录](#%E7%9B%AE%E5%BD%95)
-   [nl](#nl)
-   [split](#split)
-   [tr](#tr)
-   [xargs](#xargs)
-   [parallel（待补充）](#parallel%EF%BC%88%E5%BE%85%E8%A1%A5%E5%85%85%EF%BC%89)
-   [diff](#diff)
-   [wc](#wc)
-   [文本编辑](#%E6%96%87%E6%9C%AC%E7%BC%96%E8%BE%91)
    -   [nano（待补充）](#nano%EF%BC%88%E5%BE%85%E8%A1%A5%E5%85%85%EF%BC%89)
    -   [vim](#vim)
-   [grep](#grep)
-   [cut](#cut)
-   [awk](#awk)
    -   [案例](#%E6%A1%88%E4%BE%8B)
        -   [打印指定列数据](#%E6%89%93%E5%8D%B0%E6%8C%87%E5%AE%9A%E5%88%97%E6%95%B0%E6%8D%AE)
        -   [更多条件判断](#%E6%9B%B4%E5%A4%9A%E6%9D%A1%E4%BB%B6%E5%88%A4%E6%96%AD)
        -   [实现 vulookup](#%E5%AE%9E%E7%8E%B0+vulookup)
-   [sort](#sort)
-   [uniq](#uniq)
-   [sed](#sed)
-   [printf（待补充）](#printf%EF%BC%88%E5%BE%85%E8%A1%A5%E5%85%85%EF%BC%89)
-   [参考链接](#%E5%8F%82%E8%80%83%E9%93%BE%E6%8E%A5)

## nl

可以显示行号并打印所有内容到屏幕。

Usage：`nl file_name`

## split

将一个文件分割成多个文件。

Usage：`split [-bl] [file] [prefix] # -l按行数分割，-b 单位值：K,M,G,T,P,E,Z,Y`  
Example：`find /etc -type f | split -l 10 - lsroot`

管道 -l 后面的减号是被当做标准输入，因为 split 语法是要求对一个输入做处理，这个输入可以是 file 其他输入，然后在输出出去，这个标准输入内容是从管道穿过来的。

## tr

替换、删除字符。

Example：`echo [192.168.1.1] | tr -d []`

## xargs

可以将管道或标准输入（stdin）数据转换成命令行参数，就是将接收过来的数据当做参数交给后面的命令去执行它。

Usage：`xargs [Pn] command parameter`  
Options：

```bash
-I # 默认是参数放在命令最后面，加入 -i {} 可以指定参数替换位置，也就是说最终会把占位符 {} 替换为参数。
-p # 会启用多线程同时执行任务，最大值一般与CPU线程数一致，参数值格式是数字。
-n # 命令一次要用几个参数。
-t # 执行命令前把整体命令打印出来，方便调试。
```

Example：

```plaintext
[root@centos7 ~]# ls #1.sh内容是输出1,2.sh输出2
1.sh  2.sh

[root@centos7 ~]# ls *.sh | xargs sh #这里讲前面显示的结果当做参数传给 sh 执行
1
#？为什么结果是1不应该两个script都执行吗？

[root@centos7 ~]# ls *.sh | xargs -t sh #加上-t 可以将要执行的命令打印出来
sh 1.sh 2.sh 
1
#这里执行sh 1.sh 2.sh，这样只能执行到1.sh，默认是把显示到一行以空格隔开。

[root@centos7 ~]# ls *.sh | xargs -t -P 2 -n 1 sh # 加上-n告诉他每次处理几个参数，-P指定线程数。
sh 1.sh 
1
sh 2.sh 
2

[root@centos7 ~]# echo 123 | xargs -I {} mv {} 1 # 将 123 文件重命名为 1
```

## parallel（待补充）

和 xargs 类似，貌似功能丰富些。

## diff

比较两个文件不同处并显示出来。

Usage：`diff [r] #-r比较目录`

## wc

查看文本字符和行数。

Usage： `wc [l] #-l打印文件行数`

```bash
root@gbb:~# wc test.txt
# 输出结果
7     8     70     test.txt
行数 单词数 字节数  文件名
```

## 文本编辑

### nano（待补充）

### vim

vim 快捷操作应该多复习《鸟哥私房菜》vim 篇章，还可使用 `vimtutor` 命令快速复习 vim 用法。

复制 / 粘贴 / 删除

```bash
yy #复制当前行
y3j #复制 3 行，从当前行开始向下复制 3 行。

p #向下粘贴
P #向上粘贴

dd #删除当前行
dnj #d 是删除当前行，n 是当前行下面儿你要删除几行，j 是向下移动。d2j 就是删除当前行和下两行。
x #删除当前光标所在的字符
```

跳转

```powershell
G #跳到文件最后一行
gg #跳到文件第一行
line_num+G #行数 +G 跳到指定行，比如输入 90 再输入 G 就能准确跳到第 90 行。
N+Enter #向下移动 N 行
$ #回到行尾
0 #回到行首
```

替换

```sql
:s/old/new/g #将整个文本中的 old 替换成 new，其中的 g=global 表示全局的意思。
:5,10 s/old/new/g #替换指定行数中的内容。
```

杂项

```plaintext
u #撤回刚刚的操作
Ctrl+r #重复刚撤销的操作
:set nu #显示行号，取消显示是 :set nonu
a #在当前光标字符后面进入插入模式。
o #向下换一个新行
O #向上换一个新行
Shift+zz #保存操作并退出 vim，等同于 :wq 或 :x
:!command #在 vim 外执行一个命令，再回到当前 vim。
:r !command #在 vim 外执行一个命令，将执行结果读取当前光标处。
:r text.text #将 text.text 内容读取，写到当前光标处。
```

## grep

全称：global search REgular expression and print out the line

根据你指定匹配模式对文本每行进行过滤，把内容匹配的那行显示出来。相当于用正则匹配出你想要的字符串。

Usage：`grep [options] pattern`  
Options：

```plaintext
-i #忽略大小写(ignorecase)
-o #只显示匹配到的字符串
-v #忽略匹配到字符串(等同于长选项--invert-match)，来显示未匹配的内容。
-E #用扩展正则表达式，默认支持基本正则(选项-e)。
-c #计算总共符合条件字符的个数
-n #显示行号
-i #忽略大小写，不加默认是区分大小写的。 
-A num #显示匹配到的内容和后(after)num行
-B num #显示匹配到的内容和前(before)num行
-C num #显示匹配到的内容和内容上下num行(context)
```

基本正则：  
字符匹配

```plaintext
. #任意单个字符
[] #匹配指定范围内的任意单个字符
[^]
[:space:] #表示空格          //在使用时要加上[]列如：[[:space:]]
[:punct:] #表示所有标点符号
[:lower:] #表示所有小写字母a-z
[:upper:] #表示所有大写字母A-Z
[:alpha:] #表示所有字母(包括大小写)
[:digit:] #表示所有数字0-9
[:alnum:] #表示所有数字和大小写字母a-z,A-Z,0-9
```

匹配次数

```plaintext
* #匹配前面字符任意次
.* #匹配任意长度任意字符
\? #匹配前面字符0次或1次
\+ #匹配前面字符1次或多次
\{m\} #匹配前面字符m次
\{m,n\} #匹配前面字符最少m次最多n次
```

位置锚定

```powershell
^ #开头，写在pattern左侧
$ #结尾，写在pattern右侧
\<
\>
```

分组及引用  
...  
Example：

```plaintext
[^xxx] #托字符代表排除xxx  
'^x' #以x开头的字母，不是只能也可以用正则中的[[:lower:]]。     '^[^[:alpha:]]'  #筛选出开头不是英文字母的字符。
grep -n '^$' #筛选空行
grep -n 'a*' #匹配a这个字符0次或者多次，正则中*与shell中*不同点在于，shell任意字符0次到无数次，正则是匹配*前面一个字符无数次，这点要区分开来。
grep -n 'a.*' #.代表任意单个字符，*代表前面一个字符0次或多次，两个组合起来就是任意字符出现0次或多次，这条表达式意思是a后面可以出现任意字符0次或多次，换句话说就是以a开头的xxx。
```

## cut

截取一行数据中的一段字符

```bash
cut -d ' ' -f 1 #-d指定分隔符 -f显示分隔后的第几个数据段  
cut -d ' ' -f 1 #-d指定分隔符(你指定1个空格它真的就以1个空格来分) -f显示分隔后几段数据中哪一段 -c显示第几个字符，-c 8- 这个用法可以显示从第8个字符到后面全部字符。  
```

## awk

awk 用于处理一段一段数据，本质上还是把一行数据分成几个块来处理。除了 awk 还有 gawk，它是 GNU awk，除标准 awk 外有一些扩展功能。

Usage：`awk 'pattern { action } pattern { action }' fiel1 file2 file3`

它的语法就如 Usage 写的一样，都是由 pattern 和 action 组成，pattern 代表规则相当于它匹配了才能执行对应的 action，当然没有 pattern 就直接执行 action。

### 案例

#### 打印指定列数据

`$0` 代表整行数据，`$NF` 代表最后一个字段，`$1` 取第一个字段，`$2` 第二个字段，以此类推，所有字段通过 print 进行输出，其中字段用逗号隔开在输出时会用空格替代。

```bash
netstat -pantu | awk '{print $1, $2}'
```

不仅仅可以用逗号简单分隔多个字段，还可指定分隔符。下面我们将分隔符替换为分号。

```bash
netstat -pantu | awk 'OFS=";" {print $1, $2}'
```

awk 默认是以空格来分割数据，如果要处理的数据不是空格隔开可以用 `-F` 指定。

```bash
cat /etc/passwd | awk -F : '{print "用户名:"$1, "UID:" $3}'
```

#### 更多条件判断

也可做条件筛选，越来越像编程了，怪不得有些作者还专门为 awk 出了本书，下面将第 3 个字段 uid 为 root 的账户筛选出来，其中 BEGIN / END 可以在处理数据前或处理完成时执行一些命令：

除了 == 还有 >=，<=，>，< ，!= 这几种常见的比较运算符。

```bash
awk -F : 'BEGIN{print "处理开始"} END{print "处理结束"} $3==0 {print $0}' /etc/passwd
```

比较运算还可以举出一个例子，比如列举出 Linux 以 root 用户运行的进程。核心思路就是配合 awk 查看第一列是否未 root。

```plaintext
┌──(kali㉿kali)-[~]
└─$ ps aux | awk '$1 == "root" { print $0 }'
root           1  0.0  0.1 167776 12136 ?        Ss   19:29   0:01 /sbin/init splash
root           2  0.0  0.0      0     0 ?        S    19:29   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        I<   19:29   0:00 [rcu_gp]
root           4  0.0  0.0      0     0 ?        I<   19:29   0:00 [rcu_par_gp]
root           5  0.0  0.0      0     0 ?        I<   19:29   0:00 [slub_flushwq]
......
```

没有用 `ps aux | grep root` 原因是，这样会把包含 root 关键字的内容筛出来有误报的可能性。

```plaintext
┌──(kali㉿kali)-[~]
└─$ ps aux | grep root | head -n 122 | tail -n 1 | grep root  
root         849  0.8  2.1 478576 172344 tty7    Ssl+ 19:29   0:33 /usr/lib/xorg/Xorg :0 -seat seat0 -auth /var/run/lightdm/root/:0 -nolisten tcp vt7 -novtswitch
```

这里可以看到进程中参数包含 root 关键字 `-auth /var/run/lightdm/root/:0`。

AWK 也能够在处理文本前去执行系统命令：

```plaintext
awk 'BEGIN{system("ls -l /etc/passwd")}'
awk 'BEGIN{print "ls","-l /etc/passwd" | "/bin/bash"}'
```

在筛选数据还可使用正则这一利器，使用方法是在条件的位置 `/REG/` 其中 REG 就是匹配模式。

```plaintext
awk -F : '/bash$/ {print $1,$NF}' OFS="::" /etc/passwd
```

#### 实现 vulookup

这里再来个例子，在 Linux 下实现类似 Excel vulookup() 函数效果怎么实现？

把 fiel2 第一列作为索引放到 file1 中匹配到整行进行输出：

```plaintext
[root@iZ2ze3excf14rd7l4rsgn3Z xx.com]# cat file1
133
1234 aabb
1445 12390590a
[root@iZ2ze3excf14rd7l4rsgn3Z xx.com]# cat file2
1234
[root@iZ2ze3excf14rd7l4rsgn3Z xx.com]# awk 'NR==FNR{a[$1]=$2; next} NR!=FNR{print $1,a[$1]}' file1 file2
1234 aabb
```

`awk 'NR==FNR{a[$1]=$2; next} NR!=FNR{print $1,a[$1]}' file1 file2`

理解这条语句，我们把它拆分为两段 `NR==FNR{a[$1]=$2; next}` 和 `NR!=FNR{print $1,a[$1]}`

上面语句 awk 运行原理：首先读取 file1 用第一个模式 `NR==FNR` 进行匹配，匹配成功将数据放入数组 a，紧接着遇到 next 就不会继续匹配模式 `NR!=FNR` 以节约时间。

这个 `NR==FNR` 是两个内置变量，`FNR` 是你文件中每条数据实际所在行数，`NR` 是 awk 处理到第几条数据的行数，换句话说是 awk 处理到第几条数据那它就是几行，哪怕你在文件中属于第一行，可 awk 对这行刚好是第 3 次处理，那就是第三行。

下面例子中你会看到，test 文件中 12312348958 FNR 显示为 2 说明它在文件中所处第二行，test1 文件中 1231jsdjf NR=3 说明当前这条数据 awk 成功处理到第三条。

```plaintext
[root@iZ2ze3excf14rd7l4rsgn3Z wanmei.com]# cat test
123213123
12312348958
[root@iZ2ze3excf14rd7l4rsgn3Z wanmei.com]# cat test1
1231jsdjf
[root@iZ2ze3excf14rd7l4rsgn3Z wanmei.com]# awk '{print $1,"NR="NR, "FNR="FNR}' test test1
123213123 NR=1 FNR=1
12312348958 NR=2 FNR=2
1231jsdjf NR=3 FNR=1
```

当第一个文件处理完后，继续处理 file2 由于模式 2 不匹配，继续匹配模式 2，成功执行动作，将 file2 中的内容按行读取作为索引获取数组 a 数据。

1.  把文件一 `$1` 作为 key，`$2` 作为 value 存入数组 a。
2.  把文件二 `$1` 数据做为索引去获取文件一的值——也就是数组 a，最终打印出来。

## sort

用于排序，按照 ASSIC 排序。

Usage：`sort [dnru]`  
Options：

```bash
-n #按照数字从小到大排列
-r #翻转排序后的结果
-u #去掉重复数据，此选项跟后面提到的 uinq 去重工具功能一致，此说明在 sort GNU 文档中有提到。
-S #使用多少内存，比如 75%，或是指定 1024M，等等。
--parallel=N #使用多少核心运算。
```

不加选项默认是比较每个字符从小到大排序 (ASCII 码)，比如 123 和 145 两行数据它匹配第一个字符 1 都是一样大略过，再去匹配第二个字符 2 和 4，2 比 4 小排上面，后面的数据就不会继续排了。

Example：

```bash
[root@centos7 ~]# cat 2q
B12
150
b39
123
c00
145
A13
a03
[root@centos7 ~]# sort 2q
123
145
150
a03
A13
B12
b39
c00
```

它不像下面要介绍的 uniq 只能去重上下相邻一致的数据，比较鸡肋。

```bash
root@gbb:~# cat 1
1
1
2
1
333
12
1
root@gbb:~# cat 1 | sort -un
1
2
12
333
```

sort 经常用于文本排序然后去重，在 Linux 下处理重复文本我想到以下几个选项

1.  将大文件切割为多个小文件，并行去重合并。
2.  使用 sort -S 60%，提高内存，--parallel=2 使用多个 CPU 多核心。  
    查看 CPU 核数：grep processor /proc/cpuinfo，从 0 开始数，0 代表 1  
    查看内存容量：free -h

CPU 1 核心，2Gb 内存云主机运行，下面是测试过程。

9Gb 文本数据单纯去重速度比较，sort | uniq -i

```bash
[root@iZ2ze3excf14rd7l4rsgn3Z wanmei.com]# time sort -u tmp_all_domain.txt | uniq -i > all_domain.txt
real    111m39.162s
user    108m29.374s
sys 0m36.479s
```

启用 -S 选项去重速度上快了一刻钟吧。因为只有 1 核心，默认是跑满的。

```bash
[root@iZ2ze3excf14rd7l4rsgn3Z wanmei.com]# time sort -S 75% tmp_all_domain.txt | uniq -i | tee all_domain.txt

real    95m8.981s
user    92m0.524s
sys     0m43.655s
```

这里也说明下运行情况，当 sort 使用 -S 80%，到了 3m 会自动退出，网上都说是硬盘满了，我 df -h 看了下硬盘，容量还有，最终也不知道怎么回事，做测试一定要多多运行几遍，不然出错输出数据不完整就很烦。

```bash
[root@iZ2ze3excf14rd7l4rsgn3Z wanmei.com]# time sort -S 80% tmp_all_domain.txt | uniq -i > all_domain.txt
uniq: write error

real    96m53.798s
user    92m31.857s
sys 0m37.084s
```

## uniq

对筛选出的数据做去重操作

Usage：`uniq [ci] [file]`  
Options：

```bash
-c #显示这行数据重复了几次
-i #忽略大小写
```

它只会去除两行或多行相同数据挨在一起的情况，下面举例。

```bash
[root@centos7 ~] cat 2 #这种它会去重
192.168.1.1
192.168.1.1
192.168.1.1
192.168.1.2
192.168.1.2
[root@centos7 ~] uniq 2
192.168.1.1
192.168.1.2
```

uniq 无法处理的情况出现了。

```bash
[root@centos7 ~] cat 3 #这样的数据 uniq 会疯...
192.168.1.1
192.168.1.2
192.168.1.1
192.168.1.1
192.168.1.2
[root@centos7 ~] uniq 3
192.168.1.1
192.168.1.2
192.168.1.1
192.168.1.2
```

介于上面这种情况，一遍常见的做法如下。

```bash
[root@centos7 ~] cat 3
192.168.1.1
192.168.1.2
192.168.1.1
192.168.1.1
192.168.1.2
[root@centos7 ~] sort 3 | uniq -c #先排序后去重
      3 192.168.1.1
      2 192.168.1.2
```

## sed

sed 不光有检索还可以增删替换。

Usage：`sed [options] 动作`

Options：

```bash
-i  #将操作结果保存到源文件  
```

Actions：

```bash
s #替换
d #删除
```

Example：

```bash
date | sed 's/:/+/g' # s 替换，g 全局查找，/ 是分隔符可以替换其他字符，为了避免歧义还是不要使用字母来避免正则中元字符的冲突。  
sed 's!^!http://!' /etc/passwd #给开头加上 http://  
sed 's!$!/index.php!' /etc/passwd #给结尾加上 /index.php  
sed -n 10,15p /etc/passwd #打印 passwd 10-15 行内容。  
```

## printf（待补充）

配合 seq 可以快速生成指定格式的数字、字符。

格式字符串：

-   `%x`，十六进制，不带 0x

%x 是十六进制，02 中 0 是要填充的字符 2 是填充多少位，意思是只要个数少于两位就用 0 填充。

```plaintext
for i in `seq 1 100`; do printf "%02x\n" $i; done
```

## 参考链接

-   [Linux 基础：xargs 命令](https://www.cnblogs.com/chyingp/p/linux-command-xargs.html)
-   [简明 Vim 练级攻略](https://coolshell.cn/articles/5426.html)
-   [AWK 简明教程](https://coolshell.cn/articles/9070.html)
-   [How to Use the awk Command on Linux](https://www.howtogeek.com/562941/how-to-use-the-awk-command-on-linux/)
-   [The GNU Awk User’s Guide](https://www.gnu.org/software/gawk/manual/html_node/index.html#SEC_Contents)
-   [https://www.runoob.com/linux/linux-comm-awk.html，awk](https://www.runoob.com/linux/linux-comm-awk.html%EF%BC%8Cawk) 运行原理
-   [Linux Shell 中使用 awk 完成两个文件的关联 Join](http://lxw1234.com/archives/2016/03/621.htm)，对于数据关联原理解释的很好
-   [linux 使用 awk 实现 excel 中的 vlookup 行数匹配效](https://blog.csdn.net/qq_35515661/article/details/90488993)，完美展示数据关联功能

最近更新：2023 年 02 月 17 日 11:03:27

发布时间：2019 年 03 月 25 日 23:24:00
