# 深入JVM字节码探索switch指令、字符串、枚举

## 引言

从 C 到 C++ 到 Java 到一系列各种各样的语言，大多都支持多路分支语句，比如 Kotlin 的`when`和 Rust 的`match`等等，在 Java SE 14 版本的语言规范也添加了对`switch`表达式的支持。

本文主要针对 Java SE 8 版本中的`switch`语句从字节码层面进行研究，理解`switch`语句相关的各种细节，并尝试着对其编译产物人工地进行反编译，探索字符串`switch`和枚举`switch`的具体实现方式。

## 目录

1. `switch`关键字基础
2. `switch`所对应的两种指令
3. `switch`字符串的实现原理
4. `switch`枚举的实现原理

## 1. `switch`关键字基础

首先，引用一下语言规范中的下面几句话：

[The switch Statement](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.11)

> The switch Statement
>
> The switch statement transfers control to one of several statements depending on the value of an expression.
>
> The type of the Expression must be char, byte, short, int, Character, Byte, Short, Integer, String, or an enum type, or a compile-time error occurs.
>
> When the switch statement is executed, first the Expression is evaluated. If the Expression evaluates to null, a NullPointerException is thrown and the entire switch statement completes abruptly for that reason. Otherwise, if the result is of a reference type, it is subject to unboxing conversion.

这里提到`switch`所接收的表达式参数必须为`char`、`byte`、`short`、`int`、`Character`、`Byte`、`Short`、`Integer`、`String`、`Enum`，若该表达式参数为`null`，则抛出 NPE，否则若为引用类型则需要进行拆箱转换。

实际上`switch`在字节码层面可以认为它仅仅只支持`int`一种类型，下文我会对此进行解释。

然后我们可以在这里注意到几个细节：

1. 这里的参数类型应是编译期确定的类型，而不是运行时类型。

2. 这里的参数在运行时不可为`null`，否则将抛出 NPE。

3. 支持`Character`、`Byte`、`Short`、`Integer`这四种类型的参数

    在编译期将分别对这四种类型的参数通过`charValue`、`byteValue`、`shortValue`、`intValue`这四个方法转变为`char`、`byte`、`short`、`int`类型，也就是说实质上对于`Character`、`Byte`、`Short`、`Integer`的`switch`实质上仍旧是对于`char`、`byte`、`short`、`int`的`switch`。

4. `switch`不支持`Long`、`Double`、`Float`、`Boolean`类型的参数

    通过对上一条的理解，`switch`不支持`Long`、`Double`、`Float`、`Boolean`类型的原因应该是因为`switch`不支持`long`、`double`、`float`、`boolean`。

5. `switch`不支持`long`、`double`、`float`类型的参数

    因为从`long`、`double`、`float`类型向`int`类型进行转换可能会造成损失，所以编译期不会轻易地将它们隐式转换为`int`类型，我们只能在自己的源代码中手动地进行显式强制类型转换，才可以将它们转为`int`类型。

6. `switch`不支持`boolean`类型的参数

    从`boolean`向`int`的转换是没有损失的，但是实际上我们并没有用`switch`对`boolean`类型的参数进行多路分支的必要，毕竟我们可以直接使用`if`语句。

到此为止，我们遗留了几个主要的问题：

1. `switch`是如何支持`int`类型的

2. `switch`如何依赖对`int`的支持而提供对`char`、`byte`、`short`的支持

3. `switch`指令如何进行多路分支的跳转

4. `switch`如何依赖对`int`的支持而提供对`String`的支持

5. `switch`如何依赖对`int`的支持而提供对`Enum`的支持

下文将对以上问题进行讨论与解答。

## 2. `switch`所对应的两种指令

[Compiling Switches](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-3.html#jvms-3.10)

> Compilation of switch statements uses the tableswitch and lookupswitch instructions.
>
> The Java Virtual Machine's tableswitch and lookupswitch instructions operate only on int data. Because operations on byte, char, or short values are internally promoted to int, a switch whose expression evaluates to one of those types is compiled as though it evaluated to type int.

`switch`语句要使用`tableswitch`和`lookupswitch`这两个指令，这两个指令只针对`int`类型进行操作，而`char`、`byte`、`short`这三种类型将被隐式转换为`int`。

如果熟悉 JVM 字节码指令集，那么应该很容易理解这两种`switch`仅仅支持`int`类型的原因，事实上 JVM 中许多操作都没有对每种基本类型都专门设计单独的指令，这是因为 JVM 的所有指令都仅有一个字节而已，这样的好处是不必进行对齐，因此效率比较高，但是其弊端就是最多只能提供 256 种指令，假如真的让所有操作都同时对`boolean`、`char`、`float`、`double`、`byte`、`short`、`int`、`long`以及引用这 9 种类型都提供支持，那么 JVM 将最多只能支持二三十种操作，这绝对是不够用的，因此许多的操作都仅仅只支持其中的一部分类型，而其余的类型将在运行过程中经过类型转换。

## 2.1 `tableswitch`

对以下程序进行编译然后进行反编译：

```java
class Test {
    static int test(int var0) {
        switch (var0) {
            case 0:
                return 0;
            case 2:
                return 2;
            case 3:
                return 3;
            default:
                return -1;
        }
    }
}
```

```java
Compiled from "Test.java"
class Test {
  Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  static int test(int);
    Code:
       0: iload_0
       1: tableswitch   { // 0 to 3
                     0: 32
                     1: 38
                     2: 34
                     3: 36
               default: 38
          }
      32: iconst_0
      33: ireturn
      34: iconst_2
      35: ireturn
      36: iconst_3
      37: ireturn
      38: iconst_m1
      39: ireturn
}
```

上面这个类的文件在我编译后是这样的：

```
  Offset: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 	
00000000: CA FE BA BE 00 00 00 37 00 10 0A 00 03 00 0D 07    J~:>...7........
00000010: 00 0E 07 00 0F 01 00 06 3C 69 6E 69 74 3E 01 00    ........<init>..
00000020: 03 28 29 56 01 00 04 43 6F 64 65 01 00 0F 4C 69    .()V...Code...Li
00000030: 6E 65 4E 75 6D 62 65 72 54 61 62 6C 65 01 00 04    neNumberTable...
00000040: 74 65 73 74 01 00 04 28 49 29 49 01 00 0D 53 74    test...(I)I...St
00000050: 61 63 6B 4D 61 70 54 61 62 6C 65 01 00 0A 53 6F    ackMapTable...So
00000060: 75 72 63 65 46 69 6C 65 01 00 09 54 65 73 74 2E    urceFile...Test.
00000070: 6A 61 76 61 0C 00 04 00 05 01 00 04 54 65 73 74    java........Test
00000080: 01 00 10 6A 61 76 61 2F 6C 61 6E 67 2F 4F 62 6A    ...java/lang/Obj
00000090: 65 63 74 00 20 00 02 00 03 00 00 00 00 00 02 00    ect.............
000000a0: 00 00 04 00 05 00 01 00 06 00 00 00 1D 00 01 00    ................
000000b0: 01 00 00 00 05 2A B7 00 01 B1 00 00 00 01 00 07    .....*7..1......
000000c0: 00 00 00 06 00 01 00 00 00 01 00 08 00 08 00 09    ................
000000d0: 00 01 00 06 00 00 00 5C 00 01 00 01 00 00 00 28    .......\.......(
000000e0: 1A AA 00 00 00 00 00 25 00 00 00 00 00 00 00 03    .*.....%........
000000f0: 00 00 00 1F 00 00 00 25 00 00 00 21 00 00 00 23    .......%...!...#
00000100: 03 AC 05 AC 06 AC 02 AC 00 00 00 02 00 07 00 00    .,.,.,.,........
00000110: 00 16 00 05 00 00 00 03 00 20 00 05 00 22 00 07    ............."..
00000120: 00 24 00 09 00 26 00 0B 00 0A 00 00 00 06 00 04    .$...&..........
00000130: 20 01 01 01 00 01 00 0B 00 00 00 02 00 0C          ..............
```

从 0x0x000000e1 至 0x0x000000ff 的这 31 个字节便是`tableswitch`。

按照[tableswitch](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.tableswitch)的解释，`tableswitch`即 0xAA，其后分别跟随 default、low、high，在本例中即分别为37、0、3，再其后跟随其余的 high - low + 1 即 4 个 offset，在本例中分别为 31、37、33、35。

这里需要注意一个细节，那就是在 0xAA 与 default 之间存在 0 至 3 个填充字节，目的在于保证 default、low、high 以及后面的每一个 offset 在 class 文件中相对于这个方法的第一个指令的地址偏移都是 4 的倍数，在本例中填充字节有 2 个，该方法的第一个指令的地址是 0x000000e0。

`tableswitch`的`int`参数从操作数栈中弹出，并直接查表然后跳转。此处应注意到上面的源码中并不存在`case 1`，但是在字节码中却在`case 1`的位置存在一个和 default 相等的值。

这是因为`tableswitch`需要直接查表获取即将跳转的地址偏移，这个偏移从 0xAA 的位置开始计算，所以这个映射表必须支持随机读取，也就是说所有的`case`都要在 low 至 high 的范围内顺序排列，若参数小于 low 或大于 high 则跳转到 default 的位置，而在 table 内部被填充的表项也对应此 default 值。

`tableswitch`看起来应十分高效，因为虚拟机可以直接查表从而得到对应的 offset，但是这里也有一个缺陷，也就是上一段刚刚提到的，需要保证每个`case`的连续，即使源码中并不存在这个`case`，也要在编译后填充。因此`tableswitch`仅适用于`switch`的`case`相对来说比较密集的情况下，而在其比较稀疏的情况下则不应使用`tableswitch`而应使用`lookupswitch`。

## 2.2 `lookupswitch`

对以下程序进行编译然后进行反编译：

```java
class Test {
    static int test(int var0) {
        switch (var0) {
            case 1000:
                return 0;
            case 100:
                return 1;
            case 10:
                return 2;
            default:
                return -1;
        }
    }
}
```

```java
Compiled from "Test.java"
class Test {
  Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  static int test(int);
    Code:
       0: iload_0
       1: lookupswitch  { // 3
                    10: 40
                   100: 38
                  1000: 36
               default: 42
          }
      36: iconst_0
      37: ireturn
      38: iconst_1
      39: ireturn
      40: iconst_2
      41: ireturn
      42: iconst_m1
      43: ireturn
}
```

上面这个类的文件在我编译后是这样的：

```
  Offset: 00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F 	
00000000: CA FE BA BE 00 00 00 37 00 10 0A 00 03 00 0D 07    J~:>...7........
00000010: 00 0E 07 00 0F 01 00 06 3C 69 6E 69 74 3E 01 00    ........<init>..
00000020: 03 28 29 56 01 00 04 43 6F 64 65 01 00 0F 4C 69    .()V...Code...Li
00000030: 6E 65 4E 75 6D 62 65 72 54 61 62 6C 65 01 00 04    neNumberTable...
00000040: 74 65 73 74 01 00 04 28 49 29 49 01 00 0D 53 74    test...(I)I...St
00000050: 61 63 6B 4D 61 70 54 61 62 6C 65 01 00 0A 53 6F    ackMapTable...So
00000060: 75 72 63 65 46 69 6C 65 01 00 09 54 65 73 74 2E    urceFile...Test.
00000070: 6A 61 76 61 0C 00 04 00 05 01 00 04 54 65 73 74    java........Test
00000080: 01 00 10 6A 61 76 61 2F 6C 61 6E 67 2F 4F 62 6A    ...java/lang/Obj
00000090: 65 63 74 00 20 00 02 00 03 00 00 00 00 00 02 00    ect.............
000000a0: 00 00 04 00 05 00 01 00 06 00 00 00 1D 00 01 00    ................
000000b0: 01 00 00 00 05 2A B7 00 01 B1 00 00 00 01 00 07    .....*7..1......
000000c0: 00 00 00 06 00 01 00 00 00 01 00 08 00 08 00 09    ................
000000d0: 00 01 00 06 00 00 00 60 00 01 00 01 00 00 00 2C    .......`.......,
000000e0: 1A AB 00 00 00 00 00 29 00 00 00 03 00 00 00 0A    .+.....)........
000000f0: 00 00 00 27 00 00 00 64 00 00 00 25 00 00 03 E8    ...'...d...%...h
00000100: 00 00 00 23 03 AC 04 AC 05 AC 02 AC 00 00 00 02    ...#.,.,.,.,....
00000110: 00 07 00 00 00 16 00 05 00 00 00 03 00 24 00 05    .............$..
00000120: 00 26 00 07 00 28 00 09 00 2A 00 0B 00 0A 00 00    .&...(...*......
00000130: 00 06 00 04 24 01 01 01 00 01 00 0B 00 00 00 02    ....$...........
00000140: 00 0C                                              ..
```

从 0x0x000000e1 至 0x0x00001003 的这 35 个字节便是`lookupswitch`。

按照[lookupswitch](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.lookupswitch)的解释，`lookupswitch`即0xAB，其后分别跟随 default、npairs，在本例中即分别为41、3，再其后跟随其余的 npairs 即 3 组映射，在本例中分别为10、39、100、37、1000、35。

`lookupswitch`也需要填充 0 至 3 个字节，本例中是 2 个，该方法第一个指令的地址为 0x000000e0。

`lookupswitch`的`int`参数也从操作数栈中弹出，并在这些`case`中查找匹配项。在我上面的源码中`case`顺序是 1000、100、10，但是在字节码中却是 10、100、1000，这是因为`lookupswitch`要求所有 offset 应以递增的顺序排列，从而使 JVM 可以支持比线性更高效的查找方式，比如二分查找，但是规范中在此并没有要求虚拟机所实现的查找方式。

由于`lookupswitch`需要进行查找，而不能像`tableswitch`那样直接查表，因此或许效率会有所降低，但是在`case`相对比较稀疏的情况下，比起`tableswitch`，`lookupswitch`将节省大量的空间。

对于我们在代码中的`switch`语句，选择`lookupswitch`或是`tableswitch`将依赖于编译器的具体实现。

## 3. `switch`字符串的实现原理

以下代码作为示例：

``` java
class Test {
    static int test(String var0) {
        switch (var0) {
            case "foo":
                return 0;
            case "bar":
                return 1;
            case "10":
                return 2;
            case "0O":
                return 3;
            default:
                return -1;
        }
    }
}
```

编译后的字节码：

``` java
Compiled from "Test.java"
class Test {
  Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  static int test(java.lang.String);
    Code:
       0: aload_0
       1: astore_1
       2: iconst_m1
       3: istore_2
       4: aload_1
       5: invokevirtual #2                  // Method java/lang/String.hashCode:()I
       8: lookupswitch  { // 3
                  1567: 72
                 97299: 58
                101574: 44
               default: 97
          }
      44: aload_1
      45: ldc           #3                  // String foo
      47: invokevirtual #4                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      50: ifeq          97
      53: iconst_0
      54: istore_2
      55: goto          97
      58: aload_1
      59: ldc           #5                  // String bar
      61: invokevirtual #4                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      64: ifeq          97
      67: iconst_1
      68: istore_2
      69: goto          97
      72: aload_1
      73: ldc           #6                  // String 0O
      75: invokevirtual #4                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      78: ifeq          86
      81: iconst_3
      82: istore_2
      83: goto          97
      86: aload_1
      87: ldc           #7                  // String 10
      89: invokevirtual #4                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      92: ifeq          97
      95: iconst_2
      96: istore_2
      97: iload_2
      98: tableswitch   { // 0 to 3
                     0: 128
                     1: 130
                     2: 132
                     3: 134
               default: 136
          }
     128: iconst_0
     129: ireturn
     130: iconst_1
     131: ireturn
     132: iconst_2
     133: ireturn
     134: iconst_3
     135: ireturn
     136: iconst_m1
     137: ireturn
}
```

人工地反编译：

``` java
class Test {
    static int test(String var0) {
        String var1 = var0;
        int var2 = -1;
        switch (var1.hashCode()) {
            case 101574:
                if (var1.equals("foo")) {
                    var2 = 0;
                }
                break;
            case 97299:
                if (var1.equals("bar")) {
                    var2 = 1;
                }
                break;
            case 1567:
                if (var1.equals("0O")) {
                    var2 = 3;
                } else if (var1.equals("10")) {
                    var2 = 2;
                }
                break;
        }
        switch (var2) {
            case 0:
                return 0;
            case 1:
                return 1;
            case 2:
                return 2;
            case 3:
                return 3;
            default:
                return -1;
        }
    }
}
```

在我的环境下，这和上面的字符串`switch`编译结果几乎完全一致，`javap -c`无任何区别。

对字符串的`switch`实际上被拆分成了 5 个步骤：

1. 调用参数字符串的`hashCode`方法获取 hash 值

2. 对其 hash 值进行第一次`switch`

3. 调用`equals`方法并修改状态码

4. 对状态码进行第二次`switch`

5. 执行对应分支中的语句

这里注意几个细节：

1. 若参数字符串为`null`，则调用`hashCode`方法将抛出 NPE，符合虚拟机规范的要求。

2. 状态码的默认值为 -1，若参数字符串没有匹配任何一个`case`字面量，则会保持不变，否则将被修改为对应`case`在源代码中的序号，从 0 开始计算。

3. `case`字符串字面量的 hash 值需要在编译期经过计算并写入 class 字节码文件中，而参数字符串的`hashCode`方法需要在运行时调用才能够得到结果，这就要求同一个字符串的 hash 算法必须在编译期和运行时是对应的，否则经过第一次`switch`后状态码将被赋予错误的值，于是在第二次`switch`将走入错误的分支路径中，执行错误的逻辑。

    由于虚拟机会对被加载的类进行版本验证，因此 hash 算法的一致在类加载的流程中可以被虚拟机所保证。

4. 仅对参数字符串的 hash 进行`switch`不足以确定其是否与`case`字面量匹配，因此至少会调用一次`equals`方法。

    比如上例中`"0O"`和`"10"`这两个字符串的 hash 值相同，所以在完成对 hash 值的`switch`后，要以这两个`case`的倒序逐个调用`equals`方法并判断结果。

    此处可以参考 Java 8 中`java.lang.String`的源码：

``` java
/**
    * Returns a hash code for this string. The hash code for a
    * {@code String} object is computed as
    * <blockquote><pre>
    * s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
    * </pre></blockquote>
    * using {@code int} arithmetic, where {@code s[i]} is the
    * <i>i</i>th character of the string, {@code n} is the length of
    * the string, and {@code ^} indicates exponentiation.
    * (The hash value of the empty string is zero.)
    *
    * @return  a hash code value for this object.
    */
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

自定义类的 hash 算法在编译期是无法确定的，因此`switch`不能以和字符串字面量相同的方式实现自定义类的多路分支。

## 4. `switch`枚举的实现原理

以下代码作为示例：

```java
enum Foobar {
    FOO,
    BAR;
}
class Test {
    static int test(Foobar var0) {
        switch (var0) {
            case FOO:
                return 1;
            case BAR:
                return 2;
            default:
                return 0;
        }
    }
}
```

编译后的字节码：

```java
Compiled from "Test.java"
class Test {
  Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  static int test(Foobar);
    Code:
       0: getstatic     #2                  // Field Test$1.$SwitchMap$Foobar:[I
       3: aload_0
       4: invokevirtual #3                  // Method Foobar.ordinal:()I
       7: iaload
       8: lookupswitch  { // 2
                     1: 36
                     2: 38
               default: 40
          }
      36: iconst_1
      37: ireturn
      38: iconst_2
      39: ireturn
      40: iconst_0
      41: ireturn
}
Compiled from "Test.java"
class Test$1 {
  static final int[] $SwitchMap$Foobar;

  static {};
    Code:
       0: invokestatic  #1                  // Method Foobar.values:()[LFoobar;
       3: arraylength
       4: newarray       int
       6: putstatic     #2                  // Field $SwitchMap$Foobar:[I
       9: getstatic     #2                  // Field $SwitchMap$Foobar:[I
      12: getstatic     #3                  // Field Foobar.FOO:LFoobar;
      15: invokevirtual #4                  // Method Foobar.ordinal:()I
      18: iconst_1
      19: iastore
      20: goto          24
      23: astore_0
      24: getstatic     #2                  // Field $SwitchMap$Foobar:[I
      27: getstatic     #6                  // Field Foobar.BAR:LFoobar;
      30: invokevirtual #4                  // Method Foobar.ordinal:()I
      33: iconst_2
      34: iastore
      35: goto          39
      38: astore_0
      39: return
    Exception table:
       from    to  target type
           9    20    23   Class java/lang/NoSuchFieldError
          24    35    38   Class java/lang/NoSuchFieldError
}
```

人工地反编译：

```java
class Test {
    static int test(Foobar var0) {
        switch (Test$1.$SwitchMap$Foobar[var0.ordinal()]) {
            case 1:
                return 1;
            case 2:
                return 2;
            default:
                return 0;
        }
    }
}
class Test$1 {
    static final int[] $SwitchMap$Foobar;
    static {
        $SwitchMap$Foobar = new int[Foobar.values().length];
        try {
            $SwitchMap$Foobar[Foobar.FOO.ordinal()] = 1;
        } catch (NoSuchFieldError e) {
            ;
        }
        try {
            $SwitchMap$Foobar[Foobar.BAR.ordinal()] = 2;
        } catch (NoSuchFieldError e) {
            ;
        }
    }
}
```

在我的环境下，这和上面的枚举`switch`编译结果几乎完全一致，唯一的不同之处是在自动生成的那个匿名内部类中并没有包私有无参构造函数，但手动编写的类会被自动添加一个包私有无参构造函数。

这个 Test\$1 类是自动生成的包私有静态匿名内部类，匿名类的名字都是其外部类的名字加`'$'`加数字，`'$'`在每层嵌套类的名字之间作为间隔符，数字则是为匿名类自动生成的名字，其值在编译期被确定，因此可能在迭代过程中随着项目版本的变化而变化，本例中这个数字为 1。

这里可以发现，在 Test 类中对 Foobar 枚举的`switch`被转变成了对 Test\$1.\$SwitchMap\$Foobar[var0.ordinal()] 这一整数表达式的`switch`，这里 Test\$1 的静态成员 \$SwitchMap\$Foobar 也在编译期自动生成，这是一个`int`数组，被选取的参数在这个数组中的索引是这个 Foobar 对象的`ordinal`方法的返回值。

`ordinal`方法的返回值是这个枚举常量在其声明时的序号。这里参考一下`Enum`类的源码：

```java
/**
 * The ordinal of this enumeration constant (its position
 * in the enum declaration, where the initial constant is assigned
 * an ordinal of zero).
 *
 * Most programmers will have no use for this field.  It is designed
 * for use by sophisticated enum-based data structures, such as
 * {@link java.util.EnumSet} and {@link java.util.EnumMap}.
 */
private final int ordinal;

/**
 * Returns the ordinal of this enumeration constant (its position
 * in its enum declaration, where the initial constant is assigned
 * an ordinal of zero).
 *
 * Most programmers will have no use for this method.  It is
 * designed for use by sophisticated enum-based data structures, such
 * as {@link java.util.EnumSet} and {@link java.util.EnumMap}.
 *
 * @return the ordinal of this enumeration constant
 */
public final int ordinal() {
    return ordinal;
}
```

这里注意若枚举参数为`null`则调用`ordinal`方法会抛出 NPE，符合虚拟机规范的要求。

\$SwitchMap\$Foobar 数组的长度与在 Test 类中所依赖的那个版本的枚举 Foobar 类的枚举常量个数相同，这个数组在这个自动生成的包私有静态匿名内部类被加载时被初始化，分别取出在 Test 类中所依赖的那个版本的枚举 Foobar 类的所有枚举常量，并在它们的`ordinal`的位置设置`ordinal + 1`的值。

因此，当在 Test 类中对 Test\$1.\$SwitchMap\$Foobar[var0.ordinal()] 这个`int`类型的表达式进行`switch`时，若这个`int`参数为正数，则可以跳转到所对应的路径，并执行对应的语句，否则将跳转到`default`分支，在这种情况下参数通常为默认值`0`。

这里要注意一下，在 Test\$1 类中调用了 Foobar 类的静态方法`values`，它是在 Foobar 被编译时自动生成的。

这里稍微分析一下 Foobar 这个枚举类：

源代码：
```java
enum Foobar {
    FOO,
    BAR;
}
```

字节码：
```java
Compiled from "Foobar.java"
final class Foobar extends java.lang.Enum<Foobar> {
  public static final Foobar FOO;

  public static final Foobar BAR;

  public static Foobar[] values();
    Code:
       0: getstatic     #1                  // Field $VALUES:[LFoobar;
       3: invokevirtual #2                  // Method "[LFoobar;".clone:()Ljava/lang/Object;
       6: checkcast     #3                  // class "[LFoobar;"
       9: areturn

  public static Foobar valueOf(java.lang.String);
    Code:
       0: ldc           #4                  // class Foobar
       2: aload_0
       3: invokestatic  #5                  // Method java/lang/Enum.valueOf:(Ljava/lang/Class;Ljava/lang/String;)Ljava/lang/Enum;
       6: checkcast     #4                  // class Foobar
       9: areturn

  static {};
    Code:
       0: new           #4                  // class Foobar
       3: dup
       4: ldc           #7                  // String FOO
       6: iconst_0
       7: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
      10: putstatic     #9                  // Field FOO:LFoobar;
      13: new           #4                  // class Foobar
      16: dup
      17: ldc           #10                 // String BAR
      19: iconst_1
      20: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
      23: putstatic     #11                 // Field BAR:LFoobar;
      26: iconst_2
      27: anewarray     #4                  // class Foobar
      30: dup
      31: iconst_0
      32: getstatic     #9                  // Field FOO:LFoobar;
      35: aastore
      36: dup
      37: iconst_1
      38: getstatic     #11                 // Field BAR:LFoobar;
      41: aastore
      42: putstatic     #1                  // Field $VALUES:[LFoobar;
      45: return
}
```

反编译：
```java
final class Foobar extends java.lang.Enum<Foobar> {
    public static final Foobar FOO;
    public static final Foobar BAR;
    private static final Foobar[] $VALUES;
    public static Foobar[] values() {
        return (Foobar[]) $VALUES.clone();
    }
    public static Foobar valueOf(java.lang.String) {
        return (Foobar) java.lang.Enum.valueOf(Foobar.class, name);
    }
    private Foobar(String name, int ordinal) {
        super(name, ordinal);
    }
    static {
        FOO = new Foobar("FOO", 0);
        BAR = new Foobar("BAR", 1);
        $VALUES = new Foobar[] { FOO, BAR };
    }
}
```

假如 Test 还存在内部类，并且我们再定义了另外的两个枚举类 Foo 和 Bar，那么无论是在 Test 中对 Foo 和 Bar 参数执行`switch`，还是在 Test 的内部类中对 Foo 和 Bar 参数执行`switch`，都会依赖这同一个 Test\$1 类。

假如在 Test 及其内部类的源代码中，同时对枚举类 Foo 和 枚举类 Bar 执行了`switch`，那么在 Test\$1 类中将同时存在 \$SwitchMap\$Foo 和 \$SwitchMap\$Bar 这两个`int`数组。

如果两个不同的类 A 和类 B 都对枚举类 Foobar 执行了`switch`，或是分别对枚举类 Foo 和枚举类 Bar 执行了`switch`，无论如何只要这两个对枚举执行`switch`的类 A 和类 B 之间没有嵌套关系，那么就会分别生成 A\$1 和 B\$1 这两个类，哪怕 A\$1 和 B\$1 这两个类只有类名不同，也仍然会重复生成。

重复地生成几乎不变的冗余代码，会无谓地增多类的个数、增大包体积、浪费类加载的时间，但是实际上编译器这样处理是有必要的。

这里我们需要着重理解，Foobar、Test\$1、Test 这三个类之间的依赖关系：

Foobar 是在 Test 中被`switch`的枚举实例所属的类，在 Test 类被编译时，编译器将自动生成一个依赖于这个版本的 Test\$1 类。虽然 Test\$1 类依赖 Foobar 类的定义，但Test 类仅仅依赖 Foobar 类的声明，这就意味着即使在 Test\$1 类编译完成后，Foobar 类被修改并且重新编译，但是并没有重新编译 Test 类，那么 Test\$1 将不会因 Foobar 的改变而得到更新的机会，因为它只有在 Test 类被编译时才能够被编译。

上面这段话也许有些饶舌，那么我还是以一段代码作为实例：

首先编译 Foobar：

```java
enum Foobar {
    FOO,
    BAR;
}
```

```bash
javac Foobar.java
```

然后编译 Test：

```java
class Test {
    static int test(Foobar var0) {
        switch (var0) {
            case FOO:
                return 1;
            case BAR:
                return 2;
            default:
                return 0;
        }
    }
}
```

```bash
javac Test.java
```

然后修改 Foobar：

```java
enum Foobar {
    FOOBAR,
    NIL;
}
```

```bash
javac Foobar.java
```

然后执行以下程序：

```java
class Main {
    public static void main(String[] args) {
        System.out.println(Test.test(Foobar.FOOBAR));
    }
}
```

```bash
javac Main.java
java Main
```

结果为 0。

以上操作建议使用命令行，但是使用 IDE 重现一遍也许印象会更为深刻：

- 在 foobar 项目中将 Foobar 类作为 JAR 打包发布第一个版本
- 在 test 项目中的`pom.xml`或者`build.gradle`中添加对第一个版本的 foobar 项目的依赖并将 Test 类和 Test\$1 类作为 JAR 打包发布
- 在 foobar 项目中修改 Foobar 类的代码并作为 JAR 打包发布第二个版本
- 在 main 项目中的`pom.xml`或者`build.gradle`中添加对 test 项目的依赖以及对第二个版本的 foobar 项目的依赖随后运行 Main 类的 main 函数

无论使用命令行还是 IDE，在手动重现了这一过程后，我想应该已经能够大致理解我的意思，我在此还是展开解释：

当 Test\$1 类被加载的时候，将按照 Test 类编译时所依赖的版本的 Foobar 类的枚举常量的名字为 \$SwitchMap\$Foobar 数组初始化，假如没能找到这个枚举常量，则会抛出`NoSuchFieldError`并在 Test\$1 中被捕获且忽视，于是在 \$SwitchMap\$Foobar 数组的对应位置将要维持初始值 0，因此在 Test 类中将因这个 0 值的存在而走向`default`分支，从而确保在项目所依赖的 SDK 之间的版本匹配错误的情况下，程序还可以保证自己的健壮性，避免因走入未知的分支而威胁业务安全。

## 参考文献

- [Java 语言规范（Java SE 8 版本）](https://docs.oracle.com/javase/specs/jls/se8/html/index.html)
- [Java 虚拟机规范（Java SE 8 版本）](https://docs.oracle.com/javase/specs/jvms/se8/html/index.html)
