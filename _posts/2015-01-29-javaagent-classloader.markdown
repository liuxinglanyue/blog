---
layout: post
category: "java"
title:  "加载javaagent的classloader问题"
tags: [java,class,javaagent,classloader]
---

之前在做项目hot2hot时，采用的是javaagent方式。

其中有一个需求是，获取需要替换类的Class，然后不知道具体是啥类，只知道类的全限定名（根据class文件获取http://javagoo.tk/java/class-package.html）

由于知道了类的全限定名，所以采用Class.forName来加载，但这时你会得到ClassNotFoundException。

这时想到，应该是ClassLoader在捣乱。于是，输出javaagent中类的加载器，猜猜看是啥？null

查看官方文档：http://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
The agent class will be loaded by the system class loader (see ClassLoader.getSystemClassLoader). This is the class loader which typically loads the class containing the application main method. The premain methods will be run under the same security and classloader rules as the application main method. There are no modeling restrictions on what the agent premain method may do. Anything application main can do, including creating threads, is legal from premain.
\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

Oh! My God ，系统类加载的。

知道Java双亲委派模型的都知道，应用中自己实现的类的classloader都是sun.misc.Launcher$AppClassLoader（特殊情况除外，比如Tomcat等）。

所以，想获取替换类的Class应该如何做呢？（参看前文：http://javagoo.tk/java/hot2hot.html）

