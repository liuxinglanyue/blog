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

问题来了，Student类是如何访问School中id的呢。
InnerClass.java编译后生成三个文件，Student类反编译后如图所示：

![hello](/img/inner-student.png) 

Student类的构造方法多了School paramSchool这个参数，然后JD-GUI反编译出来的又没有将其赋值给任何变量。
我猜测是作者过滤掉了，so 我们通过javap -verbose 看下。

Student类中多了这样一个属性final com.innerClasses.School this0; 其他就不需要我多说什么了吧，都懂了。

还有一个疑问就是private类型的属性是如何访问的呢？

答案就是通过在School类中生成static的access方法来获取所需的值。
