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

