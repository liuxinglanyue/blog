---
layout: post
category: "java"
title:  "内部类类名和父类类名相同的问题"
tags: [java,inner,class]
---

这个问题的出现，首先要从jar混淆工具说起。将经过混淆的jar解压，你可能会看到两个文件 a.class  a$a.class。 写过内部类的童鞋应该知道，这种情况应该是编译不了的呢。

<pre class="prettyPrint">
public class B {
    public static void main(String[] args) {
    }

    public class B {
        public String d;
    }
}
</pre>

javac B.java

报错：

<pre class="prettyPrint">
B.java:5: 错误: 已在程序包 未命名程序包中定义了类 B
    public class B {
           ^
1 个错误
</pre>


但是为什么这个 可以执行呢，我猜是编译比较严格，执行是完全没有问题的。

于是，我就用ASM写class文件。

<pre class="prettyPrint">
import org.objectweb.asm.*;

import java.io.FileOutputStream;
import java.io.IOException;

public class OuterClass extends ClassLoader implements Opcodes {
    public static void main(final String args[]) throws Exception {
        ClassWriter cw = new ClassWriter(0);
        cw.visit(V1_1, ACC_PUBLIC, "A", null, "java/lang/Object", null);
        cw.visitInnerClass("A$A", "A", "A", ACC_PUBLIC);

        //生成默认的构造方法
        MethodVisitor mw = cw.visitMethod(ACC_PUBLIC,
                "<init>",
                "()V",
                null,
                null);

        //生成构造方法的字节码指令
        mw.visitVarInsn(ALOAD, 0);
        mw.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V");
        mw.visitInsn(RETURN);
        mw.visitMaxs(1, 1);
        mw.visitEnd();

        //生成main方法
        mw = cw.visitMethod(ACC_PUBLIC + ACC_STATIC,
                "main",
                "([Ljava/lang/String;)V",
                null,
                null);

        //生成main方法中的字节码指令
        mw.visitFieldInsn(GETSTATIC,
                "java/lang/System",
                "out",
                "Ljava/io/PrintStream;");

        mw.visitLdcInsn("Hello world!");
        mw.visitMethodInsn(INVOKEVIRTUAL,
                "java/io/PrintStream",
                "println",
                "(Ljava/lang/String;)V");
        //
        mw.visitTypeInsn(Opcodes.NEW, "A");
        mw.visitInsn(Opcodes.DUP);
        mw.visitMethodInsn(Opcodes.INVOKESPECIAL, "A", "<init>", "()V");
        mw.visitVarInsn(ASTORE, 1);

        mw.visitTypeInsn(Opcodes.NEW, "A$A");
        mw.visitInsn(Opcodes.DUP);
        mw.visitVarInsn(ALOAD, 1);
        mw.visitInsn(Opcodes.DUP);
        mw.visitMethodInsn(INVOKEVIRTUAL, "java/lang/Object", "getClass", "()Ljava/lang/Class;");
        mw.visitInsn(POP);
        mw.visitMethodInsn(Opcodes.INVOKESPECIAL, "A$A", "<init>", "(LA;)V");
        mw.visitVarInsn(ASTORE, 2);
        mw.visitVarInsn(ALOAD, 2);
        mw.visitLdcInsn("e");
        mw.visitFieldInsn(Opcodes.PUTFIELD, "A$A", "e", Type.getDescriptor(String.class));

        mw.visitFieldInsn(GETSTATIC,
                "java/lang/System",
                "out",
                "Ljava/io/PrintStream;");

        mw.visitVarInsn(ALOAD, 2);
        mw.visitFieldInsn(Opcodes.GETFIELD, "A$A", "e", Type.getDescriptor(String.class));
        mw.visitMethodInsn(INVOKEVIRTUAL,
                "java/io/PrintStream",
                "println",
                "(Ljava/lang/String;)V");

        mw.visitInsn(RETURN);
        mw.visitMaxs(4, 3);

        //字节码生成完成
        mw.visitEnd();
        cw.visitEnd();

        // 获取生成的class文件对应的二进制流
        byte[] code = cw.toByteArray();

        //将二进制流写到本地磁盘上
        FileOutputStream fos = new FileOutputStream("A.class");

        fos.write(code);
        fos.flush();
        fos.close();

        //asm 写 内部类
        byte[] outCode = out();

        //直接将二进制流加载到内存中
        OuterClass loader = new OuterClass();
        Class<?> exampleClass = loader.defineClass("A", code, 0, code.length);
        loader.defineClass("A$A", outCode, 0, outCode.length);

        //通过反射调用main方法
        exampleClass.getMethods()[0].invoke(null, new Object[] { null });
    }

    public static byte[] out() throws IOException {
        //定义一个叫做Example的类
        ClassWriter cw = new ClassWriter(0);
        cw.visit(V1_1, ACC_PUBLIC, "A$A", null, "java/lang/Object", null);

        FieldVisitor fv = cw.visitField(ACC_PUBLIC, "e", Type.getDescriptor(String.class), null, null);
        fv.visitEnd();
        FieldVisitor fv2 = cw.visitField(ACC_FINAL, "this$0", "LA;", null, null);
        fv2.visitEnd();

        //生成默认的构造方法
        MethodVisitor mw = cw.visitMethod(ACC_PUBLIC,
                "<init>",
                "(LA;)V",
                null,
                null);

        //生成构造方法的字节码指令
        mw.visitVarInsn(ALOAD, 0);
        mw.visitVarInsn(ALOAD, 1);
        mw.visitFieldInsn(Opcodes.PUTFIELD, "A$A", "this$0", "LA;");
        mw.visitVarInsn(ALOAD, 0);
        mw.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V");
        mw.visitInsn(RETURN);
        mw.visitMaxs(2, 2);
        mw.visitEnd();
        cw.visitEnd();

        // 获取生成的class文件对应的二进制流
        byte[] code = cw.toByteArray();

        //将二进制流写到本地磁盘上
        FileOutputStream fos = new FileOutputStream("A$A.class");
        fos.write(code);
        fos.flush();
        fos.close();

        return code;
    }
}
</pre>

生成的class文件有  A.class  A$A.class

执行结果：

<pre class="prettyPrint">
Hello world!
e
</pre>


gist:[https://gist.github.com/liuxinglanyue/430ba68a4123e3e0eddfe228bd2187cf](https://gist.github.com/liuxinglanyue/430ba68a4123e3e0eddfe228bd2187cf)