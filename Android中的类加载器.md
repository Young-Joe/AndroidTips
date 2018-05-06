## Android中的类加载器

基于jvm的java应用是通过ClassLoader来加载应用中的class的,而Android使用的是dalvik/art,且class文件会被打包进一个dex文件中,底层虚拟机不同,那么它们的类加载器也不同.在Android中,要加载dex文件中的class文件就需要用PathClassLoader或DexClassLoader这两个Android专用的类加载器.

- PathClassLoader与DexClassLoader都继承于BaseDexClassLoader.
- PathClassLoader与DexClassLoader都调用了父类的构造函数,但DexClassLoader多传了一个optimizedDirectory.
- PathClassLoader:只能加载已经安装到Android系统中的APK文件(/data/app目录),是Android默认使用的类加载器.
- DexClassLoader:可以加载任意目录下的dex/jar/apk/zip文件,比PathClassLoader更灵活,是实现热修复的重点.

![](http://mmbiz.qpic.cn/mmbiz_png/CvQa8Yf8vq1bx8VextvLyA0FwntMh0dDePUcG8sKLyEkRqibPZoZBUJpkspg6eL96D53URfOLaQib2fPVIG8yuOQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1)

##### 获取class

类加载器会提供一个方法来供外界找到它所加载的class,该方法就是findCLass().

BaseDexClassLoader已重写该方法.

在findClass方法中通过创建DexPathList对象,并调用其findClass()方法来获取class的.

![img](http://mmbiz.qpic.cn/mmbiz_png/CvQa8Yf8vq1bx8VextvLyA0FwntMh0dD4ibUMs6eW9aKVWynh5oKgAsbPO1H9tt1KRWHfTycvxM5m6S8JCsFbiag/640?wx_fmt=png&wxfrom=5&wx_lazy=1)



`@Retention` 保留时间：有三类值可以选择，`SOURCE`  `RUNTIME` `CLASS`。

`SOURCE` 源码时保留：使用此类的注解多为标记注解，比如 `@Override`、`@Deprecated`、`@SuppressWarnings` 等。

`RUNTIME` 运行时保留：程序在运行过程中，使用这些 Annotation, 比如我们常用的 `@Test`。

`CLASS` 编译时保留：Java 文件在编译时由 apt 自动解析，需要自定义类继承自 AbstractProcessor 并重写 Process 函数。比如 ButterKnife 中使用的  `@BindView`, `@OnClick` 等就是声明为 CLASS 的。

运行时注解
通常被定义的注解需要通过反射来获取相关值
编译时注解
在代码构建编译过程的时候,生成java文件然后供需要的类进行调用

两者根本区别在于,前者是程序员预先写好的java文件中,直接调用的,
而后者是程序员写好java代码的生成规则,程序员自己不写java文件,
交给编译器去写java文件,,java文件只有编译器编译完成后才能调用.