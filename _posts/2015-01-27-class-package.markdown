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

说明下，这种思路是没有问题的(实现复杂)，但是下面PackageFromBytes的做法不正确(某些情况下可行，具体原因读者自己分析)。

<pre class="prettyPrint">
package org.hot2hot.utils;

import org.apache.commons.lang.StringUtils;

@Deprecated
public class PackageFromBytes {
	
	private static final char[] DIGITS_UPPER = {'0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'A', 'B', 'C', 'D', 'E', 'F'};

	public static String getPackage(byte[] clazz) {
		StringBuilder pack = new StringBuilder();
		//300 是根据包名的一般长度设置的。
		char[] cs = encodeHex(clazz, 300);
		
		int i = 32;
		while(true) {
			String hexStr = new String(cs, i, 2);
			int c = Integer.parseInt(hexStr, 16);
			pack.append((char)c);
			
			i += 2;
			if(!(c >= 97 && c <= 122 || c >= 65 && c <= 90 || c == 47 || c>= 48 && c <=57 || c == 36)) {
				break;
			}
		}
		return StringUtils.replace(pack.toString(), "/", ".");
	}
	
	private static char[] encodeHex(byte[] data, int think) {
		int l = data.length;
		l = think < l ? think : l;
		char[] out = new char[l << 1];
		for (int i = 0, j = 0; i < l; i++) {
			out[j++] = DIGITS_UPPER[(0xF0 & data[i]) >>> 4];
			out[j++] = DIGITS_UPPER[0x0F & data[i]];
		}
		return out;
	}
}

</pre>

转变思路，com.sun.tools提供了ClassFile类，通过它可以得到常量池信息。

这里贴张图复习下：

![hello](/img/class-package.png) 

于是就有了下面这段代码：

<pre class="prettyPrint">
package org.hot2hot.utils;

import java.io.File;
import java.io.IOException;

import org.apache.commons.lang.StringUtils;

import com.sun.tools.classfile.ClassFile;
import com.sun.tools.classfile.ConstantPool.CONSTANT_Class_info;
import com.sun.tools.classfile.ConstantPool.CONSTANT_Utf8_info;
import com.sun.tools.classfile.ConstantPool.CPInfo;
import com.sun.tools.classfile.ConstantPoolException;

public class PackageFromClassFile {

	public static String getPackage(File paramFile) {
		try {
			ClassFile classFile = ClassFile.read(paramFile);
			CPInfo classInfo = classFile.constant_pool.get(classFile.this_class);
			if(classInfo instanceof CONSTANT_Class_info) {
				CPInfo cp = classFile.constant_pool.get(((CONSTANT_Class_info) classInfo).name_index);
				if(cp instanceof CONSTANT_Utf8_info) {
					return StringUtils.replace(((CONSTANT_Utf8_info)cp).value, "/", ".");
				}
			}
			
		} catch (IOException e) {
			e.printStackTrace();
		} catch (ConstantPoolException e) {
			e.printStackTrace();
		}
		
		return null;
	}
	
	public static void main(String[] args) {
		File file = new File("/Users/jiaojianfeng/Documents/empleyment/clazz/A.class");
		System.out.println(getPackage(file));
	}
}

</pre>

折腾。。。。。

如果有更简单的方法，望告知。