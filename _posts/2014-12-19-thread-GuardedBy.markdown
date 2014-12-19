---
layout: post
category: "java"
title:  "从GuardedBy说起的jsr305"
tags: [thread,threadSafe,guardedBy,jsr305]
---

这件事儿得从commons-pool说起，在看pool源码的过程中看到LinkedBlockingDeque。http://javagoo.tk/java/commons-pool.html

其中有三个成员变量被标注为GuardedBy("lock")， 好奇找了下，看是哪个jar中的。
<pre class="prettyPrint">
  /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    private transient Node<E> first; // \@GuardedBy("lock")
//
    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    private transient Node<E> last; // \@GuardedBy("lock")
//
    /** Number of items in the deque */
    private transient int count; // \@GuardedBy("lock")
</pre>

jsr305-2.0.1.jar 中除了GuardedBy还有ThreadSafe、NotThreadSafe、Nonnull、CheckForNull。。。。。
看过《Java并发编程实战》对ThreadSafe、NotThreadSafe肯定不陌生，书中一堆堆，还有那个可爱的图标，哈哈。

这本书中附录A对GuardedBy有不错的讲解。摘录如下

