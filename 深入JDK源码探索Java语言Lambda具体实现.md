# 深入JDK源码探索Java语言Lambda具体实现

当我刚刚开始学习Java的Lambda的时候，我曾经完全误解了它的实现方式，并且前段时间发现居然有些朋友和当时的我一样。

比如下面这个例子：

``` java
class Foobar {
    Runnable foobar() {
        return () -> {};
    }
}
```

在最初的时候，我猜想它也许会像这样实现：

``` java
class Foobar {
    Runnable foobar() {
        return new Runnable() {
            @Override
            public void run() {
                return;
            }
        };
    }
}
```

实际上，并不是。

当然了，如果真是这样做了也算是一种实现方式，但是Java这里做出了更加巧妙的设计。

这个函数在Java8中经过编译后的字节码是这样的：

``` java
  java.lang.Runnable foobar();
    descriptor: ()Ljava/lang/Runnable;
    flags:
    Code:
      stack=1, locals=1, args_size=1
         0: invokedynamic #2,  0              // InvokeDynamic #0:run:()Ljava/lang/Runnable;
         5: areturn
      LineNumberTable:
        line 3: 0
```

乍一看似乎莫名其妙，`run`方法其实也实现在了这个类里：

``` java
  private static void lambda$foobar$0();
    descriptor: ()V
    flags: ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=0, locals=0, args_size=0
         0: return
      LineNumberTable:
        line 3: 0
```

它们通过`InvokeDynamic`这个字节码指令以及一个BootstrapMethod才在两者之间建立起联系：

``` java
  0: #16 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #17 ()V
      #18 invokestatic Foobar.lambda$foobar$0:()V
      #17 ()V
```

看到这里的朋友，大概应该都是用过`MethodHandle`的吧，不然的话上面这段可能不太好理解。

用Java代码大致模拟一下，运行时的逻辑大概像是这样：

``` java
class Foobar {
    @SuppressWarnings("unchecked")
    Runnable foobar() throws Throwable {
        return (Runnable) LambdaMetafactory.metafactory(
            MethodHandles.lookup(),
            "run",
            MethodType.methodType(Runnable.class),
            MethodType.methodType(void.class),
            MethodHandles.lookup().findStatic(Foobar.class, "lambda$foobar$0", MethodType.methodType(void.class)),
            MethodType.methodType(void.class)
        ).dynamicInvoker().invoke();
    }
    private static void lambda$foobar$0() {
        System.out.println("Hello");
        return;
    }
}
```

事实上，JVM的`InvokeDynamic`比起Java的Lambda表达式和方法引用表达式有着更为久远的历史，详见[JSR-292](https://www.jcp.org/en/jsr/detail?id=292)。

`InvokeDynamic`这一条指令，首先会通过一个BootstrapMethod，获取到一个`CallSite`实例，BootstrapMethod至少需要`Lookup`、`String`、`MethodType`三个参数，以及任意多个静态参数，相关细节见虚拟机规范[invokedynamic](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.invokedynamic)。

在`LambdaMetafactory`中除了`metafactory`之外还有一个`altMetafactory`用于更加复杂的Lambda，本文暂不讨论。

此时已经可以了解到`metafactory`所需的六个参数分别代表什么，[这里](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/d103481ecd91/src/share/classes/java/lang/invoke/LambdaMetafactory.java#l291)是它在Java8中的源码。

这个方法很简洁，只有短短几行：

``` java
    public static CallSite metafactory(MethodHandles.Lookup caller,
                                       String invokedName,
                                       MethodType invokedType,
                                       MethodType samMethodType,
                                       MethodHandle implMethod,
                                       MethodType instantiatedMethodType)
            throws LambdaConversionException {
        AbstractValidatingLambdaMetafactory mf;
        mf = new InnerClassLambdaMetafactory(caller, invokedType,
                                             invokedName, samMethodType,
                                             implMethod, instantiatedMethodType,
                                             false, EMPTY_CLASS_ARRAY, EMPTY_MT_ARRAY);
        mf.validateMetafactoryArgs();
        return mf.buildCallSite();
    }
```

`validateMetafactoryArgs`是`InnerClassLambdaMetafactory`从它的超类`AbstractValidatingLambdaMetafactory`中继承而来的一个方法，其作用只是进行一些校验操作，如果发现错误就会抛出`LambdaConversionException`。它的代码在[这里](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/d103481ecd91/src/share/classes/java/lang/invoke/AbstractValidatingLambdaMetafactory.java#l173)。

[这里](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/d103481ecd91/src/share/classes/java/lang/invoke/InnerClassLambdaMetafactory.java#l193)是`buildCallSite`的源码，它大致可以分为三个部分：

``` java
    @Override
    CallSite buildCallSite() throws LambdaConversionException {
        final Class<?> innerClass = spinInnerClass();
        if (invokedType.parameterCount() == 0 && !disableEagerInitialization) {
            // ...
            final Constructor<?>[] ctrs = /* ... */;
            Object inst = ctrs[0].newInstance();
            return new ConstantCallSite(MethodHandles.constant(samBase, inst));
            // ...
        } else {
            // ...
            if (!disableEagerInitialization) {
                UNSAFE.ensureClassInitialized(innerClass);
            }
            return new ConstantCallSite(MethodHandles.Lookup.IMPL_LOOKUP.findStatic(innerClass, NAME_FACTORY, invokedType));
            // ...
        }
    }
```

`disableEagerInitialization`与`jdk.internal.lambda.disableEagerInitialization`选项有关。

首先会在`spinInnerClass`中定义一个源码中不存在的类，在[这里](http://hg.openjdk.java.net/jdk8u/jdk8u/jdk/file/d103481ecd91/src/share/classes/java/lang/invoke/InnerClassLambdaMetafactory.java#l249)用到了`asm`框架。

比方说这个例子：

``` java
class Foobar {
    Runnable foo() {
        return () -> {};
    }
    Callable<?> bar() {
        return () -> this;
    }
}
class Main {
    public static void main(String[] args) {
        new Foobar().foo();
        new Foobar().bar();
    }
}
```

可以在运行的时候加上`-Djdk.internal.lambda.dumpProxyClasses`这个选项。

它在运行时定义的两个类大致是这样的：

``` java
final class Foobar$$Lambda$1 implements java.lang.Runnable
  minor version: 0
  major version: 52
  flags: ACC_FINAL, ACC_SUPER, ACC_SYNTHETIC
Constant pool:
  // ...
{
  private Foobar$$Lambda$1();
    descriptor: ()V
    flags: ACC_PRIVATE
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #10                 // Method java/lang/Object."<init>":()V
         4: return

  public void run();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=0, locals=1, args_size=1
         0: invokestatic  #17                 // Method Foobar.lambda$foo$0:()V
         3: return
    RuntimeVisibleAnnotations:
      0: #12()
}
```

``` java
final class Foobar$$Lambda$2 implements java.util.concurrent.Callable
  minor version: 0
  major version: 52
  flags: ACC_FINAL, ACC_SUPER, ACC_SYNTHETIC
Constant pool:
  // ...
{
  private final Foobar arg$1;
    descriptor: LFoobar;
    flags: ACC_PRIVATE, ACC_FINAL

  private Foobar$$Lambda$2(Foobar);
    descriptor: (LFoobar;)V
    flags: ACC_PRIVATE
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: invokespecial #13                 // Method java/lang/Object."<init>":()V
         4: aload_0
         5: aload_1
         6: putfield      #15                 // Field arg$1:LFoobar;
         9: return

  private static java.util.concurrent.Callable get$Lambda(Foobar);
    descriptor: (LFoobar;)Ljava/util/concurrent/Callable;
    flags: ACC_PRIVATE, ACC_STATIC
    Code:
      stack=3, locals=1, args_size=1
         0: new           #2                  // class Foobar$$Lambda$2
         3: dup
         4: aload_0
         5: invokespecial #19                 // Method "<init>":(LFoobar;)V
         8: areturn

  public java.lang.Object call();
    descriptor: ()Ljava/lang/Object;
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #15                 // Field arg$1:LFoobar;
         4: invokespecial #27                 // Method Foobar.lambda$bar$1:()Ljava/lang/Object;
         7: areturn
    RuntimeVisibleAnnotations:
      0: #22()
}
```

发现没有，这个`Runnable`调用得是`Foobar`的静态方法，并且它里面也并不持有外部引用，但是如果使用匿名内部类的实现方式，就有可能造成毫无意义的内存泄漏。

现在咱们再回过头来看看`buildCallSite`的实现，它判断了`invokedType.parameterCount()`是否等于零，如果为零说明不会捕获任何外部变量，那么这样的话，也就没有必要每次调用都为此而创建一个新的对象，但如果不为零的话，则会通过`get$Lambda`获取这个函数式接口的实例。

做一个简单的实验：

``` java
class Main {
    public static void main(String[] args) {
        assert(new Foobar().foo() == new Foobar().foo());
        assert(new Foobar().bar() == new Foobar().bar());
    }
}
```

开启`-ea`选项后，以上程序将在第二次断言时崩溃，因此对于无捕获的Lambda表达式来说，并不会重复创建对象，而用匿名内部类的方式则需要手动实现复用。

经过这一套流程，我们终于获得了`CallSite`的实例，它将与这一个位置绑定，重复调用则可以复用这同一个`CallSite`。

现在可以发现，Java实际对Lambda和方法引用的实现，是经过一层一层的封装，从而优化了许多细节，但是还不止如此。

注意一下我上面说过的话，Java中`InvokeDynamic`的历史比Lambda更长，JVM是一种语言无关的运行环境，只要按照规范的要求，编译出合法的字节码，那么无论任何语言都可以在JVM平台上运行，包括Groovy这种动态语言，有了`InvokeDynamic`这条指令，可以让动态语言在JVM平台上发展得更好。

`InvokeDynamic`这条字节码指令它最大的意义是，它使得JVM新语言的实现者获得了更大的自由，每个语言的设计者可以用自己的方式创建`CallSite`，然后通过自己的编译器去调用自己所定义的BootstrapMethod，而在这一整个过程中都不需要JVM为其提供任何额外的支持。

那么最后，我来编写一段示例代码，作为这篇文章的收尾：

``` java
import jdk.internal.org.objectweb.asm.*;
import jdk.internal.org.objectweb.asm.tree.*;
import java.lang.instrument.*;
import java.lang.invoke.*;
import java.security.*;
class Main {
    public static void main(String[] args) {
        getLambda().run();
    }
    private static native Runnable getLambda();
}
class Premain extends ClassVisitor implements ClassFileTransformer {
    public static void premain(String arg, Instrumentation instrumentation) {
        instrumentation.addTransformer(new Premain());
    }
    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        if (!"Main".equals(className)) {
            return classfileBuffer;
        }
        ClassWriter cw = new ClassWriter(Opcodes.ASM5);
        super.cv = cw;
        new ClassReader(classfileBuffer).accept(this, ClassReader.EXPAND_FRAMES);
        return cw.toByteArray();
    }
    private Premain() {
        super(Opcodes.ASM5, null);
    }
    @Override
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
        if ("getLambda".equals(name)) {
            return null;
        }
        return super.visitMethod(access, name, desc, signature, exceptions);
    }
    @Override
    public void visitEnd() {
        MethodVisitor mv = super.visitMethod(Opcodes.ACC_PRIVATE | Opcodes.ACC_STATIC | Opcodes.ACC_SYNTHETIC, "getLambda", "()Ljava/lang/Runnable;", null, null);
        mv.visitCode();
        Label l0 = new Label();
        mv.visitLabel(l0);
        mv.visitInvokeDynamicInsn("run", "()Ljava/lang/Runnable;", new Handle(Opcodes.H_INVOKESTATIC, "Premain", "buildCallSite", "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;"));
        Label l1 = new Label();
        mv.visitLabel(l1);
        mv.visitInsn(Opcodes.ARETURN);
        Label l2 = new Label();
        mv.visitLabel(l2);
        mv.visitMaxs(1, 0);
        mv.visitEnd();
        super.visitEnd();
    }
    public static CallSite buildCallSite(MethodHandles.Lookup caller, String invokedName, MethodType invokedType) {
        return new ConstantCallSite(MethodHandles.constant(Runnable.class, (Runnable) () -> System.out.println("Hello, world!")));
    }
}
```

有兴趣的同学，可以通过灵活使用这一条`InvokeDynamic`指令，在JVM这个广阔的平台上，定制自己更喜欢的语言。
