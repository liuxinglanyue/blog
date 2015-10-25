---
layout: post
category: "java"
title:  "Netty boss线程 Accept"
tags: [java,netty,thread]
---

我们知道Netty可以这样创建boss、work的EventLoopGroup，如果nThreads有多个呢，如何处理分配accept的呢？

<pre class="prettyPrint">
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup();
</pre>

借用下参考资料的图片：

![netty_accept](/img/java/netty-accept.png)

<pre class="prettyPrint">
private final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
    @Override
    public EventExecutor next() {
        return children[childIndex.getAndIncrement() & children.length - 1];
    }
}

private final class GenericEventExecutorChooser implements EventExecutorChooser {
    @Override
    public EventExecutor next() {
        return children[Math.abs(childIndex.getAndIncrement() % children.length)];
    }
}
</pre>

这里并不是随机获取线程


参考资料：[Netty系列之Netty线程模型](http://www.infoq.com/cn/articles/netty-threading-model)


\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-如有错误请指正\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-