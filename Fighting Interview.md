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

##### HashMap的实现原理

- HashMap概述：HashMap是基于哈希表的Map接口的非同步实现。此实现提供所有可选的映射操作，并允许使用null值和null键。此类不保证映射的顺序，特别是它不保证该顺序恒久不变。
- HashMap的数据结构：在java编程语言中，最基本的结构就是两种，一个是数组，另外一个是模拟指针（引用），所有的数据结构都可以用这两个基本结构来构造的，HashMap也不例外。HashMap实际上是一个“链表散列”的数据结构，即数组和链表的结合体。
- HashMap底层就是一个数组结构，数组中的每一项又是一个链表。当新建一个HashMap的时候，就会初始化一个数组。

​       [容量/负载因子/put/get](https://github.com/LRH1993/android_interview/blob/master/java/basis/hashmap.md)

 [线程池执行流程](https://github.com/LRH1993/android_interview/blob/master/java/concurrence/thread-pool.md)

 [Activity就像个控制器，不负责视图部分。Window像个承载器，装着内部视图。DecorView就是个顶层视图，是所有View的最外层布局。ViewRoot像个连接器，负责沟通，通过硬件的感知来通知视图，进行用户之间的交互。](https://github.com/LRH1993/android_interview/blob/master/android/basis/decorview.md)

 

 

 

 

 

 

 