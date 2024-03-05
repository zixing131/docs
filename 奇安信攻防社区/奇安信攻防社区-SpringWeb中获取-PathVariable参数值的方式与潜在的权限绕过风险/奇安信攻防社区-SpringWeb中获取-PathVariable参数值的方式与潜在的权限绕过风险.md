

# 奇安信攻防社区-SpringWeb中获取@PathVariable参数值的方式与潜在的权限绕过风险

### SpringWeb中获取@PathVariable参数值的方式与潜在的权限绕过风险

在某些业务场景中，会获取@PathVariable 参数值，根据具体的角色权限来进行对比，以达到访问不同的数据的效果。浅谈SpringWeb中获取@PathVariable 参数值的方式与潜在的权限绕过风险。

# 0x00 前言

在实际业务中，为了防止越权操作，通常会根据对应的URL进行相关的鉴权操作。前面总结和SpringWeb中获取当前请求路径的方式以及路径前缀的方式，具体可以参考[https://forum.butian.net/share/2606](https://forum.butian.net/share/2606) 、[https://forum.butian.net/share/2761](https://forum.butian.net/share/2761) 。

除了上述场景外，还发现在某些业务场景中，会获取@PathVariable 参数值，根据具体的角色权限来进行对比，以达到访问不同的数据的效果。下面看看SpringWeb中是如何获取 @PathVariable 参数值的。

# 0x01 获取 @PathVariable 参数值的方式

在SpringWeb中，一般获取@PathVariable 参数值的方式主要有以下几种：

-   HandlerMapping属性对应的方法
-   AntPathMatcher和PathPattern自带方法

下面详细看看具体的解析逻辑：

## 1.1 HandlerMapping属性

以spring-webmvc-5.3.27为例，查看具体的实现：

### 1.1.1 URI\_TEMPLATE\_VARIABLES\_ATTRIBUTE

在获取Controller资源的过程中，RequestMappingHandlerMapping调用getHandler方法，找到匹配当前请求的HandlerMethod，然后处理匹配到的HandlerMapping，进而解析路径的参数。

主要是在org.springframework.web.servlet.mvc.method.RequestMappingInfoHandlerMapping#handleMatch方法进行处理的：

![image.png](assets/1709643457-d2f62d1548ee49448e1f84cf06ecbaef.png)

首先会获取模式匹配的条件，然后根据不同条件，调用extractMatchDetails方法选择使用PathPattern还是AntPathMatcher解析模式提取请求路径里的详细信息：

![image.png](assets/1709643457-fc475ea81ec283f082398e2ac42c598d.png)

下面分别是两种解析模式的提取过程：

-   **PathPattern解析模式**

若PathPatternsRequestCondition模式不为空的话进入对应的解析流程：

![image.png](assets/1709643457-57317b906d99a645282aa30187cf54f1.png)

首先获取当前请求的路径path，这里跟获取Controller资源的过程中的通过PathPattern匹配前获取请求路径的方式是一致的：

![image.png](assets/1709643457-da06ec363f3f11de8a870de4a7a9938a.png)

![image.png](assets/1709643457-3e76e8d5e1fc20b135b44e54c686205a.png)

获取到请求路径后，通过PathPattern模式的matchAndExtract方法来提取变量，提取完的变量可以通过getUriVariables方法获取，最终将对应的值分别设置在request的MATRIX\_VARIABLES\_ATTRIBUTE和URI\_TEMPLATE\_VARIABLES\_ATTRIBUTE中：

![image.png](assets/1709643457-192bb257fc2c1ccb4297325cba97451f.png)

-   **AntPathMatcher解析模式**

前面的逻辑跟PathPattern的情况是类似的，只是这里不重新获取请求path，而是直接对lookupPath进行处理：

![image.png](assets/1709643457-d0688af938494c897b735c000961eea1.png)

lookupPath主要通过initLookupPath获取用于初始化请求映射的路径：

![image.png](assets/1709643457-5f1cf7aa1c88f5884481708ec280d830.png)

然后AntPathMatcher解析匹配，通过其extractUriTemplateVariables方法来提取变量检验UrlPathHelper中的removeSemicolonContent参数，removeSemicolonContent默认是true，默认不会提取MatrixVariable变量，因此需要设置为false，才能进入该分支，这里主要是对@MatrixVariable进行处理：

![image.png](assets/1709643457-798243f75a2fb9988ef0a9f42a407070.png)

最后调用UrlPathHelper的decodePathVariables方法对提取出来的参数进行处理，并设置request的URI\_TEMPLATE\_VARIABLES\_ATTRIBUTE属性：

![image.png](assets/1709643457-709788a123a8d3258950d165f80c3ed3.png)

在decodePathVariables方法中，会根据urlDecode确定是否需要对请求路径里参数值进行URL解码操作：

![image.png](assets/1709643457-b3dfa0a7d0aba1e918fc0fa1dd85131a.png)

![image.png](assets/1709643457-f951c21efb2fc2cf0338af5b8f85dc01.png)

## 1.2 不同解析模式自带的方法

### 1.2.1 AntPathMatcher

-   **org.springframework.util.AntPathMatcher#extractUriTemplateVariables**

以spring-web-5.3.27.jar为例，查看具体的实现，这里会直接调用org.springframework.util.AntPathMatcher#doMatch方法进行处理，并且将对应的参数及内容封装到传入的variables参数中：

![image.png](assets/1709643457-198c53f52f3afa8664d8e7f0428eb471.png)

首先调用tokenizePattern()方法将pattern分割成了String数组，如果是全路径并且区分大小写,那么就通过简单的字符串检查，看看path是否有潜在匹配的可能，没有的话返回false，然后调用tokenizePath()方法将需要匹配的path分割成string数组,主要是通过java.util 里面的StringTokenizer来处理字符串：

![image.png](assets/1709643457-b3391571527c0682a06b884b4c032d4b.png)

然后将pathDirs和pattDirs两个数组从左到右开始匹配，主要是一些正则的转换还有通配符的匹配，这里调用了org.springframework.util.AntPathMatcher#matchStrings方法来处理URI中对应的参数内容：

![image.png](assets/1709643457-d7455868af15bd25ccc6777bd1fbccc7.png)

实际调用的是AntPathStringMatcher#matchStrings进行处理:

![image.png](assets/1709643457-1b4181cf8529944a405f44f935d5230c.png)

在matchStrings方法中，首先会根据pattern进行匹配：

![image.png](assets/1709643457-0de5e589af87ad85cb054ac53d5940bb.png)

可以看到对于@PathVariable场景，对应Pattern为`((?s).*)`，代表匹配包括换行符在内的任意字符：

![image.png](assets/1709643457-ed2bd6371e7af74e26d859ca6f1965ff.png)

匹配过程主要做了两个检查：

-   查看参数列表的数量与匹配的数量是否一致，如果不一致则抛出异常
-   然后遍历匹配的结果，这里首先会对参数列表里的参数名进行检查，如果以`*`开头的话则抛出对应的异常（主要是区分PathPattern的新特性，`{*spring}`表示匹配余下的path路径部分并将其赋值给名为spring的变量，变量名可以根据实际情况随意命名，与`@PathVariable`名称对应即可）

最后将匹配的结果封装到Map结构中，并返回：

![image.png](assets/1709643457-3a3d0d58e440db27ec58a7ce460436cf.png)

到这里已经将对应的参数及内容封装到传入的variables参数。

### 1.2.2 PathPattern

-   **org.springframework.web.util.pattern.PathPattern#matchAndExtract**

以spring-web-5.3.27.jar为例，查看具体的实现，首先会创建MatchingContext对象，然后调用matches方法在匹配过程中完成请求路径中参数的提取：

![image.png](assets/1709643457-2580bfe325254339f1e380a16a0021b6.png)

这里会根据PathPattern的链式节点中对应的PathElement的matches方法逐个进行匹配，其中对于@PathVariable的处理是在org.springframework.web.util.pattern.CaptureVariablePathElement#matches完成的。

通过matchingContext.pathElementValue获取当前节点的值并保存在candidateCapture参数中，然后通过java.util.regex.compile#matcher对匹配到的内容进行检查：

![image.png](assets/1709643457-c086bd2313f222ef0b66d0569c4df2eb.png)

最终调用set方法将candidateCapture封装：

![image.png](assets/1709643457-8751005f1aea7548dba880765047ce73.png)

![image.png](assets/1709643457-25934101e38d5e7939bddc55173a1c8e.png)

匹配结束后通过getPathMatchResult方法返回对应的刚刚封装的解析结果：

![image.png](assets/1709643457-9ef67c4ca62c2471281b5e02c2793136.png)

![image.png](assets/1709643457-faee1b703d9493b5dbd13f2230255862.png)

在创建PathElement链式结构时，会有一些处理：

-   **默认会对路径中的分号进行截断处理：**

![image.png](assets/1709643457-51f875e4450ce63d4d0bb0f051fcf516.png)

-   **默认情况下会进行URL解码操作,主要是通过decodeAndParseSegments属性控制的（默认是true）：**

![image.png](assets/1709643457-64d6986908703e81bd24ecf6871de04a.png)

# 0x02 潜在的绕过风险

本质上还是由于解析差异导致的绕过风险。在SpringWeb中绕过的方式也是大同小异。下面看一个实际遇到的例子：

首先通过HandlerMapping的PATH\_WITHIN\_HANDLER\_MAPPING\_ATTRIBUTE属性来获取请求的path，然后通过BEST\_MATCHING\_PATTERN\_ATTRIBUTE属性获取到最佳匹配的Controller。最后通过AntPathMatcher#extractUriTemplateVariables方法来提取@PathVariable 参数值，然后对对应的参数&值进行对应的检查：

```Java
String reqPath = (String) request.getAttribute(HandlerMapping.PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE);
String bestMatchPattern = (String) request.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
Map<String, String> map =  new AntPathMatcher().extractUriTemplateVariables(bestMatchPattern, reqPath);
```

类似`/admin/{param}/index`,当请求`/admin/manage/index`时会获取到的@PathVariable 参数值为manage。

Spring在获取到请求路径后，会调用lookupHandlerMethod方法，首先直接根据路径获取对应的Mapping，获取不到的话调用addMatchingMappings遍历所有的ReuqestMappingInfo对象并进行匹配。匹配完成后，会存储请求相对于Controller映射的路径：

![image.png](assets/1709643457-5eafc8f9c8b1a9e11d1ab74a399f70b9.png)

可以看到PATH\_WITHIN\_HANDLER\_MAPPING\_ATTRIBUTE属性值主要跟lookupPath有关。而lookupPath实际上还是从initLookupPath方法获取的：

![image.png](assets/1709643457-c06bd5013035825f0de2848642ac692f.png)

当使用PathPattern进行解析时，仅会移除URL路径中分号之后还有移除URL路径中的JSESSIONID参数。所以当对请求Path进行URL编码时，获取到的Path并不会进行URL解码处理，解码操作仅仅在PathPattern链式匹配时才会进行。

根据前面的分析AntPathMatcher#extractUriTemplateVariables方法在匹配时本身也不会进行URL解码操作。那么这里上述提取@PathVariable 参数值则存在一个解析差异的问题：

-   低版本使用AntPathMatcher进行路由匹配时，通过PATH\_WITHIN\_HANDLER\_MAPPING\_ATTRIBUTE属性值是经过URL解码后的，此时类似`/admin/{param}/index`,当请求`/admin/manage/index`时会获取到的@PathVariable 参数值为manage
-   高版本使用PathPattern进行路由匹配时，通过PATH\_WITHIN\_HANDLER\_MAPPING\_ATTRIBUTE属性值是没有经过解码的。此时类似`/admin/{param}/index`,当请求`/admin/%6d%61%6e%61%67%65/index`时会获取到的@PathVariable 参数值为%6d%61%6e%61%67%65。但是在PathPattern链式匹配时会进行解码，并不会影响正常的业务调用，此时若后续的鉴权判断逻辑是直接遍历map的key-value进行类似equals判断时，明显会存在绕过的风险。

# 0x03 其他

PathVariableMethodArgumentResolver主要用于解析@PathVariable注解，一般会把解析后的参数名传递给 resolveName() 方法，从请求中获取参数值，可以看到其也是通过HandlerMapping的URI\_TEMPLATE\_VARIABLES\_ATTRIBUTE来获取对应的值的：

![image.png](assets/1709643457-2219f0882b3fe4adc47bc56421f39d4a.png)
