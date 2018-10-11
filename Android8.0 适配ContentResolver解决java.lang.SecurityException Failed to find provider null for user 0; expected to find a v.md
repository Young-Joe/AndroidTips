## Android8.0 适配ContentResolver解决java.lang.SecurityException: Failed to find provider null for user 0; expected to find a valid ContentProvider for this authority

对于没有适配Android8.0+的设备在使用ContenResolver时会出现java.lang.SecurityException: Failed to find provider null for user 0; expected to find a valid ContentProvider for this authority的异常.

原因是在8.0以后,需要使用一个定义的应用内的ContentProvider来防止来自恶意应用的内容变更,以及私密数据泄露.

#### 适配方法

> 1.新建一个XXContentProvider继承自ContentProvider
>
> 2.在AndroidManifest.xml中声明该provider,并指定authority
>
> 3.在使用notifyChange和registerContentObserver时,需要将此前的Uri加上authority

如下:

```
        <provider
            android:authorities="Demo"
            android:name=".dao.offline.DbContentProvider"
            android:enabled="true"
            android:exported="false"/>
```



```
public class DbContentProvider extends ContentProvider {

    private static final String AUTHORITY = "Demo";
    private static final String SCHEME = "content";

    @Override
    public boolean onCreate() {
        return true;
    }

    public static Uri getUri(String dbName) {
        return new Uri.Builder().authority(AUTHORITY)
                .path(AppConstant.getDBPath() + File.separator + dbName)
                .scheme(SCHEME)
                .build();
    }
}
```

使用:

```
context.getContentResolver().registerContentObserver(
                DbContentProvider.getUri("YouDbName"), true, mDBObserver);
```

