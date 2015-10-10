---
layout: post
category: "java"
title:  "查看Java代码对应的汇编指令利器，hsdis"
tags: [jvm,java,hsdis,c]
---

上一篇文章中提到了汇编指令[unsafe中getAndAddInt性能疑问](http://javagoo.com/java/unsafe_getAndAddInt.html) ，正是由于对汇编不甚了解，导致走了很多误区。

这篇文章主要说下如何使用[hsdis](https://kenai.com/projects/base-hsdis)？

首先给出hsdis的下载地址[官方地址](https://kenai.com/projects/base-hsdis/downloads)，但是官方没有提供Windows的，[iteye地址](http://hllvm.group.iteye.com/group/share)

这篇文章hsdis的使用方式，在OSX、Ubuntu14.04、Centos5.6测试通过，JDK7、JDK8都可。

假设本地hsdis存放目录$HOME/hsdis

下载的hsdis文件，需要根据系统修改为对应的名称，不知道如何修改的，可以根据错误提示修改。

设置环境变量

> export LD_LIBRARY_PATH=$HOME/hsdis
 
<pre class="prettyPrint">
public class Bar {
    int a = 1;
    static int b = 2;
    public int sum(int c) {
        return a + b + c;
    }
    public static void main(String[] args) {
        new Bar().sum(3);
    }
}
</pre>

> javac Bar.java

> java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly -Xcomp -XX:CompileCommand=dontinline,Bar.sum -XX:CompileCommand=compileonly,Bar.sum Bar

1. 参数UnlockDiagnosticVMOptions从JDK7的某个版本开始必须加上。

2. 参数PrintAssembly是输出反汇编内容

3. 参数-Xcomp是让虚拟机以编译模式执行代码

4. 参数dontinline是不对Bar.sum方法做内联优化

5. 参数compileonly是仅仅编译Bar.sum方法

下面是汇编代码，这里顺便贴一下

<pre class="prettyPrint">
Java HotSpot(TM) 64-Bit Server VM warning: PrintAssembly is enabled; turning on DebugNonSafepoints to gain additional output
CompilerOracle: dontinline Bar.sum
CompilerOracle: compileonly Bar.sum
Loaded disassembler from hsdis-amd64.dylib
Decoding compiled method 0x000000010649b0d0:
Code:
[Disassembling for mach='i386:x86-64']
[Entry Point]
[Constants]
  # {method} 'sum' '(I)I' in 'Bar'
  # this:     rsi:rsi   = 'Bar'
  # parm0:    rdx       = int
  #           [sp+0x20]  (sp of caller)
  0x000000010649b200: mov    0x8(%rsi),%r10d
  0x000000010649b204: shl    $0x3,%r10
  0x000000010649b208: cmp    %r10,%rax
  0x000000010649b20b: jne    0x0000000106470960  ;   {runtime_call}
  0x000000010649b211: data32 xchg %ax,%ax
  0x000000010649b214: nopl   0x0(%rax,%rax,1)
  0x000000010649b21c: data32 data32 xchg %ax,%ax
[Verified Entry Point]
  0x000000010649b220: sub    $0x18,%rsp
  0x000000010649b227: mov    %rbp,0x10(%rsp)    ;*synchronization entry
                                                ; - Bar::sum@-1 (line 5)
  0x000000010649b22c: movabs $0x7aaadbf68,%r10  ;   {oop(a 'java/lang/Class' = 'Bar')}
  0x000000010649b236: mov    0x70(%r10),%r10d
  0x000000010649b23a: add    0xc(%rsi),%r10d
  0x000000010649b23e: mov    %edx,%eax
  0x000000010649b240: add    %r10d,%eax         ;*iadd
                                                ; - Bar::sum@9 (line 5)
  0x000000010649b243: add    $0x10,%rsp
  0x000000010649b247: pop    %rbp
  0x000000010649b248: test   %eax,-0x8ac24e(%rip)        # 0x0000000105bef000
                                                ;   {poll_return}
  0x000000010649b24e: retq
  0x000000010649b24f: hlt
  0x000000010649b250: hlt
  0x000000010649b251: hlt
  0x000000010649b252: hlt
  0x000000010649b253: hlt
  0x000000010649b254: hlt
  0x000000010649b255: hlt
  0x000000010649b256: hlt
  0x000000010649b257: hlt
  0x000000010649b258: hlt
  0x000000010649b259: hlt
  0x000000010649b25a: hlt
  0x000000010649b25b: hlt
  0x000000010649b25c: hlt
  0x000000010649b25d: hlt
  0x000000010649b25e: hlt
  0x000000010649b25f: hlt
[Exception Handler]
[Stub Code]
  0x000000010649b260: jmpq   0x00000001064980a0  ;   {no_reloc}
[Deopt Handler Code]
  0x000000010649b265: callq  0x000000010649b26a
  0x000000010649b26a: subq   $0x5,(%rsp)
  0x000000010649b26f: jmpq   0x0000000106471b00  ;   {runtime_call}
  0x000000010649b274: hlt
  0x000000010649b275: hlt
  0x000000010649b276: hlt
  0x000000010649b277: hlt
</pre>


想了解具体的指令是啥含义，可以参考这篇文章[JVM执行篇：使用HSDIS插件分析JVM代码执行细节](http://www.infoq.com/cn/articles/zzm-java-hsdis-jvm)



