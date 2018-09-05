## 			int.Integer区别

1. int:基本数据类型.其数据是存储在栈中

   Integer:继承自Object,必须经过实例化.引用存储在栈中,对象数据存储在堆中

2. int的初始化值是0,Integer初始化值是null

3. Integer是int的包装类

关于Integer和int的比较 
1、由于Integer变量实际上是对一个Integer对象的引用，所以两个通过new生成的Integer变量永远是不相等的（因为new生成的是两个对象，其内存地址不同）。

```
Integer i = new Integer(100);
Integer j = new Integer(100);
System.out.print(i == j); //false
```

2、Integer变量和int变量比较时，只要两个变量的值是向等的，则结果为true（因为包装类Integer和基本数据类型int比较时，java会自动拆包装为int，然后进行比较，实际上就变为两个int变量的比较）

```
Integer i = new Integer(100);
int j = 100；
System.out.print(i == j); //true
```

3、非new生成的Integer变量和new Integer()生成的变量比较时，结果为false。（因为非new生成的Integer变量指向的是java常量池中的对象，而new Integer()生成的变量指向堆中新建的对象，两者在内存中的地址不同）

```
Integer i = new Integer(100);
Integer j = 100;
System.out.print(i == j); //false
```

4、对于两个非new生成的Integer对象，进行比较时，如果两个变量的值在区间-128到127之间，则比较结果为true，如果两个变量的值不在此区间，则比较结果为false

```
Integer i = 100;
Integer j = 100;
System.out.print(i == j); //true
```

```
Integer i = 128;
Integer j = 128;
System.out.print(i == j); //false
```

对于第4条的原因： 
java在编译Integer i = 100 ;时，会翻译成为Integer i = Integer.valueOf(100)；，而java API中对Integer类型的valueOf的定义如下：

```
public static Integer valueOf(int i){
    assert IntegerCache.high >= 127;
    if (i >= IntegerCache.low && i <= IntegerCache.high){
        return IntegerCache.cache[i + (-IntegerCache.low)];
    }
    return new Integer(i);
}
```

java对于-128到127之间的数，会进行缓存，Integer i = 127时，会将127进行缓存，下次再写Integer j = 127时，就会直接从缓存中取，就不会new了