# AndroidTips
some trivial android knowledge...

##### mDrawable.draw(canvas)使用;

```java
Drawable mDrawable = getResources().getDrawable(R.drawable.music);
//获取drawable图像的真实宽高
int dHeight = mDrawable.getIntrinsicHeight();
int dWidth = mDrawable.getintrinsicWidth();
//在调用mDrawable.draw()方法前,必须要对mDrawable进行setBounds()
mDrawable.setBounds(0, 0, dWidth, dHeight);
//将mDrawable的图像信息画到画布上
mDrawable.draw(canvas);
```



| ldpi | mdpi | hdpi | xhdpi | xxhdpi |
| :--: | :--: | :--: | :---: | :----: |
| 120  | 160  | 240  |  320  |  480   |



##### Camera和Matrix简单使用

> init();

```java
Camera mCamera = new Camera();
Matrix mMatrix = new Matrix();
//设置camera的Z轴投射坐标,避免出现呼脸效果
mCamera.setLocation(0, 0, 100);
```

> onDraw();

```java
//使用Matrix
mCamera.save();
mMatrix.reset();
mCamera.rotateX(degree);
mCamera.getMatrix(mMatrix);
mCamera.restore();

mMatrix.preTranslate(-centerX, -centerY);
mMatrix.postTranslate(centerX, centerY);
mCanvas.concat(mMatrix);

mCanvas.drawXXX();
mCanvas.restore();
```

```java
//直接使用canvas.translate
mCamera.save();
mCamera.retateX(degree);
mCanvas.translate(centerX, centerY);
mCamera.applyToCanvas(canvas);
mCanvas.translate(-centerX, -centerY);
mCamera.restore();

mCanvas.drawXXX();
mCanvas.restore();
```



##### HandlerThread简单使用

> HandlerThread其实是内部建立了Looper的Thread.(新建线程是没有开启消息循环的)
>
> `注意:HandlerThread仍为普通线程,串行执行任务`

```java
private HandlerThread mHandlerThread;
private Handler mHander;

onCreate(){
    ...
    //创建HandlerThread,并将线程命名为:handlerThread
    mHandlerThread = new HandlerThread("handlerThread");
  	mHandlerThread.start();
  	mHandler = new Handler(mHandlerThread.getLooper){
        @Overtide
      	public void handleMessage(Message msg){
            super.handleMessage(msg);
          	//该方法运行在HandlerThread线程中,可执行耗时操作
          	Log.e("Msg:", msg.what + "线程 : " + Thread.currentThread().getName());
        }
    }
  
  	mHandler.sendEmptyMessage(0);
  	//使用post同样可以将run()操作运行在HandlerThread中
  	mHandler.post(new Runnable() {
				@Override
				public void run() {
					Log.e("Msg:", 1 + "线程 : " + Thread.currentThread().getName());
				}
			});
  
}

```

```
Log:
Msg:0 线程 : handlerThread
Msg:1 线程 : handlerThread
```



##### 权限

```java
权限验证
ContextCompat.checkSelfPermission(context, Manifest.permission.XXX) != PackageManager.PERMISSION_GRANTED

权限申请
ActivityCompat.requsetPermissions(context, permissions, 1);

权限请求结果
void onRequestPermissionsResult(int requestCode; Array permissions; intArray grantResults){};
```

```
// 设置PopupWindow在软键盘的上方
popupWindow.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE)
```


##### 集合

> ArrayList-数组实现,但数组容量有限.当超出限制容量时会增加50%容量,用System.arraycopy()复制到新的数组.默认为大小为10的数组

>LinkedList-双向链表实现.顺序访问高效,随机访问低效
>
>使用get(i)/set(i)时,指针 i > 数组/2 会从末尾移动 

>LinkedHashMap 双向循环链表,保留节点插入顺序.从而可以在遍历时按顺序显示. 非线程安全,只在单线程环境下使用



##### volatile关键字

java提供了volatile关键字来保证可见性` 并发编程三大概念:原子性,有序性,可见性` 

当一个共享变量被volatile修饰时,它会保证修改的值会立即被更新到内存,当有其他线程需要读取时,它会去内存中读取新值.

而普通的共享变量不能保证可见性,因为普通共享变量被修改后,什么时候被写入内存是不确定的,当其他线程去读取时,此时内存中可能还是原来的旧值,因此无法保证可见性.

另外,通过synchronize和Lock也能保证可见性,synchronize和Lock能保证同一时刻只有一个线程获取锁然后执行同步代码,并在释放锁之前将对变量的修改刷新到主存中.



##### RandomAccessFile

RandomAccessFile即可读取文件内容,也可以向文件输出数据.同时,RandomAccessFile支持'随机访问',可以直接跳转到文件的任意地方来读写数据

**RandomAccessFile的一个重要使用场景就是网络请求中的多线程下载及断点续传**

##### getApplication和getApplicationContext

Application本身就是一个Context，所以这里获取getApplicationContext()得到的结果就是Application本身的实例。那么问题来了，既然这两个方法得到的结果都是相同的，那么Android为什么要提供两个功能重复的方法呢？实际上这两个方法在作用域上有比较大的区别。getApplication()方法的语义性非常强，一看就知道是用来获取Application实例的，**但是这个方法只有在Activity和Service中才能调用的到**。那么也许在绝大多数情况下我们都是在Activity或者Service中使用Application的，但是如果在一些其它的场景，比如BroadcastReceiver中也想获得Application的实例，这时就可以借助getApplicationContext()方法了。