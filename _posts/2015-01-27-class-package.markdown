---
layout: post
category: "java"
title:  "从class文件中获取包名"
tags: [java,class,package,classFile]
---

有个需求，从Java的class文件中获取包名。我知道javap可以获取，但是想通过java的方式来获取。

不知道javap如何获取的，这里贴个链接：http://lydawen.iteye.com/blog/1874276 

用纯Java的方式走了点弯路，这里也顺便提下：

首先，想到的是将class文件转为byte[]，然后encodeHex，再解析。



