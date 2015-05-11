---
layout: post
category: "java"
title:  "unsafe中getAndAddInt性能疑问"
tags: [jvm,java,unsafe,cas,jdk,c]
---

首先说下为啥会有这篇文章，看了[jdk1.8中CAS的增强](http://ifeve.com/enhanced-cas-in-jdk8/)，对其中关于AtomicInteger.getAndIncrement在jdk8中的优化深入了解了下，文中提到 “用反射获取到Unsafe实例，编写了跟getAndAddInt相同的代码，但测试结果却跟jdk1.7的getAndIncrement一样慢，不知道Unsafe里面究竟玩了什么黑魔法，还请高人不吝指点”。这就是由来。

这里有个小插曲，我们来看下getAndIncrement方法

jdk7， AtomicInteger.getAndIncrement（如下）

这个方法不需要多说了吧。

<pre class="prettyPrint">
public final int getAndIncrement() {
    for (;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
            return current;
    }
}
</pre>

看下jdk8的AtomicInteger.getAndIncrement和unsafe.getAndAddInt

<pre class="prettyPrint">
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
</pre>

<pre class="prettyPrint">
public final int getAndAddInt(Object paramObject, long paramLong, int paramInt) {
    int i;
    do
      i = getIntVolatile(paramObject, paramLong);
    while (!(compareAndSwapInt(paramObject, paramLong, i, i + paramInt)));
    return i;
}
</pre>

getIntVolatile的源码在这里[sun.misc.natUnsafe.cc](https://github.com/aeste/gcc/blob/master/libjava/sun/misc/natUnsafe.cc)

<pre class="prettyPrint">
jint
sun::misc::Unsafe::getIntVolatile (jobject obj, jlong offset)
{
  volatile jint *addr = (jint *) ((char *) obj + offset);
  jint result = *addr;
  read_barrier ();
  return result;
}
</pre>

read_barrier是在此加个读屏障，不了解的复习下JVM内存模型

结论：通过jdk8中的getIntVolatile获取value值比通过jdk7中的get获取value值，能有更好的并发性能，jdk7中get获取的value在CAS时失败的几率更大

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-切入正题\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

根据文中的思路我们测试了jdk8中的getAndIncrement与自己编写的与getAndAddInt相同的代码。比较发现直接调用getAndIncrement快三倍(在多线程版本中，这里特别强调，如果是单线程，getIntVolatile也没有意义)

然后我就在思考，这是为什么呢？

#####1、 我想到了\-XX:CompileThreshold，当方法执行达到一定次数时，编译为机器码。
于是，我写了测试代码，发现差别不大

#####2、 然后我想到，看下各自编译后的字节码有何不同

rt.jar 解压后的
<pre class="prettyPrint">
public final int getAndAddInt(java.lang.Object, long, int);
    descriptor: (Ljava/lang/Object;JI)I
    flags: ACC_PUBLIC, ACC_FINAL
    Code:
      stack=7, locals=6, args_size=4
         0: aload_0
         1: aload_1
         2: lload_2
         3: invokevirtual #35                 // Method getIntVolatile:(Ljava/lang/Object;J)I
         6: istore        5
         8: aload_0
         9: aload_1
        10: lload_2
        11: iload         5
        13: iload         5
        15: iload         4
        17: iadd
        18: invokevirtual #36                 // Method compareAndSwapInt:(Ljava/lang/Object;JII)Z
        21: ifeq          0
        24: iload         5
        26: ireturn
      LineNumberTable:
        line 1031: 0
        line 1032: 8
        line 1033: 24
      StackMapTable: number_of_entries = 1
        frame_type = 0 /* same */
</pre>

自己编写的，经过javac -g ， javap -verbose 
<pre class="prettyPrint">
public final int getAndAddInt(java.lang.Object, long, int);
    descriptor: (Ljava/lang/Object;JI)I
    flags: ACC_PUBLIC, ACC_FINAL
    Code:
      stack=7, locals=6, args_size=4
         0: getstatic     #2                  // Field unsafe:Lsun/misc/Unsafe;
         3: aload_1
         4: lload_2
         5: invokevirtual #4                  // Method sun/misc/Unsafe.getIntVolatile:(Ljava/lang/Object;J)I
         8: istore        5
        10: getstatic     #2                  // Field unsafe:Lsun/misc/Unsafe;
        13: aload_1
        14: lload_2
        15: iload         5
        17: iload         5
        19: iload         4
        21: iadd
        22: invokevirtual #5                  // Method sun/misc/Unsafe.compareAndSwapInt:(Ljava/lang/Object;JII)Z
        25: ifeq          0
        28: iload         5
        30: ireturn
      LineNumberTable:
        line 46: 0
        line 47: 10
        line 48: 28
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      31     0  this   LAtomicIntegerByJJF;
            0      31     1 paramObject   Ljava/lang/Object;
            0      31     2 paramLong   J
            0      31     4 paramInt   I
           10      21     5     i   I
      StackMapTable: number_of_entries = 1
        frame_type = 0 /* same */
</pre>

仔细寻找，发现最大的差别就是unsafe的调用，getAndAddInt调用unsafe是通过aload_0。而自己写的方法呢，由于unsafe限制，只能通过反射获取，故程序中调用unsafe是通过getstatic

但是在接下来的测试发现，aload_0与getstatic并不会导致上面三倍的差距（单线程下，实际测下来可以说不相伯仲）

#####3、 这时候我就疑惑了，难道是JVM对unsafe做了多线程优化

#####4、 再次分析参考文章中的代码

#####5、 我能给出的结论是，多线程情况下，aload_0是从栈（线程私有）中获取unsafe引用，getstatic是从方法区（线程公有）中获取unsafe引用，so.....【正确性有待考证】

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-补充结论\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

看了这篇文章后[AtomicInteger Java 7 vs Java 8](http://ashkrit.blogspot.com/2014/02/atomicinteger-java-7-vs-java-8.html) 知道了为啥性能有差距了

unsafe的方法getAndAddInt，在执行时有专门的指令lock xadd。而反编译unsafe获得了getAndAddInt实现，然后自己重写一次，指令就变多了，没有直接调用快了。 so。。。。大概有3倍的性能差距。