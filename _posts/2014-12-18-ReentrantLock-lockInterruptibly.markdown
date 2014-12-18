---
layout: "post"
category: "java"
title: "关于ReentrantLock中lockInterruptibly的分析"
tags: [lock,ReentrantLock,Interrupted]
---

看一些开源项目，发现使用lockInterruptibly的情况还挺多的，故研究下。
首先从下面一段代码开始

方法1，第二次lock是会一直等待，对interrupt不感冒。

方法2，lockInterruptibly会先判断线程是否中断，如果中断直接throw InterruptedException（方法3就是这种情况）

没有中断的情况下，执行tryAcquire，这里肯定false，因为一开始已经lock了，没有unlock。

通过调试知道，线程阻塞在LockSupport.park(this);中的unsafe.park(false, 0L);

接下来如果执行thread.interrupt();会执行native方法interrupt0();  so.........

方法4，和方法2只有最后一行不同lock.unlock();这会调用LockSupport.unpark(s.thread); so.........

<pre class="prettyPrint">
package com.lock;
//
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
//
import org.junit.Test;
//
public class TestLockInterruptibly {
	//1
	@Test
	public void lock() throws InterruptedException {
		final Lock lock = new ReentrantLock();
		lock.lock();
		Thread.sleep(1000);
		Thread  thread = new Thread(new Runnable() {
			//
			@Override
			public void run() {
				lock.lock();
			}
		});
		//
		thread.start();
		Thread.sleep(2000);
		thread.interrupt();
	}
	//2
	@Test
	public void lockInterrupt() throws InterruptedException {
		final Lock lock = new ReentrantLock();
		lock.lock();
		Thread.sleep(1000);
		Thread  thread = new Thread(new Runnable() {
			//
			@Override
			public void run() {
				try {
					lock.lockInterruptibly();
				} catch (InterruptedException e) {
					System.out.println("Thread name " + Thread.currentThread().getName() + " is interrupted!");
				}
			}
		});
		//
		thread.start();
		Thread.sleep(2000);
		thread.interrupt();
	}
	//3
	@Test
	public void lockInterrupt2() throws InterruptedException {
		final Lock lock = new ReentrantLock();
		lock.lock();
		Thread.sleep(1000);
		Thread  thread = new Thread(new Runnable() {
			//
			@Override
			public void run() {
				try {
					Thread.sleep(2000);
					lock.lockInterruptibly();
				} catch (InterruptedException e) {
					System.out.println("Thread name " + Thread.currentThread().getName() + " is interrupted!");
				}
			}
		});
		//
		thread.start();
		thread.interrupt();
	}
	//4
	@Test
	public void lockInterruptUnLock() throws InterruptedException {
		final Lock lock = new ReentrantLock();
		lock.lock();
		Thread.sleep(1000);
		Thread  thread = new Thread(new Runnable() {
			//
			@Override
			public void run() {
				try {
					lock.lockInterruptibly();
				} catch (InterruptedException e) {
					System.out.println("Thread name " + Thread.currentThread().getName() + " is interrupted!");
				}
			}
		});
		//
		thread.start();
		Thread.sleep(2000);
		lock.unlock();
	}
}
</pre>
