---
layout: post
category: "java"
title:  "查看Java代码对应的汇编指令又一利器，JITWatch"
tags: [jvm,java,hsdis,c,jitwatch,jit]
---

接着上一篇文章[查看Java代码对应的汇编指令利器，hsdis](http://javagoo.tk/java/hsdis_code.html)。JITWatch提供了更好的显示方式，还有各种图表，称得上又一利器。

github地址：[JITWatch](https://github.com/AdoptOpenJDK/jitwatch)

<pre class="prettyPrint">
git clone git@github.com:AdoptOpenJDK/jitwatch.git
cd jitwatch
mvn clean install -DskipTests=true
./launchUI.sh
</pre>

我们通过一个简单的例子来看下如何使用（例子稍微复杂些，为了了解JDK8对AtomicInteger.getAndIncrement()方法做的优化），基于JDK8(jdk7的getAndIncrement()方法实现不同)

首先给出java代码，AtomicInteger_jdk8.java

<pre class="prettyPrint">
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicInteger_jdk8 {

private final static int TEST_SIZE = 100000000;
	
	private final static int THREAD_COUNT = 10;
	
	private CountDownLatch cdl = new CountDownLatch(THREAD_COUNT + 1);
	
	private AtomicInteger ai = new AtomicInteger(0);
	
	private long startTime;

	public void init() {
		startTime = System.nanoTime();
	}
	
	public class MyTask implements Runnable {
		@Override
		public void run() {
			while (true) {
				if(ai.getAndIncrement() == TEST_SIZE) {
					System.out.println(System.nanoTime() - startTime);
					cdl.countDown();
					System.exit(0);
				}
			}
		}
	}
	
	public static void main(String[] args) {
		AtomicInteger_jdk8 at = new AtomicInteger_jdk8();
		
		at.init();
		for (int n = 0; n < THREAD_COUNT; n++)
			new Thread(at.new MyTask()).start();
		System.out.println("start");
		at.cdl.countDown();
	}
}
</pre>

编译执行，并输出日志（提示：需要hsdis）

<pre class="prettyPrint">
javac AtomicInteger_jdk8.java

java -server -XX:+UnlockDiagnosticVMOptions -XX:+TraceClassLoading  -XX:+PrintAssembly -XX:+LogCompilation -XX:LogFile=jit.log  AtomicInteger_jdk8
</pre>

如图所示：

![jitwatch](/img/jitwatch.png)

1. 点击Open Log选择jit.log文件
2. 点击Start
3. 如图，右击run()方法，点击TriView


![jitwatch-2](/img/jitwatch-2.png)

AtomicInteger.getAndIncrement()方法对应的汇编指令callq。

通过JITWatch发现，getAndAddInt()已经被编译为特殊的机器指令xadd（这就是为啥jdk8比jdk7快的原因，读者可以自己看下jdk7是啥）

![jitwatch-3](/img/jitwatch-3.png)

