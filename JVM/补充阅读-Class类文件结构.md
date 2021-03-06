title: 补充阅读1-Class类文件结构
tag: JVM
---
总体概览一下Class文件是什么以及有什么。
<!-- more -->

## 整体感知

`class`文件是一种8位字节的二进制流文件， 各个数据项按顺序紧密的从前向后排列， 相邻的项之间没有间隙， 这样可以使得`class`文件非常紧凑， 体积轻巧， 可以被JVM快速的加载至内存， 并且占据较少的内存空间。 我们的Java源文件， 在被编译之后， 每个类（或者接口）都单独占据一个`class`文件， 并且类中的所有信息都会在`class`文件中有相应的描述， 由于`class`文件很灵活， 它甚至比Java源文件有着更强的描述能力。

## Class文件格式

![image](http://javajvm.oursnail.cn/Class%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F.png)

**换成表格的形式：**

类型 | 名称 | 数量
---|---|---
u4 | magic | 1
u2 | minor_version | 1
u2 | major_version | 1
u2 | constant_pool_count |	1
cp_info | constant_pool	| constant_pool_count - 1
u2 | access_flags |	1
u2 | this_class | 1
u2 | super_class | 1
u2 | interfaces_count |	1
u2 | interfaces | interfaces_count
u2 | fields_count |	1
field_info | fields | fields_count
u2 | methods_count | 1
method_info | methods |	methods_count
u2 | attribute_count | 1
attribute_info | attributes |attributes_count

### NO1. 魔数(magic)

所有的由Java编译器编译而成的class文件的前4个字节都是“0xCAFEBABE”

它的作用在于：

> 当JVM在尝试加载某个文件到内存中来的时候，会首先判断此class文件有没有JVM认为可以接受的“签名”，即JVM会首先读取文件的前4个字节，判断该4个字节是否是“0xCAFEBABE”，如果是，则JVM会认为可以将此文件当作class文件来加载并使用。

### NO2.版本号(minor_version,major_version)

主版本号和次版本号在class文件中各占两个字节，**副版本号占用第5、6两个字节，而主版本号则占用第7，8两个字节**。JDK1.0的主版本号为45，以后的每个新主版本都会在原先版本的基础上加1。若现在使用的是JDK1.7编译出来的class文件，则相应的主版本号应该是51,对应的7，8个字节的十六进制的值应该是 0x33。

JVM在加载class文件的时候，会读取出主版本号，然后比较这个class文件的主版本号和JVM本身的版本号，如果JVM本身的版本号小于class文件的版本号，JVM会认为加载不了这个class文件，会抛出我们经常见到的`"java.lang.UnsupportedClassVersionError: Bad version number in .class file " Error `错误；反之，JVM会认为可以加载此class文件，继续加载此class文件。


### NO3.常量池计数器(constant_pool_count)

 常量池是class文件中非常重要的结构，它描述着整个class文件的字面量信息。 常量池是由一组`constant_pool`结构体数组组成的，而数组的大小则由常量池计数器指定。常量池计数器`constant_pool_count` 的值等于`constant_pool`表中的成员数+ 1。`constant_pool`表的索引值只有在大于 0 且小于`constant_pool_count`时(即1~(constant_pool_count-1))才会被认为是有效的。
 
 这个容量计数是从1而不是从0开始的，如果常量池容量为十六进制数0x0016，即十进制22，这就代表着常量池中有21个常量，索引值范围为1-21。在Class文件格式规范制定时，设计者将第0项常量空出来是有特殊考虑的，用于在特定情况下表达“不引用任何一个常量池项目”。

### NO4.常量池数据区(constant_pool[contstant_pool_count-1])

常量池，constant_pool是一种表结构,它包含 Class 文件结构及其子结构中引用的所有字符串常量、 类或接口名、字段名和其它常量。 常量池中的每一项都具备相同的格式特征——第一个字节作为类型标记用于识别该项是哪种类型的常量，称为 “tag byte” 。常量池的索引范围是 1 至constant_pool_count−1。常量池的具体细节我们会稍后讨论。

### NO6.访问标志(access_flags)

访问标志，access_flags 是一种掩码标志，用于表示某个类或者接口的访问权限及基础属性。

![image](http://javajvm.oursnail.cn/%E8%AE%BF%E9%97%AE%E6%A0%87%E5%BF%97.png)

### NO7.类索引(this_class)

类索引，this_class的值必须是对constant_pool表中项目的一个有效索引值。constant_pool表在这个索引处的项必须为CONSTANT_Class_info 类型常量，表示这个 Class 文件所定义的类或接口。

### NO8.父类索引(super_class)

父类索引，对于类来说，super_class 的值必须为 0 或者是对constant_pool 表中项目的一个有效索引值。如果它的值不为 0，那 constant_pool 表在这个索引处的项必须为CONSTANT_Class_info 类型常量，表示这个 Class 文件所定义的类的直接父类。当前类的直接父类，以及它所有间接父类的access_flag 中都不能带有ACC_FINAL 标记。对于接口来说，它的Class文件的super_class项的值必须是对constant_pool表中项目的一个有效索引值。constant_pool表在这个索引处的项必须为代表 java.lang.Object 的 CONSTANT_Class_info 类型常量 。如果 Class 文件的 super_class的值为 0，那这个Class文件只可能是定义的是java.lang.Object类，只有它是唯一没有父类的类。

### NO9.接口计数器(interfaces_count)

 接口计数器，interfaces_count的值表示当前类或接口的直接父接口数量。

### NO10.接口信息数据区(interfaces[interfaces_count])

接口表，interfaces[]数组中的每个成员的值必须是一个对constant_pool表中项目的一个有效索引值， 它的长度为 interfaces_count。每个成员 interfaces[i]  必须为 CONSTANT_Class_info类型常量，其中 0 ≤ i <interfaces_count。在interfaces[]数组中，成员所表示的接口顺序和对应的源代码中给定的接口顺序（从左至右）一样，即interfaces[0]对应的是源代码中最左边的接口。

### NO11.字段计数器(fields_count)

字段计数器，fields_count的值表示当前 Class 文件 fields[]数组的成员个数。 fields[]数组中每一项都是一个field_info结构的数据项，它用于表示该类或接口声明的类字段或者实例字段。

### NO12.字段信息数据区(fields[fields_count])

字段表，fields[]数组中的每个成员都必须是一个fields_info结构的数据项，用于表示当前类或接口中某个字段的完整描述。 fields[]数组描述当前类或接口声明的所有字段，但不包括从父类或父接口继承的部分。

### NO13.方法计数器(methods_count)

 方法计数器， methods_count的值表示当前Class 文件 methods[]数组的成员个数。Methods[]数组中每一项都是一个 method_info 结构的数据项。

### NO14.方法信息数据区(methods[methods_count])

方法表，methods[] 数组中的每个成员都必须是一个 method_info 结构的数据项，用于表示当前类或接口中某个方法的完整描述。如果某个method_info 结构的access_flags 项既没有设置 ACC_NATIVE 标志也没有设置ACC_ABSTRACT 标志，那么它所对应的方法体就应当可以被 Java 虚拟机直接从当前类加载，而不需要引用其它类。 method_info结构可以表示类和接口中定义的所有方法，包括实例方法、类方法、实例初始化方法方法和类或接口初始化方法方法 。methods[]数组只描述当前类或接口中声明的方法，不包括从父类或父接口继承的方法。

### NO15.属性计数器(attributes_count)

属性计数器，attributes_count的值表示当前 Class 文件attributes表的成员个数。attributes表中每一项都是一个attribute_info 结构的数据项。

### NO16.属性信息数据区(attributes[attributes_count])

属性表，attributes 表的每个项的值必须是attribute_info结构。

在Java 7 规范里，Class文件结构中的attributes表的项包括下列定义的属性： InnerClasses  、 EnclosingMethod 、 Synthetic  、Signature、SourceFile，SourceDebugExtension 、Deprecated、RuntimeVisibleAnnotations 、RuntimeInvisibleAnnotations以及BootstrapMethods属性。

对于支持 Class 文件格式版本号为 49.0 或更高的 Java 虚拟机实现，必须正确识别并读取attributes表中的Signature、RuntimeVisibleAnnotations和RuntimeInvisibleAnnotations属性。对于支持Class文件格式版本号为 51.0 或更高的 Java 虚拟机实现，必须正确识别并读取 attributes表中的BootstrapMethods属性。Java 7 规范 要求任一 Java 虚拟机实现可以自动忽略 Class 文件的 attributes表中的若干 （甚至全部） 它不可识别的属性项。任何本规范未定义的属性不能影响Class文件的语义，只能提供附加的描述信息 。