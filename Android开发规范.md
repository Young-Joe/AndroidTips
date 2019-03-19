# 		Android开发规范

1. Activity 间通过隐式 Intent 的跳转,在发出 Intent 之前必须通过 resolveActivity
   检查,避免找不到合适的调用组件,造成 ActivityNotFoundException 的异常。

   eg:

   ```java
   public void viewUrl(String url, String mimeType) {
   	Intent intent = new Intent(Intent.ACTION_VIEW);
   	intent.setDataAndType(Uri.parse(url), mimeType);
   	if (getPackageManager().resolveActivity(intent, 					PackageManager.MATCH_DEFAULT_ ONLY) != null) {
   			try { 
                 	startActivity(intent);
   			} catch (ActivityNotFoundException e) { 
                 if (Config.LOGD) {
   				Log.d(LOGTAG, "activity not found for " + mimeType 					+ " over " + Uri.parse(url). getScheme(), e);
   			}
            }
   	} 
   }						
   ```

2. 避免在 Service#onStartCommand()/onBind()方法中执行耗时操作,如果确
   实有需求,应改用 IntentService 或采用其他异步机制完成。

3. 避免在 BroadcastReceiver#onReceive()中执行耗时操作,如果有耗时工作,应该创建 IntentService 完成,而不应该在 BroadcastReceiver 内创建子线程去做。

   说明:由于该方法是在主线程执行,如果执行耗时操作会导致 UI 不流畅。可以使用IntentService 、 创 建 HandlerThread 或 者 调 用Context#registerReceiver(BroadcastReceiver, IntentFilter, String, Handler)方法等方式,在其他 Wroker 线程执行 onReceive 方法。BroadcastReceiver#onReceive()方法耗时超过 10 秒钟,可能会被系统杀死。

4. 避免使用隐式 Intent 广播敏感信息,信息可能被其他注册了对应BroadcastReceiver 的 App 接收。

   说明:通过 Context#sendBroadcast()发送的隐式广播会被所有感兴趣的 receiver 接收,恶意应用注册监听该广播的 receiver 可能会获取到 Intent 中传递的敏感信息,并进行其他危险操作。如果发送的广播为使用 Context#sendOrderedBroadcast()方法发送的有序广播,优先级较高的恶意 receiver 可能直接丢弃该广播,造成服务不可用,或者向广播结果塞入恶意数据。

   如果广播仅限于应用内,则可以使用 LocalBroadcastManager#sendBroadcast()实现,避免敏感信息外泄和 Intent 拦截的风险

   ```
    Intent intent = new Intent("my-sensitive-event");   intent.putExtra("event", "this is a test event");
   ```

   ​

5. 应用间共享文件时,不要通过放宽文件系统权限的方式去实现,而应使用
   FileProvider。

   ```
   <!-- AndroidManifest.xml -->
   <manifest> ...
   <application> ...
   <provider android:name="android.support.v4.content.FileProvider" android:authorities="com.example.fileprovider"
   android:exported="false" android:grantUriPermissions="true"> <meta-data
   android:name="android.support.FILE_PROVIDER_PATHS"
   android:resource="@xml/provider_paths" /> </provider>
   ... </application>
   </manifest>
   <!-- res/xml/provider_paths.xml --> <paths>
   <files-path path="album/" name="myimages" /> </paths>
   void getAlbumImage(String imagePath) {
   File image = new File(imagePath);
   Intent getAlbumImageIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE); Uri imageUri = FileProvider.getUriForFile(
   this, "com.example.provider", image);
   getAlbumImageIntent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);
   startActivityForResult(takePhotoIntent, REQUEST_GET_ALBUMIMAGE); 
   }
   ```

6. 对于内部使用的组件,显示设置组件的"android:exported"属性为 false。	

7. 应用发布前确保 android:debuggable 属性设置为 false


   ​			
   ​		

   ​