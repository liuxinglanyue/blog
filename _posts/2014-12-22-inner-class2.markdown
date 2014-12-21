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
