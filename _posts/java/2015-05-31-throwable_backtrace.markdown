---
layout: post
category: "java"
title:  "Java类Throwable的属性backtrace，BUG"
tags: [java,backtrace,throwable,bug]
---

通过[jol](http://openjdk.java.net/projects/code-tools/jol/)查看Java Object Layout时，发现一个奇怪现象。

<pre class="prettyPrint">
public static void main(String[] args) throws Exception {
    out.println(VMSupport.vmDetails());
    out.println(ClassLayout.parseClass(Throwable.class).toPrintable());
}
</pre>

运行结果如下：

![throwable-layout](/img/java/throwable-layout.png)

仔细点会发现，属性backtrace竟然不在此列

<pre class="prettyPrint">
/**
 * Native code saves some indication of the stack backtrace in this slot.
 */
private transient Object backtrace;
</pre>

按理说只有static的才不在此列。深入源码，没发现啥黑洞。

google，google，找到了问题的缘由。

[https://bugs.openjdk.java.net/browse/JDK-4496456](https://bugs.openjdk.java.net/browse/JDK-4496456)

[https://bugs.openjdk.java.net/browse/JDK-8033735](https://bugs.openjdk.java.net/browse/JDK-8033735)

[https://bugs.openjdk.java.net/browse/JDK-8033735](https://bugs.openjdk.java.net/browse/JDK-8033735)

将属性backtrace排除在外，是JDK有意为之的。

如果想解决，官方也提供了patch修复

<pre class="prettyPrint">
diff -r c84312468f5c src/share/vm/prims/jvm.cpp
--- a/src/share/vm/prims/jvm.cpp	Fri Jan 24 13:06:52 2014 +0100
+++ b/src/share/vm/prims/jvm.cpp	Wed Feb 05 17:43:53 2014 -0800
@@ -1790,9 +1790,6 @@ JVM_ENTRY(jobjectArray, JVM_GetClassDecl
   // Ensure class is linked
   k->link_class(CHECK_NULL);
 
-  // 4496456 We need to filter out java.lang.Throwable.backtrace
-  bool skip_backtrace = false;
-
   // Allocate result
   int num_fields;
 
@@ -1803,11 +1800,6 @@ JVM_ENTRY(jobjectArray, JVM_GetClassDecl
     }
   } else {
     num_fields = k->java_fields_count();
-
-    if (k() == SystemDictionary::Throwable_klass()) {
-      num_fields--;
-      skip_backtrace = true;
-    }
   }
 
   objArrayOop r = oopFactory::new_objArray(SystemDictionary::reflect_Field_klass(), num_fields, CHECK_NULL);
@@ -1816,12 +1808,6 @@ JVM_ENTRY(jobjectArray, JVM_GetClassDecl
   int out_idx = 0;
   fieldDescriptor fd;
   for (JavaFieldStream fs(k); !fs.done(); fs.next()) {
-    if (skip_backtrace) {
-      // 4496456 skip java.lang.Throwable.backtrace
-      int offset = fs.offset();
-      if (offset == java_lang_Throwable::get_backtrace_offset()) continue;
-    }
-
     if (!publicOnly || fs.access_flags().is_public()) {
       fd.reinitialize(k(), fs.index());
       oop field = Reflection::new_field(&fd, UseNewReflection, CHECK_NULL);
</pre>
