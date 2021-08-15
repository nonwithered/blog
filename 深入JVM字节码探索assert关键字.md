# 深入JVM字节码探索assert关键字

## 引言

如果熟悉 C 语言，那么也许会使用过`assert.h`中的`assert`函数，在 Java 中的`assert`关键字也能够提供运行时断言这一功能，不过不同之处在于，Java 的断言可以在运行中决定是否开启，因此不必重新编译字节码。

本文主要对`assert`这个关键字在 JVM 字节码层面的实现原理进行分析，讨论其运行时配置断言启用与禁用的方式。

## 目录

1. `assert`关键字基础
2. `assert`实现原理分析
3. `assert`运行时配置启用与禁用

## 1. `assert`关键字基础

``` java
class Main {
    public static void main(String[] args) {
        assert null instanceof Object : "Hello, world!";
    }
}
```

用法很简单，只需要一个参数或者两个参数，在运行时检查第一个参数的值是否为`true`。

这段程序直接运行不会输出任何内容，因为默认禁用断言，启用断言可以使用`-ea`选项。

``` bash
$ javac Main.java
$ java -ea Main
Exception in thread "main" java.lang.AssertionError: Hello, world!
        at Main.main(Main.java:3)
```

## 2. `assert`实现原理分析

首先看一看上面这个类的字节码：

``` java
class Main {
  static final boolean $assertionsDisabled;

  Main();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: getstatic     #2                  // Field $assertionsDisabled:Z
       3: ifne          23
       6: aconst_null
       7: instanceof    #3                  // class java/lang/Object
      10: ifne          23
      13: new           #4                  // class java/lang/AssertionError
      16: dup
      17: ldc           #5                  // String Hello, world!
      19: invokespecial #6                  // Method java/lang/AssertionError."<init>":(Ljava/lang/Object;)V
      22: athrow
      23: return

  static {};
    Code:
       0: ldc           #7                  // class Main
       2: invokevirtual #8                  // Method java/lang/Class.desiredAssertionStatus:()Z
       5: ifne          12
       8: iconst_1
       9: goto          13
      12: iconst_0
      13: putstatic     #2                  // Field $assertionsDisabled:Z
      16: return
}
```

人工反编译：

``` java
class Main {
    static final boolean $assertionsDisabled;
    public static void main(String[] args) {
        if (!$assertionsDisabled) {
            if (!(null instanceof Object)) {
                throw new AssertionError("Hello, world!");
            }
        }
    }
    static {
        $assertionsDisabled = !Main.class.desiredAssertionStatus() ? true : false;
    }
}
```

在每个使用了`assert`语句的类中，都被添加了一个包私有布尔常量`$assertionsDisabled`，在每一个`assert`语句处都要检查这个值是否为`false`，若为`false`则判断`assert`关键字的第一个表达式的值是否为`false`，若为`false`则以第二个表达式的值作为构造函数的参数抛出一个`AssertionError`，若这个`assert`语句仅有一个表达式则以无参构造函数实例化一个`AssertionError`并抛出。

`AssertionError`这个异常的构造函数有许多种重载，对于除了`()V`、`(Ljava/lang/String;Ljava/lang/Throwable;)V`和`(Ljava/lang/String;)V`之外的 7 种重载，它们会通过`String`类的静态方法`valueOf`将参数转变为字符串然后再调用构造函数的`(Ljava/lang/String;)V`重载。特别的是，对于`(Ljava/lang/Object;)V`这个重载，会判断参数是否为`Throwable`实例，如果是`Throwable`实例则会通过`Throwable`的`initCause`实例方法为`cause`赋值。

## 3. `assert`运行时配置启用与禁用

在[Java语言规范](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.10)中提出了一个类似这样的示例：

``` java
class Main {
    public static void main(String[] args) throws Exception {
        Class.forName("Main$Test");
    }
    static class Test extends Foobar {
        static void test() {
            assert null instanceof Object : "Hello, world!";
        }
    }
    static class Foobar {
        static {
            Test.test();
        }
    }
}
```

在上面这个示例中，无论是否使用`-ea`选项，这个断言都会被触发。

首先看一下堆栈：

``` bash
java Main
Exception in thread "main" java.lang.AssertionError: Hello, world!
        at Main$Test.test(Main.java:7)
        at Main$Foobar.<clinit>(Main.java:12)
        at java.base/java.lang.Class.forName0(Native Method)
        at java.base/java.lang.Class.forName(Class.java:315)
        at Main.main(Main.java:3)
```

在 Main#main 中通过 java.lang.Class#forName 加载 Main\$Test 类，由于 Main\$Foobar 是 Main\$Test 的超类，因此要加载 Main\$Foobar 类并在 Main\$Test#\<clinit\> 被调用之前先调用 Main\$Foobar#\<clinit\>，但是在 Main\$Foobar#\<clinit\> 却中调用了 Main\$Test#test，但是此时尚未调用Main\$Test#\<clinit\>，所以被 Main\$Test#test 所访问到的 Main\$Test#\$assertionsDisabled 在此时尚未被初始化，其值为默认值`false`，无论是否使用`-ea`选项。

当一个使用了断言的类被加载时，在这个类的`<clinit>`中将调用`java.lang.Class#desiredAssertionStatus`并根据它的返回值为`$assertionsDisabled`初始化。

此处参考`java.lang.Class#desiredAssertionStatus`的源码：

``` java
/**
 * Returns the assertion status that would be assigned to this
 * class if it were to be initialized at the time this method is invoked.
 * If this class has had its assertion status set, the most recent
 * setting will be returned; otherwise, if any package default assertion
 * status pertains to this class, the most recent setting for the most
 * specific pertinent package default assertion status is returned;
 * otherwise, if this class is not a system class (i.e., it has a
 * class loader) its class loader's default assertion status is returned;
 * otherwise, the system class default assertion status is returned.
 * <p>
 * Few programmers will have any need for this method; it is provided
 * for the benefit of the JRE itself.  (It allows a class to determine at
 * the time that it is initialized whether assertions should be enabled.)
 * Note that this method is not guaranteed to return the actual
 * assertion status that was (or will be) associated with the specified
 * class when it was (or will be) initialized.
 *
 * @return the desired assertion status of the specified class.
 * @see    java.lang.ClassLoader#setClassAssertionStatus
 * @see    java.lang.ClassLoader#setPackageAssertionStatus
 * @see    java.lang.ClassLoader#setDefaultAssertionStatus
 * @since  1.4
 */
public boolean desiredAssertionStatus() {
    ClassLoader loader = getClassLoader0();
    // If the loader is null this is a system class, so ask the VM
    if (loader == null)
        return desiredAssertionStatus0(this);

    // If the classloader has been initialized with the assertion
    // directives, ask it. Otherwise, ask the VM.
    synchronized(loader.assertionLock) {
        if (loader.classAssertionStatus != null) {
            return loader.desiredAssertionStatus(getName());
        }
    }
    return desiredAssertionStatus0(this);
}

// Retrieves the desired assertion status of this class from the VM
private static native boolean desiredAssertionStatus0(Class<?> clazz);
```

首先会获取这个类的类加载器，假如这个类加载器为`null`即[Bootstrap Class Loader](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-5.html#jvms-5.3.1)，则返回`desiredAssertionStatus0`的结果，这个结果对于上面的 Main\$Test 来说，取决于是否使用了`-ea`参数，但是由于 Main\$Test 的类加载器并不是`null`，所以`desiredAssertionStatus`不会在这里直接返回。

类加载器的`classAssertionStatus`是一个`Map<String, Boolean>`，如果不在代码中通过`java.lang.ClassLoader`的`setClassAssertionStatus`、`setPackageAssertionStatus`和`setDefaultAssertionStatus`专门设置特定的断言开关，也没有调用过`java.lang.ClassLoader#clearAssertionStatus`，则这个`Map`将始终为`null`，因此 Main\$Test 的`desiredAssertionStatus`所返回的结果仍旧取决于是否使用了`-ea`选项。

最后参考一下`java.lang.ClassLoader#desiredAssertionStatus`的源码：

``` java
/**
 * Returns the assertion status that would be assigned to the specified
 * class if it were to be initialized at the time this method is invoked.
 * If the named class has had its assertion status set, the most recent
 * setting will be returned; otherwise, if any package default assertion
 * status pertains to this class, the most recent setting for the most
 * specific pertinent package default assertion status is returned;
 * otherwise, this class loader's default assertion status is returned.
 * </p>
 *
 * @param  className
 *         The fully qualified class name of the class whose desired
 *         assertion status is being queried.
 *
 * @return  The desired assertion status of the specified class.
 *
 * @see  #setClassAssertionStatus(String, boolean)
 * @see  #setPackageAssertionStatus(String, boolean)
 * @see  #setDefaultAssertionStatus(boolean)
 *
 * @since  1.4
 */
boolean desiredAssertionStatus(String className) {
    synchronized (assertionLock) {
        // assert classAssertionStatus   != null;
        // assert packageAssertionStatus != null;

        // Check for a class entry
        Boolean result = classAssertionStatus.get(className);
        if (result != null)
            return result.booleanValue();

        // Check for most specific package entry
        int dotIndex = className.lastIndexOf('.');
        if (dotIndex < 0) { // default package
            result = packageAssertionStatus.get(null);
            if (result != null)
                return result.booleanValue();
        }
        while(dotIndex > 0) {
            className = className.substring(0, dotIndex);
            result = packageAssertionStatus.get(className);
            if (result != null)
                return result.booleanValue();
            dotIndex = className.lastIndexOf('.', dotIndex-1);
        }

        // Return the classloader default
        return defaultAssertionStatus;
    }
}
```

这个方法将通过在`classAssertionStatus`和`packageAssertionStatus`这两个`Map`中查找对应的类名以及各级包名的断言开关，没有找到相关的设置则直接返回`defaultAssertionStatus`，我们可以用`java.lang.ClassLoader`的`setClassAssertionStatus`、`setPackageAssertionStatus`和`setDefaultAssertionStatus`设置某个类、某个包或者某个类加载器的断言启用与禁用，这种设定生效的优先级将高于`-ea`选项。

## 参考文献

- [Java 语言规范（Java SE 8 版本）](https://docs.oracle.com/javase/specs/jls/se8/html/index.html)
