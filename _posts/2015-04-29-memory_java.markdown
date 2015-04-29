---
layout: post
category: "java"
title:  "一点点想法，关于Spark执行引擎大幅优化"
tags: [java,jvm,memory,performance]
---

其中有三点性能优化的方法。

原文：[Project Tungsten: Bringing Spark Closer to Bare Metal](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html)

1. Memory Management and Binary Processing: leveraging application semantics to manage memory explicitly and eliminate the overhead of JVM object model and garbage collection

2. Cache-aware computation: algorithms and data structures to exploit memory hierarchy

3. Code generation: using code generation to exploit modern compilers and CPUs

贴一下第一点的图。

![binary-map](/img/binary-map.png)

想了下Spark Binary Map，通过unsafe的确可以访问对象的内存地址，如果java中保存的是内存地址，查找如何做呢？

等待。。。。。。