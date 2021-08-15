# 探索Java中隐藏的访问权限synthetic

首先参考一下 Java8 虚拟机规范中对[The Synthetic Attribute](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.7.8)的相关描述:

> A class member that does not appear in the source code must be marked using a Synthetic attribute, or else it must have its ACC_SYNTHETIC flag set. The only exceptions to this requirement are compiler-generated methods which are not considered implementation artifacts, namely the instance initialization method representing a default constructor of the Java programming language, the class initialization method, and the Enum.values() and Enum.valueOf() methods.
>
> The Synthetic attribute was introduced in JDK 1.1 to support nested classes and interfaces.

一个类的源码中未出现的成员必须使用`synthetic`属性或者`ACC_SYNTHETIC`标志，但例外是编译期所生成的默认构造函数、类初始化方法，以及`Enum.values()`和`Enum.valueOf()`。

首先我来尝试着编译出一些存在`synthetic`访问权限的字节码：

``` java
public final class Singleton {
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
    private static final class Holder {
        private static final Singleton INSTANCE = new Singleton();
        private Holder() {
            throw new UnsupportedOperationException();
        }
    }
    private Singleton() {
        // TODO
    }
}
```

这是一个典型的静态内部类单例模式，人工反编译结果如下：

``` java
public final class Singleton {
    public static Singleton getInstance() {
        return Holder.access$000();
    }
    private static final class Holder {
        private static final Singleton INSTANCE;
        static {
            INSTANCE = new Singleton(null);
        }
        private Holder() {
            throw new UnsupportedOperationException();
        }
        static Singleton access$000() { // synthetic
            return INSTANCE;
        }
    }
    private Singleton() {
        super();
        // TODO
    }
    Singleton(Singleton$1 var0) { // synthetic
        this();
    }
    static class Singleton$1 { // synthetic
    }
}
```

在 Java8 版本下，编译出的代码实际上会多出来两个方法和一个类，从而增大包体积，如果比较在意二进制包体积的大小，建议在这种场景下不要添加`private`访问权限修饰符。

`private`访问权限我们很熟悉，私有成员仅被允许在其所在的类以及与其有嵌套关系的类中使用，在 Java11 以下的版本中，实现嵌套类之间访问私有成员的方式，就是在编译期额外生成一些包私有的方法，比如上例中的三处。

嵌套类之间访问私有方法与私有变量，编译期会额外生成带有`ACC_SYNTHETIC`标志的包私有方法去间接地访问私有成员，这些方法的名字形如 access$000，数字与这些自动生成的方法的顺序有关。对于构造函数，由于其名称必然为`<init>`，无法用改名的方式实现，因此编译器会额外定义一个类，并以重载的方式自动生成包私有构造函数。实际上在编译后将上例中自动生成的类直接删除也不会影响程序正常运行，因为这个参数被传入`null`，所以这个类并不会被加载。

在我上次一篇文章讲`switch`枚举的时候所生成的类，和这里生成的类是同一个，那个用于`switch`枚举的`int[]`也带有`ACC_SYNTHETIC`标志。

由于这些类、方法、字段在源代码中并不存在，因此当发布二进制包后，引用这些二进制包的代码，也不应访问得到这些带有`ACC_SYNTHETIC`标志的符号，于是编译器将以找不到符号为理由拒绝编译，甚至即使带有`ACC_PUBLIC`符号也不行。

可以通过使用[asm框架](https://asm.ow2.io/)修改字节码的方式，强行添加`ACC_SYNTHETIC`标志，从而将一部分`public`的API隐藏起来。我已经通过实验证实了这种操作的可行性，在此就不详细展开了。

## 参考文献

- [Java 虚拟机规范（Java SE 8 版本）](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)
- [JEP 181: Nest-Based Access Control](http://openjdk.java.net/jeps/181)
