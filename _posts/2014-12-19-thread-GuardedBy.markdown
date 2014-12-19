---
layout: post
category: "java"
title:  "从GuardedBy说起的jsr305"
tags: [thread,threadSafe,guardedBy,jsr305]
---

这件事儿得从commons-pool说起，在看pool源码的过程中看到LinkedBlockingDeque。http://javagoo.tk/java/commons-pool.html

其中有三个成员变量被标注为\@GuardedBy("lock")， 好奇找了下，看是哪个jar中的。
<pre class="prettyPrint">
  /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    private transient Node<E> first; // @GuardedBy("lock")
//
    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    private transient Node<E> last; // @GuardedBy("lock")
//
    /** Number of items in the deque */
    private transient int count; // @GuardedBy("lock")
</pre>

