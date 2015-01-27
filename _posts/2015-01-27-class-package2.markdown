---
layout: post
category: "java"
title:  "从class文件中获取包名(二)"
tags: [java,class,package,classFile]
---

上文中，采用的是ClassFile来获取常量池信息，从而获得包名。

缺点是，依赖tools.jar包。

本文代码是将ClassFile解析代码抽取出来，进行简化，提升性能。

还是那句话，

如果有更简单的方法，望告知。
