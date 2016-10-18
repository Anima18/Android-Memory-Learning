

## Android 内存泄漏场景

##### 1. 非静态内部类的静态实例

由于内部类默认持有外部类的引用，而静态实例属于类。所以，当外部类被销毁时，内部类仍然持有外部类的引用，致使外部类无法被GC回收。因此造成内存泄露。
    
```
private static TestResource mResource = null;
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_second);
    mResource = new TestResource();
}
class TestResource {
}
```
**正确做法**

内部类声明为static，则该内部类不持有外部Acitivity的引用，则不会阻塞Activity对象的释放。


##### 2. Handler对象

Handler泄露的关键点有两个：
    1). 内部类
    2). 生命周期和Activity不一定一致
- 内部类

    Handler使用的比较多，经常需要在Activity中创建内部类，所以这种场景还是很多的。

```
static class MyHandler extends Handler {

    private WeakReference<HandlerLeakActivity > activityWeakReference;
    public MyHandler(HandlerLeakActivity activity) {
        activityWeakReference= new WeakReference<HandlerLeakActivity >(activity);
    }

    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);
        HandlerLeakActivity activity = activityWeakReference.get();
        if(activity != null) {
            activity.setTextView(msg.getData().getString("Message"));
        }
    }
}
```

- 生命周期和Activity不一定一致

    其实不单指内部类，而是所有Handler对象，如何解决上面说的Handler对象有Message在排队，而不阻塞Activity对象释放
    
```
public void onDestroy() {
    //  If null, all callbacks and messages will be removed.
    mHandler.removeCallbacksAndMessages(null);
}
```


##### 3. AsyncTask对象

AsyncTask泄漏的原理和Handler差不多  
1). 内部类  
2). 生命周期和Activity不一定一致  
第一点的做法和Handler一样。  
第二点，在activity退出的时候，终止AsyncTask中的后台任务。 
```
@Override
protected void onDestroy() {
    super.onDestroy();
    downloadFilesTask.cancel(true);
}
```
但要注意一点，downloadFilesTask.cancel(true)，不一定生效；最好在doInBackground方法检查if (isCancelled()) break;

##### 4. BroadcastReceiver对象

动态注册，没有调用到unregister()方法。

##### 5. 单例模式
```
public class Singleton {
    private static Singleton instance;
    private Context mContext;
    private Singleton(Context mContext){
        this.mContext = mContext;
    }
    public static Singleton getInstance(Context context){
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(context);
                }
            }
        }
        return instance;
    }
}
```
单例模式有个Context字段，如果Context参数为Activity时，当退出该Activity时，单例对象依然持有该Activity的引用，致使Activity无法被GC回收。
正确的做法是，参数为context.getApplicationContext()。

##### 6. 资源对象没关闭造成的内存泄漏



## Android 内存泄漏监听
##### 1. Memory Analyzer tool(MAT)
MAT(Memory Analyzer Tool)，一个基于Eclipse的内存分析工具，是一个快速、功能丰富的JAVA heap分析工具，它可以帮助我们查找内存泄漏和减少内存消耗。使用内存分析工具从众多的对象中进行分析，快速的计算出在内存中对象的占用大小，看看是谁阻止了垃圾收集器的回收工作，并可以通过报表直观的查看到可能造成这种结果的对象。  
使用步骤：  
- **下载Memory Analyzer tool**  

    下载地址[http://www.eclipse.org/mat/downloads.php](http://www.eclipse.org/mat/downloads.php)

- **编写案例**  
    在这里做一个非静态内部类的静态实例，由于内部类默认持有外部类的引用，而静态实例属于类。所以，当外部类被销毁时，内部类仍然持有外部类的引用，致使外部类无法被GC回收。因此造成内存泄露。
    ```
    private static TestResource mResource = null;
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        mResource = new TestResource();
    }
    class TestResource {
    }
    ```

- **导出hprof文件**  
    ![image](http://img.blog.csdn.net/20161011152636444)  
    在Android Studio的Android Monitor中点击Dump Java Heap，会生成内存快照hprof 文件，这个文件保存当前应用的对象的内存使用情况，在后面会MAT分析这个文件。
- 转换hprof格式  
    如果用MAT直接打开第二步导出的hprof文件，MAT报错： Unknown HPROF Version (JAVA PROFILE 1.0.3) (java.io.IOException)。  
    原因是android 刚刚生成的 .hprof 文件在这里需要进行转换一下格式。
打开命令行窗口，在android sdk tools 目录，执行以下命令：
hprof-conv  1.hprof   2.hprof, 然后再次打开MAT程序，打开2.hprof文件，就可以看到正确的分析界面


- **分析hprof文件**  
打开MAT，选择转化好的HPROF文件，可以看到Overview的界面如下图：
![image](http://img.blog.csdn.net/20161011152706011)  
MAT工具分析了heap dump后在界面上非常直观的展示了一个饼图，中间的饼状图就是根据我们上文所说的Retained heap的概念得到的内存中一些Retained Size最大的对象。点击饼状图能看到这些对象类型，但对内存泄漏的分析还远远不够。我们需要从**Dominator Tree**和**Histogram**分析。  
  
    在分析之前先介绍2个概念，**Shallowheap**和**Retained heap**。Shallow heap表示对象本身所占内存大小，一个内存大小100bytes的对象Shallow heap就是100bytes。Retained heap表示通过回收这一个对象总共能回收的内存，比方说一个100bytes的对象还直接或者间接地持有了另外3个100bytes的对象引用，回收这个对象的时候如果另外3个对象没有其他引用也能被回收掉的时候，Retained heap就是400bytes。  

    1.  Dominator Tree  
        Dominator Tree是显示最大存活对象的列表，可以通过过滤查看想关注的对象： 
        ![image](http://img.blog.csdn.net/20161011152730917)  
        
        现在想查看SecondActivity对象有可能被那些对象所引用导致内存泄漏，就是说SecondActivity应该被垃圾回收的，但是因为SecondActivity对象被其他对象强引用，导致无法回收。  
        
        ![image](http://img.blog.csdn.net/20161011152804397)   
        可以看到SecondActivity被mResource对象强引用，很有可能mResource导致SecondActivity内存泄漏。  
        
        ![image](http://img.blog.csdn.net/20161011152827653)  
    2.  Histogram  
        Histogram按类名将所有的实例对象列出来，可以点击表头进行排序,在表的第一行可以输入正则表达式来匹配结果 :  
        ![image](http://img.blog.csdn.net/20161011152844528)  
        
        可以看到存在一个SecondActivity对象，想知道这个对象被那个对象引用，同样的做法：  
        ![image](http://img.blog.csdn.net/20161011152901538)   
        
        可以看到SecondActivity被mResource对象强引用，很有可能mResource导致SecondActivity内存泄漏。
        ![image](http://img.blog.csdn.net/20161011152920998)  
 
 
    
##### 2. LeakCanary
LeakCanary 是 Android 和 Java 内存泄露检测框架。LeakCanary能自动完成检测Activity内存泄漏，甚至在发生 OOM 之前，就把内存泄漏报告给你。  
使用步骤：  
- **在build.gradle配置**
```
debugCompile 'com.squareup.leakcanary:leakcanary-android:1.4-beta2'
releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.4-beta2'
testCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.4-beta2'

```
- **自定义的Application**

```
public class MyApplication extends Application {

    private RefWatcher refWatcher;

    public static RefWatcher getRefWatcher(Context context) {
        MyApplication application = (MyApplication) context.getApplicationContext();
        return application.refWatcher;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        refWatcher = LeakCanary.install(this, DisplayLeakService.class, AndroidExcludedRefs.createAppDefaults().build());
    }
}

```
- **发现存在内存泄漏**  
在调试版本中，如果发现存在内存泄漏，LeakCanary会自动显示一个通知给你。
![image](http://img.blog.csdn.net/20161011152940195)  
很容易看出来，AsyncTask持有MainActivity的引用，当MainActivity退出时，GC没法回收MainActivity的内存，导致内存泄漏。  

***使用LeakCanary有注意的地方***：
- 除了Activity对象，其他对象的检测需要显示监听
```
RefWatcher refWatcher = MyApplication.getRefWatcher(this);
refWatcher.watch(schrodingerCat);

```
- 同一时间段出现多个对象内存泄漏，只能解析一个对象的内存泄漏信息
```
if (isInAnalyzerProcess(application)) {
  return RefWatcher.DISABLED;
}

```
- 官方建议不在发布版本使用
> LeakCanary should only be used in debug builds, and should be disabled in release builds. We provide a special empty dependency for your release builds: leakcanary-android-no-op.
The full version of LeakCanary is bigger and should never ship in your release builds

- hprof文件保存在本地download目录  
  
