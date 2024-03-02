

# 快速审计记录 (五)--某 blog 系统 - 先知社区

快速审计记录 (五)--某 blog 系统

- - -

# SQL 注入

首先去 mapper.xml 中去查找是否有`${`进行拼接的情况  
[![](assets/1708920458-90e61cd36a57db8d3c9311b62840b6fd.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707814441908-0737b8b5-f611-48ec-a484-35290f7e419d.png#averageHue=%23253034&clientId=u7eb54710-2f09-4&from=paste&height=198&id=lG49E&originHeight=297&originWidth=846&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=39520&status=done&style=none&taskId=u55f7536d-c600-4cf8-a305-23a8eaed69e&title=&width=564)  
找出对应的 dao 层  
[![](assets/1708920458-d1ced98d5fa78610cecc050092e2628c.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707814533724-db31fb27-7a51-4b71-9bd3-f73e32c1c607.png#averageHue=%23202225&clientId=u7eb54710-2f09-4&from=paste&height=50&id=uc5bf10ef&originHeight=75&originWidth=1333&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=12412&status=done&style=none&taskId=ud92a874d-7476-4c9f-867c-8eac21c2961&title=&width=888.6666666666666)  
调用方法为 update 方法  
[![](assets/1708920458-4dba4bae6d7b9a01416a22355499a56b.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707814631458-d7d2838c-d0db-43f6-852c-d368748328dd.png#averageHue=%23212226&clientId=u7eb54710-2f09-4&from=paste&height=462&id=ua1a29a5a&originHeight=693&originWidth=1191&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=119211&status=done&style=none&taskId=u3df43a4d-8155-4c83-aabf-203005d4362&title=&width=794)  
到保存文章的界面  
[![](assets/1708920458-fc83d97ba54abc70740fcd1c612d4bbd.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707814673304-40182591-d39a-45ee-a0c4-ac2830dd6a1b.png#averageHue=%23202226&clientId=u7eb54710-2f09-4&from=paste&height=497&id=ub9952d79&originHeight=745&originWidth=1387&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=149163&status=done&style=none&taskId=u33c65047-29c3-4c0b-ba8c-dbe8996ac00&title=&width=924.6666666666666) 本以为有希望，但是发现在 xml 中`parameterType="java.lang.Integer"`只允许接受 Integer 类型的参数  
之后找到一个`Example_Where_Clause`但是全局搜索并未找到 dao 层，估计是用来做示例的  
[![](assets/1708920458-6b0ff8cbb77f1cb2c50989aee6983f3e.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707815718254-9d00726e-d70f-4f54-b4d9-0ee79c5529d1.png#averageHue=%2328383c&clientId=u7eb54710-2f09-4&from=paste&height=502&id=u3766b4c6&originHeight=753&originWidth=1251&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=116511&status=done&style=none&taskId=uea1987ca-a5a0-4ec2-9f43-d1faf09c761&title=&width=834)  
最终找到一个  
[![](assets/1708920458-7529ae71db3a24969f3297a76eefb466.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707815828822-e1b937e2-773a-496d-b888-1b2b5d7ed64e.png#averageHue=%23263438&clientId=u7eb54710-2f09-4&from=paste&height=355&id=u736c6f4e&originHeight=532&originWidth=1336&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=80313&status=done&style=none&taskId=ucdcbd45a-63d7-4af5-9ae5-18db5349e86&title=&width=890.6666666666666)  
对应的 dao  
[![](assets/1708920458-9aba946491418fc9fa823318df73c855.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707815856162-d133582f-99eb-4f8a-8794-5f73b69c57b3.png#averageHue=%2327343c&clientId=u7eb54710-2f09-4&from=paste&height=36&id=u27f436ab&originHeight=54&originWidth=660&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=9662&status=done&style=none&taskId=u00757f9d-8fd9-4671-ae3a-9fc112cfbca&title=&width=440)  
对应的 service  
[![](assets/1708920458-1ccecc476784cbe142b239cdcc9d4c7f.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707816002987-5e3ff27c-398f-4989-b056-5072015c3f2a.png#averageHue=%23212327&clientId=u7eb54710-2f09-4&from=paste&height=260&id=ued5d3310&originHeight=390&originWidth=850&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=43598&status=done&style=none&taskId=u9b2ecb34-8443-4e19-a200-af3bca3c5c6&title=&width=566.6666666666666)  
但是经过查看代码，这种`xxxByExample`都是不可控的。细看这个 service 都是传入`themeName`到那时传入`selectByExample`的是`themeExample`

# 网站注册处逻辑处理错误导致用户登录异常

此 blog 在第一次使用的时候会出现用户注册的情况  
[![](assets/1708920458-9417e98507a7690b4f63d5c3ba18b0cb.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707817734764-b83c0660-9a64-44c7-ae0b-7c3398ac0009.png#averageHue=%23e9eef4&clientId=u7eb54710-2f09-4&from=paste&height=341&id=ua1fd012d&originHeight=511&originWidth=1335&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=90936&status=done&style=none&taskId=uee2d5016-3bdc-46ee-a73f-944aaab2daf&title=&width=890)  
再次重发  
[![](assets/1708920458-ad2c7f07610a088a7b4c34860841ade9.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707817869568-2d7fc18c-12ee-4023-8d9e-b3e2847f2410.png#averageHue=%23eaeef3&clientId=u7eb54710-2f09-4&from=paste&height=466&id=u2db440ff&originHeight=699&originWidth=1315&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=124517&status=done&style=none&taskId=u071dc63d-aa22-4613-927c-0d805aa9d26&title=&width=876.6666666666666)  
发现出现错误  
去查看代码处理  
[![](assets/1708920458-d44eab237f26f95fecda610d16108971.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707817957165-27d162dc-7535-433b-9fa2-f85f94c4381f.png#averageHue=%2321252c&clientId=u7eb54710-2f09-4&from=paste&height=414&id=uf3b652bd&originHeight=621&originWidth=1737&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=179453&status=done&style=none&taskId=u1dfd19c9-dc61-4054-9d83-314e3c12be6&title=&width=1158)  
并未做多次写入的判断，直接进行了保存  
[![](assets/1708920458-e37216e176473ad10473b0284b59c8c4.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707818026651-7a4b60aa-db2c-49e2-93ba-ceaceba990cc.png#averageHue=%23202226&clientId=u7eb54710-2f09-4&from=paste&height=699&id=ud6471f76&originHeight=1048&originWidth=1405&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=210962&status=done&style=none&taskId=u0371decc-26ef-4996-bb07-e81b382a398&title=&width=936.6666666666666)  
两次`userService.save``optionsService.save`都进行了保存，在进行`insert`操作的时候出现了错误，导致前端回显错误  
查看数据库表时候，可以看到成功插入  
[![](assets/1708920458-89f5db7df6a6d1f924f918a300f1e2a8.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707818113101-ebf9f2fe-64ae-4ea2-b495-c58da25391a8.png#averageHue=%23fcfbfb&clientId=u7eb54710-2f09-4&from=paste&height=417&id=ubc1ac201&originHeight=625&originWidth=1408&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=77659&status=done&style=none&taskId=ue8ae0be4-a8c6-43e0-8639-7d602747b7d&title=&width=938.6666666666666)  
之后进行登录时出现错误  
[![](assets/1708920458-748ff96f5cc5ce37b942678ba78ba797.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707818187974-0b1548b7-b923-48b9-aad1-6b82b12e5dd8.png#averageHue=%23ebeff4&clientId=u7eb54710-2f09-4&from=paste&height=332&id=ued12dd63&originHeight=498&originWidth=1315&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=84717&status=done&style=none&taskId=uc704af4d-c362-4684-a007-69762dbe265&title=&width=876.6666666666666)  
查看报错  
[![](assets/1708920458-a9558a0621efddd5379e432050a90ca5.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707818255869-db1607a4-90f8-45ce-a984-e3de602122a4.png#averageHue=%23222428&clientId=u7eb54710-2f09-4&from=paste&height=515&id=uf30e4e45&originHeight=772&originWidth=2184&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=267240&status=done&style=none&taskId=ud923a265-5b21-4140-ae1f-cafa5cea2a2&title=&width=1456)  
应该返回一个或者 null 但是返回了多个结果，导致用户无法登录。可直接影响管理员用户的登录。

# 任意文件上传

此网站的所有上传接口均为此接口  
[![](assets/1708920458-96a8853c5e507d779260e8d8386899b8.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707819567394-069d59c7-052c-4386-b82e-2d96a471bf00.png#averageHue=%23eaeef3&clientId=u7eb54710-2f09-4&from=paste&height=309&id=uce7c06f4&originHeight=463&originWidth=1471&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=78861&status=done&style=none&taskId=uc86aecae-e093-4e8b-89c2-b7f47181be4&title=&width=980.6666666666666)  
查看上传文件详情的时候可以看到路径  
[![](assets/1708920458-58f80967793e648748ff616e1ae1d288.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707819782818-9962964d-7b98-4b99-aa15-d9b2d24511df.png#averageHue=%23fefefe&clientId=u7eb54710-2f09-4&from=paste&height=584&id=ubae766c8&originHeight=876&originWidth=1738&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=60245&status=done&style=none&taskId=u370e69f1-db3d-4845-b22d-d5cff953e71&title=&width=1158.6666666666667)  
查看代码  
[![](assets/1708920458-eeb46b15d925192b0f44aa75bab342b9.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707819679146-4e155e47-52f4-41bd-8edf-fb25c4cfe184.png#averageHue=%23202125&clientId=u7eb54710-2f09-4&from=paste&height=97&id=ue5e821ea&originHeight=145&originWidth=1552&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=23416&status=done&style=none&taskId=u8e3344d9-84f3-420c-b615-ea83d757f84&title=&width=1034.6666666666667)  
调用`uploadAttachment`方法  
[![](assets/1708920458-a69877fc859f36155a910f1b5cb8c74d.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707819724189-01b32ba7-e58d-4cda-b4ac-569bf967e464.png#averageHue=%23202226&clientId=u7eb54710-2f09-4&from=paste&height=756&id=u53c237e1&originHeight=1134&originWidth=1674&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=277714&status=done&style=none&taskId=u8839ae06-2629-4428-8550-b77beda8e9a&title=&width=1116)  
主要对上传文件进行了重命名和获取后缀，也就是无法通过`../`进行目录穿越，只能上传 webshell，但是此 blog 不解析 jsp

# 失败的任意文件删除

[![](assets/1708920458-4c0dfab5c65f699d474d0ca67b6a2a95.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707819901462-27e85fd9-2e49-47bf-bbf2-fe555d82dec9.png#averageHue=%23ebeef4&clientId=u7eb54710-2f09-4&from=paste&height=326&id=ueb814f87&originHeight=489&originWidth=1026&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=57914&status=done&style=none&taskId=u59e73fdb-071d-4552-aa8f-2fa3472e3af&title=&width=684)  
删除处通过上传的时候赋予的 id 进行删除操作，查看代码  
[![](assets/1708920458-34fedc5c88a96ecd0a9ac8163d9ff242.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707820017468-9e986821-bce8-4ab4-9454-25fb7a0522db.png#averageHue=%23202329&clientId=u7eb54710-2f09-4&from=paste&height=655&id=u0e50f2e9&originHeight=982&originWidth=1917&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=220961&status=done&style=none&taskId=u458d7df7-f7ce-4015-ac83-e929f7e4fd5&title=&width=1278)  
可以看到 id 的类型为 int 无法进行目录穿越进行任意文件删除

# fastjson

[![](assets/1708920458-17d52abbe8750a4d3c3924772ec28600.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707820751933-4e98e0fe-ee63-48ed-a181-ae731c734bb6.png#averageHue=%231f2124&clientId=u7eb54710-2f09-4&from=paste&height=308&id=ue7ca0275&originHeight=462&originWidth=1081&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=61752&status=done&style=none&taskId=uef64ab03-2215-423a-b0fa-f95d1c8e76f&title=&width=720.6666666666666)  
1.2.83，但是代码中并未使用 paresobject  
[![](assets/1708920458-b1fc4a5f069b99e3256794a3c0586c98.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707820881907-1b67510e-adcc-4f30-a286-6b49f96e5456.png#averageHue=%232e3239&clientId=u65c6d588-2f43-4&from=paste&height=313&id=u3adbff2d&originHeight=469&originWidth=990&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=110408&status=done&style=none&taskId=u73ec69e0-7272-4941-be64-45bc5e7134a&title=&width=660)  
参考[https://y4tacker.github.io/2023/04/26/year/2023/4/FastJson%E4%B8%8E%E5%8E%9F%E7%94%9F%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96-%E4%BA%8C/](https://y4tacker.github.io/2023/04/26/year/2023/4/FastJson%E4%B8%8E%E5%8E%9F%E7%94%9F%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96-%E4%BA%8C/)

# 失败的未授权记录

[![](assets/1708920458-3bf60f9b1582c91b01b1839759babd52.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707829885269-2145b7d0-aa25-41e2-b220-44abd1b08ec6.png#averageHue=%23202226&clientId=u65c6d588-2f43-4&from=paste&height=207&id=uf6e227bc&originHeight=310&originWidth=1530&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=66040&status=done&style=none&taskId=u4c53b968-4aed-4220-9840-ec81ba9717a&title=&width=1020)  
鉴权系统为 Spring Web MVC 拦截器  
这里主要看`excludePathPatterns`要走`loginAuthenticator`和`installInterceptor`  
[![](assets/1708920458-c30b88720362bc7b05a923ee9adf7377.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707830088722-2ace49da-771d-4854-94f7-19f7baef7fd1.png#averageHue=%231f2024&clientId=u65c6d588-2f43-4&from=paste&height=334&id=ub48b2f3a&originHeight=501&originWidth=1419&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=63820&status=done&style=none&taskId=u8d6f60ab-4afb-421b-9cd4-538e87c751b&title=&width=946)  
[![](assets/1708920458-f5481fa06dfdb11e3dc3b5ac974436c9.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707830104091-b0ba3ca4-9580-4fdc-871e-3bdb0ced4247.png#averageHue=%231f2124&clientId=u65c6d588-2f43-4&from=paste&height=286&id=u5a6b0e3e&originHeight=429&originWidth=1431&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=52126&status=done&style=none&taskId=u80f7e2d5-fbca-4ce7-9d2a-3e2424213e4&title=&width=954) 通过代码得知，`loginAuthenticator`必须要有 user 的 session 才可以返回 true，`installInterceptor`必须已经注册，这里我想到是否可以通过`/install/../admin/`进行未授权调用 admin 接口，这里却直接解析到`loginAuthenticator`  
[![](assets/1708920458-129c8ce4cfadcdcfaf7ccde7ba945bee.png)](https://cdn.nlark.com/yuque/0/2024/png/21762749/1707830361270-9ecb2631-75f2-4ab7-ad92-c9f7f83bbbc0.png#averageHue=%2320252e&clientId=u65c6d588-2f43-4&from=paste&height=253&id=u41630222&originHeight=379&originWidth=1864&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=77234&status=done&style=none&taskId=u76f49ba3-b8ff-4eb1-b6ed-50d1476c0a9&title=&width=1242.6666666666667)
