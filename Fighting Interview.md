##					Fighting Interview

#####onStart和onResume区别

onStart只是表示进程处于前台,但并不能进行交互.

#####Activity启动流程

startActivity最终会调用startActivityForResult,通过ActivityManagerProxy调用systemServer进程中ActivityManagerService的startActivity方法,如果需要启动的Activity所在进程未启动,则调用Zygote孵化应用进程,进程创建后调用应用的ActivityThread的main方法,main方法调用attach方法将应用进程绑定到ActivityManagerService(保存应用的ApplicationThread的代理对象)并开启loop循环接受消息.ActivityManagerService通过ApplicationThread的代理发送Message通知启动Activity,ActivityThread内部Handler处理handleLaunchActivity,依次调用performLaunchActivity,handleResumeActivity(即activity的onCreate,onStart...)

#####requestLayout，invalidate，postInvalidate区别与联系

相同点：三个方法都有刷新界面的效果。
不同点：invalidate和postInvalidate只会调用onDraw()方法；requestLayout则会重新调用onMeasure、onLayout、不一定会调用onDraw。一般说来需要重新布局就调用requestLayout()方法，需要重新绘制就调用invalidate()方法。

### Binder机制，共享内存实现原理

**为什么使用Binder？**

![img](https://upload-images.jianshu.io/upload_images/4657803-b596110487d1ef17.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/720)

**概念**
进程隔离
进程空间划分：用户空间(User Space)/内核空间(Kernel Space)
系统调用：用户态与内核态

**原理**
跨进程通信是需要内核空间做支持的。传统的 IPC 机制如管道、Socket 都是内核的一部分，因此通过内核支持来实现进程间通信自然是没问题的。但是 Binder 并不是 Linux 系统内核的一部分，那怎么办呢？这就得益于 Linux 的动态内核可加载模块（Loadable Kernel Module，LKM）的机制；模块是具有独立功能的程序，它可以被单独编译，但是不能独立运行。它在运行时被链接到内核作为内核的一部分运行。这样，Android 系统就可以通过动态添加一个内核模块运行在内核空间，用户进程之间通过这个内核模块作为桥梁来实现通信。

> 在 Android 系统中，这个运行在内核空间，负责各个用户进程通过 Binder 实现通信的内核模块就叫 Binder 驱动（Binder Dirver）。

那么在 Android 系统中用户进程之间是如何通过这个内核模块（Binder 驱动）来实现通信的呢？难道是和前面说的传统 IPC 机制一样，先将数据从发送方进程拷贝到内核缓存区，然后再将数据从内核缓存区拷贝到接收方进程，通过两次拷贝来实现吗？显然不是，否则也不会有开篇所说的 Binder 在性能方面的优势了。

这就不得不通道 Linux 下的另一个概念：**内存映射**。

Binder IPC 机制中涉及到的内存映射通过 mmap() 来实现，mmap() 是操作系统中一种内存映射的方法。内存映射简单的讲就是将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间。
**一次完整的 Binder IPC 通信过程通常是这样：**

1. 首先 Binder 驱动在内核空间创建一个数据接收缓存区；
2. 接着在内核空间开辟一块内核缓存区，建立内核缓存区和内核中数据接收缓存区之间的映射关系，以及内核中数据接收缓存区和接收进程用户空间地址的映射关系；
3. 发送方进程通过系统调用 copyfromuser() 将数据 copy 到内核中的内核缓存区，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。

**Binder通讯模型**
Binder是基于C/S架构的，其中定义了4个角色：Client、Server、Binder驱动和ServiceManager。

- Binder驱动：类似网络通信中的路由器，负责将Client的请求转发到具体的Server中执行，并将Server返回的数据传回给Client。
- ServiceManager：类似网络通信中的DNS服务器，负责将Client请求的Binder描述符转化为具体的Server地址，以便Binder驱动能够转发给具体的Server。Server如需提供Binder服务，需要向ServiceManager注册。
  **具体的通讯过程**

1. Server向ServiceManager注册。Server通过Binder驱动向ServiceManager注册，声明可以对外提供服务。ServiceManager中会保留一份映射表。
2. Client向ServiceManager请求Server的Binder引用。Client想要请求Server的数据时，需要先通过Binder驱动向ServiceManager请求Server的Binder引用（代理对象）。
3. 向具体的Server发送请求。Client拿到这个Binder代理对象后，就可以通过Binder驱动和Server进行通信了。
4. Server返回结果。Server响应请求后，需要再次通过Binder驱动将结果返回给Client。

**ServiceManager是一个单独的进程，那么Server与ServiceManager通讯是靠什么呢？**
当Android系统启动后，会创建一个名称为servicemanager的进程，这个进程通过一个约定的命令BINDERSETCONTEXT_MGR向Binder驱动注册，申请成为为ServiceManager，Binder驱动会自动为ServiceManager创建一个Binder实体。并且这个Binder实体的引用在所有的Client中都为0，也就说各个Client通过这个0号引用就可以和ServiceManager进行通信。Server通过0号引用向ServiceManager进行注册，Client通过0号引用就可以获取到要通信的Server的Binder引用。

> 系统启动时init进程会创建Zygote进程,Zygote通过FORK子进程进行其他进程的创建和启动
>
> Zygote进程创建的第一个是SystemServer(负责启动系统的关键服务,如PackageManagerService.ActivityManagerService).
>
> 当需要启动一个应用时,AMS会通过socket进程间通信机制,通知Zygote进程.

##### RecyclerView与ListView(缓存原理，区别联系，优缺点)

**缓存区别：**

1. 层级不同：
   ListView有两级缓存，在屏幕与非屏幕内。
   RecyclerView比ListView多两级缓存，支持多个离屏ItemView缓存（匹配pos获取目标位置的缓存，如果匹配则无需再次bindView），支持开发者自定义缓存处理逻辑，支持所有RecyclerView共用同一个RecyclerViewPool(缓存池)。
2. 缓存不同：
   ListView缓存View。
   RecyclerView缓存RecyclerView.ViewHolder，抽象可理解为：
   View + ViewHolder(避免每次createView时调用findViewById) + flag(标识状态)；

**优点**
RecylerView提供了局部刷新的接口，通过局部刷新，就能避免调用许多无用的bindView。
RecyclerView的扩展性更强大（LayoutManager、ItemDecoration等）。

 ##### 垃圾收集算法

1. 标记-清除算法（Mark-Sweep）
   在标记阶段，确定所有要回收的对象，并做标记。清除阶段紧随标记阶段，将标记阶段确定不可用的对象清除。标记—清除算法是基础的收集算法，有两个不足：1）标记和清除阶段的效率不高；2）清除后回产生大量的不连续空间，这样当程序需要分配大内存对象时，可能无法找到足够的连续空间。

2. 复制算法（Copying）
   复制算法是把内存分成大小相等的两块，每次使用其中一块，当垃圾回收的时候，把存活的对象复制到另一块上，然后把这块内存整个清理掉。复制算法实现简单，运行效率高，但是由于每次只能使用其中的一半，造成内存的利用率不高。现在的JVM 用复制方法收集新生代，由于新生代中大部分对象（98%）都是朝生夕死的，所以会分成1块大内存Eden和两块小内存Survivor(大概是8:1:1)，每次使用1块大内存和1块小内存，当回收时将2块内存中存活的对象赋值到另一块小内存中，然后清理剩下的。

3. 标记—整理算法（Mark-Compact）
   标记—整理算法和复制算法一样，但是标记—整理算法不是把存活对象复制到另一块内存，而是把存活对象往内存的一端移动，然后直接回收边界以外的内存。标记—整理算法提高了内存的利用率，并且它适合在收集对象存活时间较长的老年代。

4. 分代收集（Generational Collection）
   分代收集是根据对象的存活时间把内存分为新生代和老年代，根据各代对象的存活特点，每个代采用不同的垃圾回收算法。新生代采用复制算法，老年代采用标记—整理算法。

   Java垃圾回收机制最基本的做法是分代回收。内存中的区域被划分成不同的世代，对象根据其存活的时间被保存在对应世代的区域中。一般的实现是划分成3个世代：年轻、年老和永久。内存的分配是发生在年轻世代中的。当一个对象存活时间足够长的时候，它就会被复制到年老世代中。对于不同的世代可以使用不同的垃圾回收算法。进行世代划分的出发点是对应用中对象存活时间进行研究之后得出的统计规律。一般来说，一个应用中的大部分对象的存活时间都很短。比如局部变量的存活时间就只在方法的执行过程中。基于这一点，对于年轻世代的垃圾回收算法就可以很有针对性。

   > 新生代:minor gc 代用标记整理算法
   >
   > 老年代:major gc 或full gc. System.gc()强制执行的就是full gc

##### Dalvik VM和JVM的差异

- JVM运行的是java字节码.Dalvik运行的是dex文件

- **jvm:基于栈.**java类会被编译成一个或多个字节码文件(.class)然后打包到jar.然后再从相应的class文件中获取相应的字节码.

  **Dalvik基于寄存器**则是在编译成class文件后,通过dx将所有的class文件转换成dex文件(减少整体的文件尺寸和I/O操作,提高了类查找)

##### Dalvik和ART的差异

Dalvik:在第一次加载后会生成cache文件,以提供下次快速加载,所以第一次很慢.

ART:在安装应用的时候会进行一次预编译,在安装应用时会将代码转换成**机器码**存储在本地(dex2oat),这样在运行时就不需每次都进行一次编译了.

##### HashMap的实现原理

- HashMap概述：HashMap是基于哈希表的Map接口的非同步实现。此实现提供所有可选的映射操作，并允许使用null值和null键。此类不保证映射的顺序，特别是它不保证该顺序恒久不变。
- HashMap的数据结构：在java编程语言中，最基本的结构就是两种，一个是数组，另外一个是模拟指针（引用），所有的数据结构都可以用这两个基本结构来构造的，HashMap也不例外。HashMap实际上是一个“链表散列”的数据结构，即数组和链表的结合体。
- HashMap底层就是一个数组结构，数组中的每一项又是一个链表。当新建一个HashMap的时候，就会初始化一个数组。

​       [容量/负载因子/put/get](https://github.com/LRH1993/android_interview/blob/master/java/basis/hashmap.md)

- hashmap是否线程安全?不安全会出什么问题?

  HashMap在put的时候，插入的元素超过了容量（由负载因子决定）的范围就会触发扩容操作，就是rehash，这个会重新将原数组的内容重新hash到新的扩容数组中，在多线程的环境下，存在同时其他的元素也在进行put操作，如果hash值相同，可能出现同时在同一数组下用链表表示，造成闭环，导致在get时会出现死循环，所以HashMap是线程不安全的。

 [线程池执行流程](https://github.com/LRH1993/android_interview/blob/master/java/concurrence/thread-pool.md)

 [Activity就像个控制器，不负责视图部分。Window像个承载器，装着内部视图。DecorView就是个顶层视图，是所有View的最外层布局。ViewRoot像个连接器，负责沟通，通过硬件的感知来通知视图，进行用户之间的交互。](https://github.com/LRH1993/android_interview/blob/master/android/basis/decorview.md)

##### dp.dip.sp

dp:是像素密度 dip:设备无关像素,(后来为了与sp统一使用了dp).sp:与缩放无关的抽象像素

#####[LinearLayout.FrameLayout.RelativeLayout性能对比,为什么](https://blog.csdn.net/hejjunlin/article/details/51159419)

- 表现层：用Espresso 2和Android Instrumentation测试框架测试UI。
- 领域层：JUnit + Mockito —— 它是Java的标准模块。
- 数据层：将测试组合换成了Robolectric 3 + JUnit + Mockito

##### SP的commit()和apply()的异同

commit是直接同步的提交到磁盘.apply是先写入内存,然后**异步**的写入磁盘.

##### MD5是加密方法么,Base64

MD5是一种哈希算法,(把任意数据转换为定长数据的算法统称).也可以理解为不可逆加密

##### Fragment add和replace的区别

add:覆盖原fragment, 添加入一个新fragment后, 原来的fragment仍然存活

replace:先remove掉相同id的所有fragment，然后在add当前的这个fragment

##### **Fragment的启动栈和回退栈**

加入回退栈:addToBackStack(null)   弹出回退栈:popBackStack()

[**热修复技术**](https://www.cnblogs.com/popfisher/p/8543973.html)

**对于 Animation 动画:**

他的实现机制是,在每次进行绘图的时候,通过对整块画布的矩阵进行变换,从而实现一种视图坐标的移动,但实际上其在 View 内部真实的坐标位置及其他相关属性始终恒定.

**对于 Animator 动画:**

Animator 动画的实现机制说起来其实更加简单一点,因为他其实只是计算动画开启之后,结束之前,到某个时间点得时候,某个属性应该有的值,然后通过回调接口去设置具体值,其实 Animator 内部并没有针对某个 view 进行刷新,来实现动画的行为,动画的实现是在设置具体值的时候,方法内部自行调取的类似 invalidate 之类的方法实现的.也就是说,使用 Animator ,内部的属性发生了变化.

- 补间动画click事件还在原位怎么解决?

  如果一个View如果在做补间动画中的平移、旋转、缩放动画，那么它的点击事件一定要进行的矩阵处理。

  在ParentView中重写onTouchEvent(MotionEvent event)，拦截点击点击事件的x、y坐标。

#####[在java中==和equals()的区别](https://blog.csdn.net/lcsy000/article/details/82782864)

[hashcode 和 equals作用](https://www.cnblogs.com/keyi/p/7119825.html)

[Java并发问题--乐观锁与悲观锁以及乐观锁的一种实现方式-CAS](https://www.cnblogs.com/qjjazry/p/6581568.html)

#####导致内存泄露常见原因:

1)静态变量直接或者间接地引用了Activity对象就会造成内存泄露

2)Activity使用了静态的View(View会持有Activity的对象的引用)

3)Activity定义了静态View变量???

4)ImageSpan引用了Activity Context

5)单例中引用了Activity的Context(需要使用Application的Context)

6)对于使用了BraodcastReceiver，ContentObserver，File，Cursor，Stream，Bitmap等资源，应该在Activity销毁时及时关闭或者注销，否则这些资源将不会被回收，从而造成内存泄漏。

7)静态集合保存的对象没有及时消除(不使用的时候置为null)

8)在Java中,非静态(匿名)内部类会引用外部类对象,而静态内部类不会引用外部类对象

9)在Activity中,创建了非静态内部类(内部类直接或者间接引用了Activity)的静态成员变量

10)线程包括AsyncTask的使用,Activity退出后线程还在运行(线程在死循环),并且在线程中使用了Activity或view对象(解决方法:不要直接写死循环,可以设置一个布尔类型的TAG,当activity推出的时候,设置TAG为False)

11)Handler对象的使用,Activity退出后Handler还是有消息需要处理(解决方法:在退出activity之后,移除消息)

12)WebView造成的内存泄漏(在onDestory中销毁)

[JIT/AOP+解释运行](https://mp.weixin.qq.com/s/q3uxdoENJM_jt7QxuE1IrA)

[Android APP性能优化的四个方面最全总结](https://blog.csdn.net/zhangbijun1230/article/details/79449725#commentBox)

##### Serializable Parcelable

Serializable:java api.通过I/O读写存储磁盘的,使用反射来实现.容易触发GC.可以网络传输/本地存储

Parcelable:直接在内存中读写.无法网络传输,无法本地存储