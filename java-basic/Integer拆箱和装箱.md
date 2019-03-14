title: Integer拆箱和装箱
tag: java基础
---

由于笔试经常遇到，所以这里整理一下。将Integer这一块一网打尽。
<!-- more -->

## 拆箱和装箱

这里以面试笔试经常出现的`Integer`类型为例，请看下面的代码：
```java
public static void main(String[] args) {
    /*第一组*/
    Integer i = new Integer(128);
    Integer i2 = 128;
    System.out.println(i == i2);

    Integer i3 = new Integer(127);
    Integer i4 = 127;
    System.out.println(i3 == i4);

    /*第二组*/
    Integer i5 = 128;
    Integer i6 = 128;
    System.out.println(i5 == i6);

    Integer i7 = 127;
    Integer i8 = 127;
    System.out.println(i7 == i8);

    /*第三组*/
    Integer i9 = new Integer(128);
    int i10 = 128;
    System.out.println(i9 == i10);

    Integer i11 = new Integer(127);
    int i12 = 127;
    System.out.println(i11== i12);

    /*第四组*/
    Integer i13 = new Integer(128);
    Integer i14 = Integer.valueOf(128);
    System.out.println(i13 == i14);

    Integer i15 = new Integer(127);
    Integer i16 = Integer.valueOf(127);
    System.out.println(i13 == i14);

    /*第五组*/
    Integer i17 = Integer.valueOf(128);
    Integer i18 = 128;
    System.out.println(i17 == i18);

    Integer i19 = Integer.valueOf(127);
    Integer i20 = 127;
    System.out.println(i19 == i20);

}
```


执行结果为：


```
false
false

false
true

true
true

false
false

false
true
```
翻开源码(jdk8)，我们可以看到一个私有的静态类，叫做整形缓存。顾名思义，就是缓存某些整型值，我们可以看到，它默认将-128-127之间数字封装成对象，放进一个常量池中，以后定义类似于`Integer a = 1`里面的`a`就可以直接从这个常量池中取对象即可，不需要重新`new`

```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```
对于`Integer.valueOf()`:


```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

我们可以看到，也是先看看是不是在-128到127之间的范围，是的话就从 `cache` 中取出相应的 `Integer` 对象即可。

- 第一组中
    - 第一种情况，`i` 是创建的一个`Integer`的对象，取值是128。`i2` 是进行自动装箱的实例，因为这里超出了-128到127的范围，所以是创建了新的`Integer`对象。由于`==`比较的是地址，所以两者必然不一样。
    - 第二种情况就不一样了，`i4`是不需要自己`new`，而是可以直接从缓存中取，但是`i3`是`new`出来的，地址还是不一样。
- 第二组中
    - 第一种情况是都超出范围了，所以都要自己分别去`new`，所以不一样
    - 第二种情况是在范围内，都去缓存中取，实际上都指向同一个对象，所以一样
- 第三组中
    - `i10`和`i12`都是`int`型，`i9`和`i11`与它们比较的时候都要自动拆箱，所以比较的是数值，所以都一样
- 第四组中
    - 与第一组原理一样
- 四五组中
    - 与第二组原理一样


所以啊，`new Integer()`是每次都直接`new`对象出来，而`Integer.valueOf()`可能会用到缓存，所以后者效率高一点。


## 总结

- `int` 和 `Integer` 在进行比较的时候， `Integer` 会进行拆箱，转为 `int 值与 `int` 进行比较。
- `Integer` 与 `Integer` 比较的时候，由于直接赋值的时候会进行自动的装箱，那么这里就需要注意两个问题，一个是 `-128<= x<=127` 的整数，将会直接缓存在 `IntegerCache` 中，那么当赋值在这个区间的时候，不会创建新的 `Integer` 对象，而是从缓存中获取已经创建好的 `Integer` 对象。二：当大于这个范围的时候，直接 `new Integer` 来创建 `Integer` 对象。
- `new Integer(1)` 和 `Integer a = 1` 不同，前者会创建对象，存储在堆中，而后者因为在-128到127的范围内，不会创建新的对象，而是从 `IntegerCache` 中获取的。那么 `Integer a = 128`, 大于该范围的话才会直接通过 `new Integer(128)`创建对象，进行装箱。
