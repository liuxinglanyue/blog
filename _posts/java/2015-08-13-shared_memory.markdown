---
layout: post
category: "java"
title:  "关于Java共享内存"
tags: [java,shared,jvm,c,unsafe]
---

今天在想多进程、甚至多机共享内存的问题。我们知道数据库可以搭建共享存储来实现XXX。进程间可以通过Unix域来通信【性能和127.0.0.1差别不大，前提Linux内核，Windows不清楚】。Java可以通过MappedByteBuffer实现数据共享。

但是这篇文章，我们讨论的是如何通过Unsafe实现数据共享数据【暂没有找到好办法】。


贴下MappedByteBuffer数据共享代码

<pre class="prettyPrint">
public class MemMap1 {
    public static void main(String[] args) throws IOException, InterruptedException {
        File file = new File("/tmp/map.txt");
        if(!file.exists()) {
            file.createNewFile();
        }

        RandomAccessFile memoryMappedFile = new RandomAccessFile(file, "rw");
        MappedByteBuffer buffer = memoryMappedFile.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, 100);

        buffer.putLong(123);
        buffer.putLong(456);
        buffer.putLong(789);

        Thread.sleep(20000);

        memoryMappedFile.close();
    }
}
</pre>

<pre class="prettyPrint">
public class MemMap2 {

    public static void main(String[] args) throws IOException, InterruptedException {
        File file = new File("/tmp/map.txt");
        if(!file.exists()) {
            return;
        }

        RandomAccessFile memoryMappedFile = new RandomAccessFile(file, "rw");
        MappedByteBuffer buffer = memoryMappedFile.getChannel().map(FileChannel.MapMode.READ_WRITE, 0, 100);

        System.out.println(buffer.getLong());
        System.out.println(buffer.getLong());
        System.out.println(buffer.getLong());

        memoryMappedFile.close();
    }
}
</pre>

MappedByteBuffer中，getXXX、putXXX也是通过Unsafe来实现的。

MemMap1调用unsafe.putLong，MemMap2调用unsafe.getLong，但是两者操作的内存地址不同【这是最关键的】，就是说，不同的进程映射同一个文件，但是内存空间却是分开的。


\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-如有错误请指正\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-