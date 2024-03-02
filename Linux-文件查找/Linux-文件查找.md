
# [Linux - 文件查找](https://www.raingray.com/archives/1207.html)

## 目录

-   [目录](#%E7%9B%AE%E5%BD%95)
-   [locate](#locate)
-   [find](#find)
    -   [案例](#%E6%A1%88%E4%BE%8B)

## locate

locate 特性是非实时性，根据自己的数据库查找文件，CentOS 通过任务计划每天更新一次数据库，安装完 locate 会生成 `/var/lib/mlocate` 目录来存放数据库，`mlocate.db` 是数据库。另外一个特性是模糊匹配，默认情况它会匹配目录名，查询结果包含了关键字就都显示出来。

语法：`locate [option] file_name`

```plaintext
updatedb  #遍历整个根文件系统来更新locate数据库，运行时占用资源多。
yum -y install mlocate #安装locate工具
```

选项：

```bash
-b #只匹配基名
#例如这样一段路径/root/anaconda-ks.cfg,anaconda-ks.cfg是基名,/var/www/是目录名
-c #显示匹配到的条数(count)
```

## find

find 实时遍历目录下所有文件和目录，不像 locate 有索引库，因此速度会稍微慢一些，其次与 locate 另一个区别是精确匹配，有就是有没有就是没有。

语法：`find [path] [options] [action]`

选项：

-   匹配模式（可以用[通配符](https://www.raingray.com/archives/31.html#%E5%91%BD%E4%BB%A4%E8%A1%8C%E9%80%9A%E9%85%8D%EF%BC%88Wildcard%EF%BC%89)）
    
    -   \-name，精确匹配
    -   \-iname，不区分大小写
-   根据属主属组查找
    
    -   \-user user\_name
    -   \-group gourp\_name
    -   \-uid id\_number
    -   \-gid id\_number
    -   \-nouser，没有属主的文件，呈现的样子是直接显示 uid
    -   \-nogourp，没有属主的文件，呈现的样子是直接显示 uid
-   根据文件类型查找
    
    -   \-type
        -   f，普通文件
        -   d，目录文件
        -   l，link 文件
        -   b，block 设备文件
        -   c，字符设备文件
        -   p，管道文件
        -   s，套接字文件（soket）
-   按照大小查找
    
    -   `-size [+|-] k M G`
-   根据时间戳查找
    
    -   `-atime [+|-]`，按天查
        -   5，从当前时间计算往前推五天
        -   +5，距离当前时间五天外
        -   \-5，从当前时间计算五天内
    -   \-mtime
    -   \-ctime
    -   \-amin，按分钟查
    -   \-mmin
    -   \-cmin
-   根据权限查找  
    \-perm
    
    -   644，只匹配 644 权限
    -   /644，只要 9 位权限中包含 1 位就可以，644 里 6 是 rw，只要有 r 或者 w 其中一位就能匹配到。
    -   \-644，只要你包含 644 权限就匹配，777 也包含 644 权限，655 也包含这个权限。
-   组合条件，几个选项之间做逻辑运算。
    
-   \-a，默认值
    
-   \-o
    
-   \-not，用 ! 也是一样的意思
    

动作：

-   `-print`，默认输出到标准输出设备
-   `-ls`，等同于 ls -l
-   `-ok command {} \;`，询问要不要执行操作，要以 \\; 结尾，{} 代表你前面找到文件的文件名，也就是文件占位符，要是不需要命令对文件操作就删掉。
-   `-exec command {} \;`，不询问直接执行操作。

### 案例

**搜索**

搜索 /tmp 目录，文件名不包含 fstab 的文件。

```plaintext
find /tmp ! -name "*fstab*" 
```

搜索 /tmp 目录，文件属主不是用户 root 而且文件名不包含 fstab。

```plaintext
find /tmp ! -user root ! -name "*fstab*" 
```

查找 /tmp 下没有属组属主的普通文件，并执行 -ls 动作。

```plaintext
find /tmp -nouser -nogroup -type f -ls
```

找当前目录下所有没有属主属组的文件，执行 -ok 动作，命令是 chown root.root 将它归属赋与 root 用户。

```plaintext
find ./ -notuser -notgroup -ok chown root.root {} \;
```

**权限查找**

644 这 9 位权限中 (user,group.other)，只要有一位匹配就显示。换算为二进制是：110,100,100。只要筛选的文件满足其中一位权限就显示。比如查找 700,600 也会显示因为匹配到第一和第二位 (110)。

```plaintext
find /etc -perm /644 -type f -name '[[:digit:]]' -ls
```

查找包含 744 权限的文件，755 也是包含 744 的。也就是说你可以大于这个权限范围，但不能小于，比如 700 就不显示。

```plaintext
find /etc -perm -700 -ls
```

找其他用户有读写权限的文件

```plaintext
find ./ -perm -006
```

最近更新：2023 年 02 月 17 日 11:30:06

发布时间：2019 年 03 月 23 日 02:28:00
