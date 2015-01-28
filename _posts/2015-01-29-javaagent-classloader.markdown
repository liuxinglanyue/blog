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




