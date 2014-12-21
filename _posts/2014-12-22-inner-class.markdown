---
layout: post
category: "java"
title:  "对Java内部类的一点点深入分析"
tags: [java,inner,InnerClasses]
---

Java内部类，我们平时用的不算多，但是想JDK源码等等类库用的还是挺多的，故这里简单分析下它的实现原理。

首先我们带着下面的疑问来看问题：

内部类如何访问该类的外部类所有的成员变量、方法，特别是私有变量

<pre class="prettyPrint">
package com.innerClasses;
//
public class InnerClass {
	public static void main(String[] args) {
	}
}
//
class School {
	//
	private int id;
	//
	public void cacl() {
		Student student = new Student();
	}
	//
	private class Student {
		public void set(String name) {
			System.out.println(id + name);
		}
		private String name;
	}
}
</pre>

上面是一段简单的包含内部类的代码
