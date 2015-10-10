---
layout: post
category: "java"
title:  "从GuardedBy说起的jsr305"
tags: [thread,threadSafe,guardedBy,jsr305]
---

这件事儿得从commons-pool说起，在看pool源码的过程中看到LinkedBlockingDeque。http://javagoo.com/java/commons-pool.html

其中有三个成员变量被标注为GuardedBy("lock")， 好奇找了下，看是哪个jar中的。

注意：文中的At省略了
--------------------
<pre class="prettyPrint">
  /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    private transient Node<E> first; // GuardedBy("lock")
//
    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    private transient Node<E> last; // GuardedBy("lock")
//
    /** Number of items in the deque */
    private transient int count; // GuardedBy("lock")
</pre>

jsr305-2.0.1.jar 中除了GuardedBy还有ThreadSafe、NotThreadSafe、Nonnull、CheckForNull。。。。。

看过《Java并发编程实战》对ThreadSafe、NotThreadSafe肯定不陌生，书中一堆堆，还有那个可爱的图标，哈哈。

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

这本书中附录A对GuardedBy有不错的讲解。摘录如下

在使用加锁的类中，应该说明哪些状态变量由哪些锁保护的，以及哪些锁被用于保护这些变量。

一种造成不安全性的常见原因是：某个线程安全的类一直通过加锁来保护其状态，但随后又对这个类进行了修改，

并添加了一些未通过锁来保护的新变量，或者没有使用正确加锁来保护现有状态变量的新方法。

通过说明哪些变量由哪些锁来保护，有助于避免这些疏忽。

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

这里有篇文章是关于JSR305的，http://www.infoq.com/cn/news/2008/07/jsr-305-update

