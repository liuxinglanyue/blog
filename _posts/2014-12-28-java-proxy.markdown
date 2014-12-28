---
layout: post
category: "java"
title:  "Java动态代理原理-字节码"
tags: [java,class,reflect,proxy]
---

首先摘录一段话，讲述了啥是动态代理(来自http://docs.oracle.com/javase/7/docs/api/)：

A dynamic proxy class (simply referred to as a proxy class below) is a class that implements a list of interfaces specified at runtime when the class is created, with behavior as described below. A proxy interface is such an interface that is implemented by a proxy class. A proxy instance is an instance of a proxy class. Each proxy instance has an associated invocation handler object, which implements the interface InvocationHandler. A method invocation on a proxy instance through one of its proxy interfaces will be dispatched to the invoke method of the instance's invocation handler, passing the proxy instance, a java.lang.reflect.Method object identifying the method that was invoked, and an array of type Object containing the arguments. The invocation handler processes the encoded method invocation as appropriate and the result that it returns will be returned as the result of the method invocation on the proxy instance.

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

如何写Java动态代理的代码，google下，一堆堆。 官方api也有说明。

本文主要的目的是：怎么就可以代理了呢？如何实现的。

<pre class="prettyPrint">
package com.forName;

public interface EntityInter {

    public void print();
}
</pre>

这只是一个简单的接口，有个print方法。

<pre class="prettyPrint">
byte[] b = ProxyGenerator.generateProxyClass("com.forName.EntityProxy",
                new Class[] { EntityInter.class });

System.out.println(Hex.encodeHexStr(b));

org.apache.commons.io.IOUtils
        .write(b,
                new java.io.FileOutputStream(
                        "/Users/jiaojianfeng/Documents/empleyment/tmp/proxy/EntityProxy.class"));
</pre>

这里简单说明下，上面的代码。

类sun.misc.ProxyGenerator 调用静态方法generateProxyClass获得代理类的源码。

类IOUtils将源码输出到文件中，仅此而已。

然后，用JD-GUI工具，将class反编译得到下面的代码

<pre class="prettyPrint">
package com.forName;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class EntityProxy extends Proxy
  implements EntityInter
{
  private static Method m1;
  private static Method m0;
  private static Method m3;
  private static Method m2;

  public EntityProxy(InvocationHandler paramInvocationHandler)
    throws 
  {
    super(paramInvocationHandler);
  }

  public final boolean equals(Object paramObject)
    throws 
  {
    try
    {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final int hashCode()
    throws 
  {
    try
    {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final void print()
    throws 
  {
    try
    {
      this.h.invoke(this, m3, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  public final String toString()
    throws 
  {
    try
    {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }

  static
  {
    try
    {
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m3 = Class.forName("com.forName.EntityInter").getMethod("print", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }
}
</pre>

其他的方法，我们暂不关心，看print方法 this.h.invoke(this, m3, null);

h是Proxy的属性，是调用Proxy.newProxyInstance需要传得参数。

m3是通过反射得到的方法

其他的应该一看便知，end
