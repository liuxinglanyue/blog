---
layout: post
category: "java"
title:  "通过btrace对fastjson的性能测试"
tags: [java,fastjson,btrace,time]
---

首先需要从一个issues说起： 

FastJSON 1.1.33版本调用JSON.toJSONString(Object)方法性能问题（https://github.com/alibaba/fastjson/issues/272）

楼主说：我们的应用里定义了一个就一个层级的实体对象，有些时候序列化需要4秒多，造成了调用API超时。想咨询一下这个问题如何解决。下面附上实体类：
调用JSON.toJSONString(oAuthToken)花费了4秒多

<pre class="prettyPrint">
public class OAuthToken {
    private Long id;
    private Long userId;
    private Long appId;
    private String  accessToken;
    private Long expiresIn;
    private Integer scope;
    private String refreshToken;
    private Integer state;
    private Timestamp createTime;
    //省略get和set方法
}
</pre>

我这边测试了下，第一次toJSONString 花了349ms（第一次需要生成字节码，所以时间长），第二次0ms。

由于没有楼主的源码：所以这里写了btrace代码来分析fastjson（当然还可以直接下载fastjson源码，加上时间输出日志，更方便。前提：有源码）

附上我写的btrace代码：

<pre class="prettyPrint">
import static com.sun.btrace.BTraceUtils.*;

import com.sun.btrace.annotations.BTrace;
import com.sun.btrace.annotations.Kind;
import com.sun.btrace.annotations.Location;
import com.sun.btrace.annotations.OnMethod;
import com.sun.btrace.annotations.Return;
import com.sun.btrace.annotations.TLS;

@BTrace
public class BTraceFastJsonTest {

    @TLS
    private static long startTime_getObjectWriter;

    @OnMethod(clazz = "com.alibaba.fastjson.serializer.JSONSerializer", method = "getObjectWriter")
    public static void start_getObjectWriter() {
        startTime_getObjectWriter = timeMillis();
    }

    @TLS
    private static String serializerClazzName;

    @OnMethod(clazz = "com.alibaba.fastjson.serializer.JSONSerializer", method = "getObjectWriter", location = @Location(Kind.RETURN))
    public static void end_getObjectWriter(
            @Return com.alibaba.fastjson.serializer.ObjectSerializer result,
            Class<?> clazz) {
        println(str("--------------------1-----------------------"));
        serializerClazzName = name(classOf(result));
        println(strcat("JSONSerializer.getObjectWriter clazz:", str(clazz)));
        println(strcat("JSONSerializer.getObjectWriter ObjectSerializer:",
                serializerClazzName));
        println(strcat("JSONSerializer.getObjectWriter time:", str(timeMillis()
                - startTime_getObjectWriter)));
    }

    @TLS
    private static long startTime_write;

    @OnMethod(clazz = "com.alibaba.fastjson.serializer.JSONSerializer", method = "write")
    public static void start_write() {
        startTime_write = timeMillis();
    }

    @OnMethod(clazz = "com.alibaba.fastjson.serializer.JSONSerializer", method = "write", location = @Location(Kind.RETURN))
    public static void end_write(Object object) {
        println(str("---------------------3----------------------"));
        println(strcat("JSONSerializer.write object:", str(object)));
        println(strcat("JSONSerializer.write time:", str(timeMillis()
                - startTime_write)));
    }

    @TLS
    private static long startTime_serialize_write;

    @OnMethod(clazz = "Serializer_1", method = "write")
    public static void start_serialize_write() {
        startTime_serialize_write = timeMillis();
    }

    @OnMethod(clazz = "Serializer_1", method = "write", location = @Location(Kind.RETURN))
    public static void end_serialize_write() {
        println(str("-------------------2------------------------"));
        println(strcat("Serializer_1.write time:", str(timeMillis()
                - startTime_serialize_write)));
    }

    @TLS
    private static long startTime;

    @OnMethod(clazz = "com.alibaba.fastjson.JSON", method = "toJSONString")
    public static void start() {
        startTime = timeMillis();
    }

    @OnMethod(clazz = "com.alibaba.fastjson.JSON", method = "toJSONString", location = @Location(Kind.RETURN))
    public static void end() {
        println(str("-------------------4-----------------------"));
        println(strcat("JSON.toJsonString time:", str(timeMillis() - startTime)));
    }
}
</pre>

说明下：start_serialize_write、end_serialize_write这两个方法的clazz需要根据实际情况修改（即为：serializerClazzName）


