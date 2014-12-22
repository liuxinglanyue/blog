---
layout: post
category: "java"
title:  "StampedLock学习记录"
tags: [java,lock,stampedLock]
---

Java并发编程实战中谈到过ABA问题，这里摘录下：

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

ABA问题是一种异常现象：如果在算法中的节点可以被循环使用，那么在使用“比较并交换”指令时就可能出现这种问题
（主要在没有垃圾回收机制的环境中）。在CAS操作中将判断“V的值是否仍然为A？”，并且如果是的话就继续执行更新操作。
在大多数情况下，包括这本书给出的示例，这种判断是完全足够的。然而，有时候还需要知道“自从上次看到V的值为A以来，
这个值是否发生了变化？”在某些算法中，如果V的值首先由A变成B，再由B变成A，那么仍然被认为是发生了变化，
并需要重新执行算法中的某些步骤。

解决办法就是“版本号”

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

我们来看一下StampedLock的官方例子，

<pre class="prettyPrint">
import java.util.concurrent.locks.StampedLock;
//
public class Point {
//
	private double x, y;
	private final StampedLock sl = new StampedLock();
//
	void move(double deltaX, double deltaY) { // an exclusively locked method
		long stamp = sl.writeLock();
		try {
			x += deltaX;
			y += deltaY;
		} finally {
			sl.unlockWrite(stamp);
		}
	}
//
	double distanceFromOrigin() { // A read-only method
		long stamp = sl.tryOptimisticRead();
		double currentX = x, currentY = y;
		if (!sl.validate(stamp)) {
			stamp = sl.readLock();
			try {
				currentX = x;
				currentY = y;
			} finally {
				sl.unlockRead(stamp);
			}
		}
		return Math.sqrt(currentX * currentX + currentY * currentY);
	}
//
	void moveIfAtOrigin(double newX, double newY) { // upgrade
		// Could instead start with optimistic, not read mode
		long stamp = sl.readLock();
		try {
			while (x == 0.0 && y == 0.0) {
				long ws = sl.tryConvertToWriteLock(stamp); //升级为写锁
				if (ws != 0L) { //upgrade成功
					stamp = ws; 
					x = newX;
					y = newY;
					break;
				} else {
					sl.unlockRead(stamp);
					stamp = sl.writeLock();
				}
			}
		} finally {
			sl.unlock(stamp);
		}
	}
	//
	public static void main(String[] args) {
		Point point = new Point();
		point.moveIfAtOrigin(10, 20);
		point.distanceFromOrigin();
	}
}
</pre>

move方法中writeLock返回long，这个stamp就是版本号。

distanceFromOrigin方法中tryOptimisticRead是乐观读，源码中可知，实际没有lock。
然后validate会校验stamp是否变化。

其中U.loadFence();是关键，作用是 强制读取操作和验证操作在一些情况下的内存排序问题。

<pre class="prettyPrint">
public long tryOptimisticRead() {
	long s;
	return (((s = state) & WBIT) == 0L) ? (s & SBITS) : 0L;
}
//
public boolean validate(long stamp) {
        U.loadFence();
        return (stamp & SBITS) == (state & SBITS);
}
</pre>

这里有篇文章，在官方文档的基础上加入自己的理解，特分享下。
http://lianming.info/java8/2014/02/10/stamped-lock/

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

到这里了还没有结束，JaonHui有篇文章《Bug:StampedLock的中断问题导致CPU爆满》
http://ifeve.com/stampedlock-bug-cpu/

<pre class="prettyPrint">
import java.util.concurrent.locks.LockSupport;
import java.util.concurrent.locks.StampedLock;
//
public class TestStampedLock {
	public static void main(String[] args) throws InterruptedException {
		final StampedLock lock = new StampedLock();
		new Thread() {
			public void run() {
				long readLong = lock.writeLock();
				LockSupport.parkNanos(6100000000L);
				lock.unlockWrite(readLong);
			}
		}.start();
		Thread.sleep(100);
		for (int i = 0; i < 3; ++i)
			new Thread(new OccupiedCPUReadThread(lock)).start();
		System.out.println("end");
	}
//
	private static class OccupiedCPUReadThread implements Runnable {
		private StampedLock lock;
//
		public OccupiedCPUReadThread(StampedLock lock) {
			this.lock = lock;
		}
//
		public void run() {
			Thread.currentThread().interrupt();
			long lockr = lock.readLock();
			System.out.println(Thread.currentThread().getName()
					+ " get read lock");
			lock.unlockRead(lockr);
		}
	}
}
</pre>

这个例子说的是对中断的不作为。作者JaonHui给出了修复，见这里：
https://github.com/zuai/Hui/blob/master/StampedLock.java
