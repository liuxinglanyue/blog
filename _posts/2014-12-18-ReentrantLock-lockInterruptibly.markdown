---
layout: "post"
category: "java"
title: "关于ReentrantLock中lockInterruptibly的分析"
tags: [lock,ReentrantLock,Interrupted]
---


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
