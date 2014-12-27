---
layout: post
category: "java"
title:  "Json序列化利器FastJSON"
tags: [java,json,fastjson,serialize]
---

首先介绍一篇文章， 【问底】静行：FastJSON实现详解
http://www.csdn.net/article/2014-09-25/2821866

这篇文章讲到FastJSON通过createJavaBeanSerializer()创建的序列化类。通过该方法会获取两种序列化类，一种是直接的JavaBeanSerializer（根据类的get方法、public filed等JavaBean特征序列化），另一种是createASMSerializer（通过ASM框架生成的序列化字节码），优先使用第二种。

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

看了FastJSON实现后，发现这种序列化方式和ProtoBuf很类似。在序列化类的基础上进行封装，就没有那么多的反射了（反射挺消耗时间的）

我们这里来分析下： 首先将类ASMSerializerFactory中方法createJavaBeanSerializer的IOUtils.write注释代码取消。修改下输出路径。

<pre class="prettyPrint">
//
 org.apache.commons.io.IOUtils.write(code, new java.io.FileOutputStream(
 "/Users/jiaojianfeng/Documents/empleyment/tmp/fastjson/"
 + className + ".class"));
</pre>

我们将生成的Serialize0.class反编译后得到：

<pre class="prettyPrint">
import com.alibaba.fastjson.serializer.FilterUtils;
import com.alibaba.fastjson.serializer.JSONSerializer;
import com.alibaba.fastjson.serializer.JavaBeanSerializer;
import com.alibaba.fastjson.serializer.ObjectSerializer;
import com.alibaba.fastjson.serializer.SerialContext;
import com.alibaba.fastjson.serializer.SerializeWriter;
import com.alibaba.fastjson.serializer.SerializerFeature;
import com.alibaba.fastjson.util.ASMUtils;
import com.jjf.Person;
import java.io.IOException;
import java.lang.reflect.Type;

public class Serialize0 implements ObjectSerializer {
    private JavaBeanSerializer nature;
    public Type name_asm_fieldPrefix;
    public Type name_asm_fieldType = ASMUtils.getMethodType(Person.class,
            "getName");
    public Type age_asm_fieldPrefix;
    public Type age_asm_fieldType = ASMUtils.getMethodType(Person.class,
            "getAge");

    public void write(JSONSerializer paramJSONSerializer, Object paramObject1,
            Object paramObject2, Type paramType) throws IOException {
        SerializeWriter localSerializeWriter = paramJSONSerializer.getWriter();
        if (localSerializeWriter.isEnabled(SerializerFeature.SortField)) {
            write1(paramJSONSerializer, paramObject1, paramObject2, paramType);
            return;
        }
        Person localPerson = (Person) paramObject1;
        if (localSerializeWriter.isEnabled(SerializerFeature.PrettyFormat)) {
            if (this.nature == null)
                this.nature = new JavaBeanSerializer(Person.class);
            this.nature.write(paramJSONSerializer, paramObject1, paramObject2,
                    paramType);
            return;
        }
        if (paramJSONSerializer.containsReference(paramObject1)) {
            if (this.nature == null)
                this.nature = new JavaBeanSerializer(Person.class);
            this.nature.writeReference(paramJSONSerializer, paramObject1);
            return;
        }
        if (paramJSONSerializer.isWriteAsArray(paramObject1, paramType)) {
            writeAsArray(paramJSONSerializer, paramObject1, paramObject2,
                    paramType);
            return;
        }
        SerialContext localSerialContext = paramJSONSerializer.getContext();
        paramJSONSerializer.setContext(localSerialContext, paramObject1,
                paramObject2);
        localSerializeWriter.write("{\"@type\":\"com.jjf.Person\"");
        char c = (paramJSONSerializer.isWriteClassName(paramType, paramObject1))
                && (paramType != paramObject1.getClass()) ? ',' : '{';
        c = FilterUtils.writeBefore(paramJSONSerializer, paramObject1, c);
        String str1 = "name";
        Object localObject1;
        Object localObject2;
        if (FilterUtils.applyName(paramJSONSerializer, paramObject1, str1)) {
            String str2 = localPerson.getName();
            if (FilterUtils
                    .apply(paramJSONSerializer, paramObject1, str1, str2)) {
                str1 = FilterUtils.processKey(paramJSONSerializer,
                        paramObject1, str1, str2);
                localObject1 = str2;
                localObject2 = FilterUtils.processValue(paramJSONSerializer,
                        paramObject1, str1, localObject1);
                if (localObject1 != localObject2) {
                    if (localObject2 == null) {
                        if (localSerializeWriter
                                .isEnabled(SerializerFeature.WriteMapNullValue)) {
                            localSerializeWriter.writeFieldNullString(c, str1);
                            c = ',';
                        }
                    } else {
                        localSerializeWriter.write(c);
                        localSerializeWriter.writeFieldName(str1);
                        paramJSONSerializer.writeWithFieldName(localObject2,
                                str1, this.name_asm_fieldType);
                        c = ',';
                    }
                } else if (str2 == null) {
                    if (localSerializeWriter
                            .isEnabled(SerializerFeature.WriteMapNullValue)) {
                        localSerializeWriter.writeFieldNullString(c, str1);
                        c = ',';
                    }
                } else {
                    localSerializeWriter.writeFieldValue(c, str1, str2);
                    c = ',';
                }
            }
        }
        str1 = "age";
        if (FilterUtils.applyName(paramJSONSerializer, paramObject1, str1)) {
            int i = localPerson.getAge();
            if (FilterUtils.apply(paramJSONSerializer, paramObject1, str1, i)) {
                str1 = FilterUtils.processKey(paramJSONSerializer,
                        paramObject1, str1, i);
                localObject1 = Integer.valueOf(i);
                localObject2 = FilterUtils.processValue(paramJSONSerializer,
                        paramObject1, str1, localObject1);
                if (localObject1 != localObject2) {
                    if (localObject2 == null) {
                        if (localSerializeWriter
                                .isEnabled(SerializerFeature.WriteMapNullValue)) {
                            localSerializeWriter.writeFieldNull(c, str1);
                            c = ',';
                        }
                    } else {
                        localSerializeWriter.write(c);
                        localSerializeWriter.writeFieldName(str1);
                        paramJSONSerializer.writeWithFieldName(localObject2,
                                str1);
                        c = ',';
                    }
                } else {
                    localSerializeWriter.writeFieldValue(c, str1, i);
                    c = ',';
                }
            }
        }
        c = FilterUtils.writeAfter(paramJSONSerializer, paramObject1, c);
        if (c == '{')
            localSerializeWriter.write('{');
        localSerializeWriter.write('}');
        paramJSONSerializer.setContext(localSerialContext);
    }

    public void write1(JSONSerializer paramJSONSerializer, Object paramObject1,
            Object paramObject2, Type paramType) throws IOException {
        SerializeWriter localSerializeWriter = paramJSONSerializer.getWriter();
        Person localPerson = (Person) paramObject1;
        if (localSerializeWriter.isEnabled(SerializerFeature.PrettyFormat)) {
            if (this.nature == null)
                this.nature = new JavaBeanSerializer(Person.class);
            this.nature.write(paramJSONSerializer, paramObject1, paramObject2,
                    paramType);
            return;
        }
        if (paramJSONSerializer.containsReference(paramObject1)) {
            if (this.nature == null)
                this.nature = new JavaBeanSerializer(Person.class);
            this.nature.writeReference(paramJSONSerializer, paramObject1);
            return;
        }
        if (paramJSONSerializer.isWriteAsArray(paramObject1, paramType)) {
            writeAsArray(paramJSONSerializer, paramObject1, paramObject2,
                    paramType);
            return;
        }
        SerialContext localSerialContext = paramJSONSerializer.getContext();
        paramJSONSerializer.setContext(localSerialContext, paramObject1,
                paramObject2);
        localSerializeWriter.write("{\"@type\":\"com.jjf.Person\"");
        char c = (paramJSONSerializer.isWriteClassName(paramType, paramObject1))
                && (paramType != paramObject1.getClass()) ? ',' : '{';
        c = FilterUtils.writeBefore(paramJSONSerializer, paramObject1, c);
        String str1 = "age";
        Object localObject1;
        Object localObject2;
        if (FilterUtils.applyName(paramJSONSerializer, paramObject1, str1)) {
            int i = localPerson.getAge();
            if (FilterUtils.apply(paramJSONSerializer, paramObject1, str1, i)) {
                str1 = FilterUtils.processKey(paramJSONSerializer,
                        paramObject1, str1, i);
                localObject1 = Integer.valueOf(i);
                localObject2 = FilterUtils.processValue(paramJSONSerializer,
                        paramObject1, str1, localObject1);
                if (localObject1 != localObject2) {
                    if (localObject2 == null) {
                        if (localSerializeWriter
                                .isEnabled(SerializerFeature.WriteMapNullValue)) {
                            localSerializeWriter.writeFieldNull(c, str1);
                            c = ',';
                        }
                    } else {
                        localSerializeWriter.write(c);
                        localSerializeWriter.writeFieldName(str1);
                        paramJSONSerializer.writeWithFieldName(localObject2,
                                str1);
                        c = ',';
                    }
                } else {
                    localSerializeWriter.writeFieldValue(c, str1, i);
                    c = ',';
                }
            }
        }
        str1 = "name";
        if (FilterUtils.applyName(paramJSONSerializer, paramObject1, str1)) {
            String str2 = localPerson.getName();
            if (FilterUtils
                    .apply(paramJSONSerializer, paramObject1, str1, str2)) {
                str1 = FilterUtils.processKey(paramJSONSerializer,
                        paramObject1, str1, str2);
                localObject1 = str2;
                localObject2 = FilterUtils.processValue(paramJSONSerializer,
                        paramObject1, str1, localObject1);
                if (localObject1 != localObject2) {
                    if (localObject2 == null) {
                        if (localSerializeWriter
                                .isEnabled(SerializerFeature.WriteMapNullValue)) {
                            localSerializeWriter.writeFieldNullString(c, str1);
                            c = ',';
                        }
                    } else {
                        localSerializeWriter.write(c);
                        localSerializeWriter.writeFieldName(str1);
                        paramJSONSerializer.writeWithFieldName(localObject2,
                                str1, this.name_asm_fieldType);
                        c = ',';
                    }
                } else if (str2 == null) {
                    if (localSerializeWriter
                            .isEnabled(SerializerFeature.WriteMapNullValue)) {
                        localSerializeWriter.writeFieldNullString(c, str1);
                        c = ',';
                    }
                } else {
                    localSerializeWriter.writeFieldValue(c, str1, str2);
                    c = ',';
                }
            }
        }
        c = FilterUtils.writeAfter(paramJSONSerializer, paramObject1, c);
        if (c == '{')
            localSerializeWriter.write('{');
        localSerializeWriter.write('}');
        paramJSONSerializer.setContext(localSerialContext);
    }

    public void writeAsArray(JSONSerializer paramJSONSerializer,
            Object paramObject1, Object paramObject2, Type paramType)
            throws IOException {
        SerializeWriter localSerializeWriter = paramJSONSerializer.getWriter();
        Person localPerson = (Person) paramObject1;
        localSerializeWriter.write('[');
        String str = "age";
        localSerializeWriter.writeIntAndChar(localPerson.getAge(), ',');
        str = "name";
        localSerializeWriter.writeString(localPerson.getName(), ']');
    }
}
</pre>

序列化时调用的就是write方法。读者可以将源码修改下，让其调用类Serialize0的write方法，方便调试，查看处理过程。

当然，FastJSON中还有很多优化方案，像工具类SerializeWriter、IdentityHashMap、sort field、sort field fast match 等等。可自行通过源码学习。



