

# 三分钟数据持久化：Spring Boot, JPA 与 SQLite 的完美融合

![](assets/1709195636-6ababfd1e2cbe54908b145b9121b5f93.png)

**程序猿阿朗**

: ) 早睡早起，坚持不懈。

88 篇原创内容

公众号

> 三分钟，迎接一个更加高效和简便的开发体验。

在快节奏的软件开发领域，每一个简化工作流程的机会都不容错过。想要一个无需繁琐配置、能够迅速启动的数据持久化方案吗？这篇文章将是你的首选攻略。在这里，我们将向你展示如何将 **Spring Boot** 的便捷性、**JPA** 的强大查询能力和 **SQLite** 的轻量级特性结合在一起，实现快速而又优雅的数据管理。

## 为什么选择 SQLite

**SQLite** 是一个用 C 语言编写的开源、轻量级、快速、独立且高可靠性的 SQL 数据库引擎，它提供了功能齐全的数据库解决方案。对于大多数的应用，**SQLite** 都可以满足。使用 SQLite 可以**零配置**启动，对于小型应用或者快速原型设计是一个非常大的优势。

使用 SQLite 具有下面几个优点：

1.  1\. 轻量级：SQLite 很小巧，不需要独立服务器，便于集成到应用中。
    
2.  2\. 零配置：启用 SQLite 无需复杂配置，只需指定一个文件路径存放 DB 文件，简化了数据库的设置流程。
    
3.  3\. 便于移植：数据库是单一文件，方便备份和在不同环境间迁移。
    
4.  4\. 跨平台：SQLite 支持各种操作系统，容易实现应用的跨平台运行。
    
5.  5\. 性能良好：对于小型应用，SQLite 提供足够的读写性能。
    
6.  6\. 遵循 ACID：SQLite 事务符合 ACID 原则，数据操作可靠。
    
7.  7\. 社区支持：虽然简单，但拥有强大的社区和广泛的文档资源。
    

之前写过一篇 [SQLite 入门教程](http://mp.weixin.qq.com/s?__biz=MzI1MDIxNjQ1OQ==&mid=2247485543&idx=1&sn=9081e684715e29971aa84b7fbe298df0&chksm=e984e103def36815263db558c5827cf77b5eb105a5b0d2f39ed578e4cec0181a14746d7bf4bc&scene=21#wechat_redirect) (https://www.wdbyte.com/db/sqlite/)\[1\]，感情的同学可以参考。

## 为什么 选择 JPA

**Spring Data JPA** 是 Spring Data 项目的一部分，旨在简化基于 JPA（Java Persistence API）的数据访问层（Repository 层）的实现。JPA 是一种 ORM（对象关系映射）规范，它允许开发者以面向对象的方式来操作数据库，

通常应用程序实现数据访问层可能非常麻烦，必须编写太多的样板代码才能实现简单的查询，更不用说分页等其他操作，而 Spring Data JPA 可以让开发者非常容易地实现对数据库的各种操作，显著减少实际需要的工作量。

详细介绍 JPA 并不是本文目的，关于 JPA 的更多内容可以访问：

1.  1. Spring Data JPA 官网：https://spring.io/projects/spring-data-jpa\[2\]。
    
2.  2. Spring Boot 使用 Spring Data JPA\[3\]
    

## 创建 Spring Boot 项目

用于后续演示，首先创建一个简单的 Spring Boot 项目。你可以自由创建，或者使用 Spring 官网提供的快速创建工具：https://start.spring.io/\[4\]

> 注意，文章示例项目使用 Java 21 进行演示。

为了方便开发，创建一个基础的 Spring Boot 项目后，添加以下依赖。

```plain
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!-- 从 Hibernate 6 开始，支持 SQLite 方言。-->
<!-- https://mvnrepository.com/artifact/org.hibernate.orm/hibernate-community-dialects -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-community-dialects</artifactId>
    <version>6.4.3.Final</version>
</dependency>
<!-- sqlite jdbc 驱动 -->
<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.45.1.0</version>
</dependency>
<!-- Lombok 简化 get set tostring log .. -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
<!-- apache java 通用工具库 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.13.0</version>
</dependency>
<!-- 编码通用工具库 -->
<dependency>
    <groupId>commons-codec</groupId>
    <artifactId>commons-codec</artifactId>
    <version>1.16.0</version>
</dependency>
```

## 配置 SQLite & JPA

在 Spring Boot 中，对 SQLite 的配置非常简单，只需要指定一个位置存放 SQLite 数据库文件。SQLite 无服务端，因此可以直接启动。

```plain
spring.datasource.url=jdbc:sqlite:springboot-sqlite-jpa.db
spring.datasource.driver-class-name=org.sqlite.JDBC
# JPA Properties
spring.jpa.database-platform=org.hibernate.community.dialect.SQLiteDialect
# create 每次都重新创建表，update，表若存在则不重建
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

## 配置实体映射

在使用 JPA 开发时，就是使用 `jakarta.persistence` 包中的注解配置 Java 实体类和表的映射关系，比如使用 `@Table` 指定表名，使用 `@Column` 配置字段信息。

```plain
import java.time.LocalDateTime;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import lombok.Getter;
import lombok.Setter;
import lombok.ToString;

@Entity
@Getter
@Setter
@ToString
@Table(name = "website_user")
public class WebsiteUser {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "username", nullable = false, unique = true, length = 64)
    private String username;

    @Column(name = "password", nullable = false, length = 255)
    private String password;

    @Column(name = "salt", nullable = false, length = 16)
    private String salt;

    @Column(name = "status", nullable = false, length = 16, columnDefinition = "VARCHAR(16) DEFAULT 'active'")
    private String status;

    @Column(name = "created_at", nullable = false, columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP")
    private LocalDateTime createdAt;
  
    @Column(name = "updated_at", nullable = false, columnDefinition = "TIMESTAMP DEFAULT CURRENT_TIMESTAMP")
    private LocalDateTime updatedAt;
}
```

## 编写 JPA 查询方法

Spring Data JPA 提供了多种便捷的方法来实现对数据库的查询操作，使得能够以非常简洁的方式编写对数据库的访问和查询逻辑。比如 Spring Data JPA 允许通过在接口中定义遵循一定命名方法的方式来创建数据库查询。如`findByName` 将生成一个根据 `name` 查询指定实体的 SQL。

代码示例：

```plain
@Repository
public interface WebsiteUserRepository extends CrudRepository<WebsiteUser, Long> {

    /**
     * 根据 username 查询数据
     * @param name
     * @return
     */
    WebsiteUser findByUsername(String name);
}
```

代码示例中，继承的 `CrudRepository` 接口中包含了常见的 CURD 操作方法。自定义的 `findByUsername` 方法可以根据 `WebsiteUser` 中的 Username 进行查询。

## 编写 Controller

编写三个 API 用来演示 Spring Boot 结合 SQLite 以及 JPA 是否成功。

初始化方法 `init()`：

-   • 映射到 `"/sqlite/init"` 的 GET 请求。
    
-   • 创建了 10 个 `WebsiteUser` 实体，为每个用户生成随机的用户名和盐值，并用 MD5 加密其密码（"123456" + 盐）。
    
-   • 用户信息包括用户名、加盐后的密码、创建和更新的时间戳，以及用户状态。
    
-   • 用户信息被保存到数据库中，并记录日志。
    

查找用户方法 `findByUsername(String username)`：

-   • 映射到 `"/sqlite/find"` 的 GET 请求。
    
-   • 通过用户名查询用户。如果找到，返回用户的字符串表示；否则返回 `null`。
    

登录方法 `findByUsername(String username, String password)`：

-   • 映射到 `"/sqlite/login"` 的 GET 请求。
    
-   • 验证传入的用户名和密码。首先通过用户名查询用户，然后将传入的密码与盐值结合，并与数据库中存储的加盐密码进行 MD5 加密比对。
    
-   • 如果密码匹配，则认证成功，返回 "login succeeded"；否则，返回 "login failed"。
    

代码示例：

```plain
import java.time.LocalDateTime;
import com.wdbyte.springsqlite.model.WebsiteUser;
import com.wdbyte.springsqlite.repository.WebsiteUserRepository;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.lang3.RandomStringUtils;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
/**
 * @author https://www.wdbyte.com
 */
@Slf4j
@RestController
public class SqliteController {

    @Autowired
    private WebsiteUserRepository userRepository;

    @GetMapping("/sqlite/init")
    public String init() {
        for (int i = 0; i < 10; i++) {
            WebsiteUser websiteUser = new WebsiteUser();
            // 随机 4 个字母
            websiteUser.setUsername(RandomStringUtils.randomAlphabetic(4));
            // 随机 16 个字符用于密码加盐加密
            websiteUser.setSalt(RandomStringUtils.randomAlphanumeric(16));
            String password = "123456";
            // 密码存储 = md5(密码 + 盐)
            password = password + websiteUser.getSalt();
            websiteUser.setPassword(DigestUtils.md5Hex(password));
            websiteUser.setCreatedAt(LocalDateTime.now());
            websiteUser.setUpdatedAt(LocalDateTime.now());
            websiteUser.setStatus("active");
            WebsiteUser saved = userRepository.save(websiteUser);
            log.info("init user {}", saved.getUsername());
        }
        return "init success";
    }

    @GetMapping("/sqlite/find")
    public String findByUsername(String username) {
        WebsiteUser websiteUser = userRepository.findByUsername(username);
        if (websiteUser == null) {
            return null;
        }
        return websiteUser.toString();
    }

    @GetMapping("/sqlite/login")
    public String findByUsername(String username, String password) {
        WebsiteUser websiteUser = userRepository.findByUsername(username);
        if (websiteUser == null) {
            return "login failed";
        }
        password = password + websiteUser.getSalt();
        if (StringUtils.equals(DigestUtils.md5Hex(password), websiteUser.getPassword())) {
            return "login succeeded";
        } else {
            return "login failed";
        }
    }
}
```

至此，项目编写完成，完整目录结构如下：

```plain
├── pom.xml
└── src
    ├── main
        ├── java
        │   └── com
        │       └── wdbyte
        │           └── springsqlite
        │               ├── SpringBootSqliteApp.java
        │               ├── controller
        │               │   └── SqliteController.java
        │               ├── model
        │               │   └── WebsiteUser.java
        │               └── repository
        │                   └── WebsiteUserRepository.java
        └── resources
            ├── application.properties
            ├── static
            └── templates
```

## 启动测试

Spring Boot 启动时由于库表不存在，自动创建库表：

```plain
Hibernate: create table website_user (id integer, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP not null, password varchar(255) not null, salt varchar(16) not null, status VARCHAR(16) DEFAULT 'active' not null, updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP not null, username varchar(64) not null unique, primary key (id))
Hibernate: alter table website_user drop constraint UK_61p1pfkd4ht22uhlib72oj301
2024-02-27T20:00:21.279+08:00  INFO 70956 --- [main] j.LocalContainerEntityManagerFactoryBean : Initialized JPA EntityManagerFactory for persistence unit 'default'
2024-02-27T20:00:21.578+08:00  WARN 70956 --- [main] JpaBaseConfiguration$JpaWebConfiguration : spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning
2024-02-27T20:00:21.931+08:00  INFO 70956 --- [main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port 8080 (http) with context path ''
2024-02-27T20:00:21.938+08:00  INFO 70956 --- [main] c.w.springsqlite.SpringBootSqliteApp     : Started SpringBootSqliteApp in 3.944 seconds (process running for 5.061)
```

### 请求初始化接口

```plain
$ curl http://127.0.0.1:8080/sqlite/init
init success
```

可以看到输出日志成功写入了 10 条数据，且输出了 `username` 值。

```plain
2024-02-27T20:01:04.120+08:00 ...SqliteController : init user HUyz
Hibernate: insert into website_user (created_at,password,salt,status,updated_at,username) values (?,?,?,?,?,?)
Hibernate: select last_insert_rowid()
2024-02-27T20:01:04.123+08:00 ...SqliteController : init user ifQU
Hibernate: insert into website_user (created_at,password,salt,status,updated_at,username) values (?,?,?,?,?,?)
Hibernate: select last_insert_rowid()
2024-02-27T20:01:04.126+08:00 ...SqliteController : init user GBPK
Hibernate: insert into website_user (created_at,password,salt,status,updated_at,username) values (?,?,?,?,?,?)
Hibernate: select last_insert_rowid()
2024-02-27T20:01:04.129+08:00 ...SqliteController : init user rytE
Hibernate: insert into website_user (created_at,password,salt,status,updated_at,username) values (?,?,?,?,?,?)
Hibernate: select last_insert_rowid()
2024-02-27T20:01:04.132+08:00 ...SqliteController : init user iATH
Hibernate: insert into website_user (created_at,password,salt,status,updated_at,username) values (?,?,?,?,?,?)
Hibernate: select last_insert_rowid()
2024-02-27T20:01:04.134+08:00 ...SqliteController : init user ZQRW
Hibernate: insert into website_user (created_at,password,salt,status,updated_at,username) values (?,?,?,?,?,?)
Hibernate: select last_insert_rowid()
2024-02-27T20:01:04.137+08:00 ...SqliteController : init user cIPM
Hibernate: insert into website_user (created_at,password,salt,status,updated_at,username) values (?,?,?,?,?,?)
Hibernate: select last_insert_rowid()
2024-02-27T20:01:04.140+08:00 ...SqliteController : init user MemS
Hibernate: insert into website_user (created_at,password,salt,status,updated_at,username) values (?,?,?,?,?,?)
Hibernate: select last_insert_rowid()
2024-02-27T20:01:04.143+08:00 ...SqliteController : init user GEeX
Hibernate: insert into website_user (created_at,password,salt,status,updated_at,username) values (?,?,?,?,?,?)
Hibernate: select last_insert_rowid()
2024-02-27T20:01:04.146+08:00 ...SqliteController : init user ZQrT
```

### 请求查询用户接口

```plain
$ curl http://127.0.0.1:8080/sqlite/find\?username\=ZQrT
WebsiteUser(id=10, username=ZQrT, password=538ea3b5fbacd1f9354a1f367b36135a, salt=RxaivBHlyJCxtOEv, status=active, createdAt=2024-02-27T20:01:04.144, updatedAt=2024-02-27T20:01:04.144)
```

查询成功，回显了查询到的用户信息。

### 请求登录接口

在初始化数据时，密码统一配置为 123456，下面的测试可以看到使用正确的密码可以通过校验。

```plain
$ curl http://127.0.0.1:8080/sqlite/login\?username\=ZQrT\&password\=123456
login succeeded
$ curl http://127.0.0.1:8080/sqlite/login\?username\=ZQrT\&password\=12345
login failed
```

## SQLite 3 数据审查

使用 Sqlite3 命令行工具查看 SQLite 数据库内容。

```plain
$ ./sqlite3 springboot-sqlite-jpa.db
SQLite version 3.42.0 2023-05-16 12:36:15
Enter ".help" for usage hints.
sqlite> .tables
website_user

sqlite> .mode table
sqlite> select * from website_user;
+----+---------------+----------------------------------+------------------+--------+---------------+----------+
| id |  created_at   |             password             |       salt       | status |  updated_at   | username |
+----+---------------+----------------------------------+------------------+--------+---------------+----------+
| 1  | 1709035264074 | 4b2b68c0df77669540fc0a487d753400 | 1njFP8ykWmlu01Z8 | active | 1709035264074 | HUyz     |
| 2  | 1709035264120 | 7e6444d57f753cfa6c1592a17e68e66e | 9X3El5jQaMhrROSf | active | 1709035264120 | ifQU     |
| 3  | 1709035264124 | 1d24c4ddb351eb56f665adb13708f981 | Jn9IrT6MYqVqzpu8 | active | 1709035264124 | GBPK     |
| 4  | 1709035264126 | 960747cc48aeed71e8ff714deae42e87 | wq8pb1G9pIalGHwP | active | 1709035264126 | rytE     |
| 5  | 1709035264129 | cf1037b95a997a1b1b9d9aa598b9f96b | An0hwV2n9cN4wpOy | active | 1709035264129 | iATH     |
| 6  | 1709035264132 | b68d42108e5046bd25b74cda947e0ffc | EozfDAkpn5Yx4yin | active | 1709035264132 | ZQRW     |
| 7  | 1709035264134 | 78d4841af9a12603204f077b9bf30dcc | 2FRNQ2zWksJHOyX9 | active | 1709035264135 | cIPM     |
| 8  | 1709035264137 | 60b8051ca3379c569a3fb41ed5ff05aa | KpT3IGwWmhlWIUq7 | active | 1709035264137 | MemS     |
| 9  | 1709035264140 | 0ca0a2dce442315c11f5488c0127f905 | RhGOYnNEMYbnWoat | active | 1709035264140 | GEeX     |
| 10 | 1709035264144 | 538ea3b5fbacd1f9354a1f367b36135a | RxaivBHlyJCxtOEv | active | 1709035264144 | ZQrT     |
+----+---------------+----------------------------------+------------------+--------+---------------+----------+
sqlite> 
```

一如既往，文章中代码存放在 Github.com/niumoo/javaNotes\[5\].

**参考**

-   • https://docs.spring.io/spring-data/jpa/reference/jpa.html
    

#### 引用链接

`[1]` SQLite 入门教程 (https://www.wdbyte.com/db/sqlite/): *https://www.wdbyte.com/db/sqlite/*  
`[2]` Spring Data JPA 官网：https://spring.io/projects/spring-data-jpa: *https://spring.io/projects/spring-data-jpa*  
`[3]` Spring Boot 使用 Spring Data JPA: *https://www.wdbyte.com/2019/03/springboot/springboot-10-data-jpa*  
`[4]` https://start.spring.io/: *https://start.spring.io/#!type=maven-project&language=java&platformVersion=3.2.3&packaging=jar&jvmVersion=21&groupId=com.example&artifactId=springboot-sqlite-jpa&name=springboot-sqlite-jpa&description=Demo%20project%20for%20Spring%20Boot&packageName=com.example.springboot-sqlite-jpa&dependencies=web,lombok,data-jpa*  
`[5]` Github.com/niumoo/javaNotes: *https://github.com/niumoo/JavaNotes/tree/master/springboot/springboot-sqlite-jpa*

  

**最后的话**

> 文章已经开源在 Github.com/niumoo/JavaNotes，欢迎 Star 和建议。

\---- END ----
