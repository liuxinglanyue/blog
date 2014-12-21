---
layout: post
category: "java"
title:  "对Java内部类的一点点深入分析（补充）"
tags: [inner,InnerClasses]
---

这篇是对上一篇文章的补充，有种初始化是匿名类初始化，我们这里就来比较下。

<pre class="prettyPrint">
import java.util.ArrayList;
import java.util.List;
//
public class InnerClass {
	//
	public void initList() {
		List<String> l = new ArrayList<String>();
		l.add("Hello");
		l.add("World!");
	}
	//
	public void initInnerList() {
		List<String> l = new ArrayList<String>() {
			{
				add("Hello");
				add("World!");
			}
		};
	}
}
</pre>

initInnerList方法中l的初始化是在{}中，接下来我们直接看反编译后的代码

![hello](/img/inner-list.png)

![hello](/img/inner-list-2.png)

initInnerList方法中初始化代码没有了，另外多了个class。
真没有了吗？实际上调用了匿名类的构造方法来初始化，呵呵。

InnerClass1构造方法不需要多介绍了吧，可以看上一篇博客。

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

这里有篇不错的文章，推荐下。
Efficiency of Java “Double Brace Initialization”?
http://stackoverflow.com/questions/924285/efficiency-of-java-double-brace-initialization
