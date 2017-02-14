##Android Lint常见问题分析(for studio)
@(jarlen)[Android lint|静态分析]

-------------------

[TOC]

###Android
####android resource Validation
 验证Android xml文件内的资源引用
```
Unresolved class 'MainActivity'
Cannot resolve symbol '@android:attr/textAppearanceMedium'
Cannot resolve symbol '@android:attr/textAppearanceMedium'
```

####Android XML root tag validation
检查XML资源是否存储在规定的资源文件夹中
```
<animation-list> XML file should be in "drawable", not "anim"
//应该写在drawable文件中
```
####Missing JNI function
本地方法声明在java里但没有相应的JNI函数
注: 本地库放在jniLibs文件夹下，而不是libs下。cpu架构兼容问题通过ndk{abiFilters ""} 

####onClick handler is missing in the related activity
onClick 中的方法需要在响应的activity中声明实现关联。
```
<LinearLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:padding="10dp"
            android:gravity="center"
        	android:layout_margin="5dp"
            android:background="@drawable/back_ground"
            android:onClick="onCarPlateLayout"
            android:orientation="vertical" >
```
###Android > Lint > Performance(性能)
####FrameLayout can be replaced with <merge> tag

使用<merge>代替FrameLayout实现视图叠加。节约资源。
[资料查看](http://blog.csdn.net/achellies/article/details/7105840)

####Handler reference leaks
使用Handler引发的内存泄露

```
public class SampleActivity extends Activity {
  
  private final Handler mLeakyHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ...
    }
  }
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // Post a message and delay its execution for 10 minutes.
    mLeakyHandler.postDelayed(new Runnable() {
      @Override
      public void run() { /* ... */ }
    }, 1000 * 60 * 10);
    // Go back to the previous Activity.
    finish();
  }
}
```
当我们执行了Activity的finish方法，被延迟的消息会在被处理之前存在于主线程消息队列中10分钟，而这个消息中又包含了Handler的引用，而Handler是一个匿名内部类的实例，其持有外面的SampleActivity的引用，所以这导致了SampleActivity无法回收，进行导致SampleActivity持有的很多资源都无法回收，这就是我们常说的内存泄露。

要解决这种问题，思路就是不使用非静态内部类，继承Handler时，要么是放在单独的类文件中，要么就是使用静态内部类。因为静态的内部类不会持有外部类的引用，所以不会导致外部类实例的内存泄露。当你需要在静态内部类中调用外部的Activity时，我们可以使用弱引用来处理。另外关于同样也需要将Runnable设置为静态的成员属性。注意：一个静态的匿名内部类实例不会持有外部类的引用。 修改后不会导致内存泄露的代码如下:
```
public class SampleActivity extends Activity {
 
  private static class MyHandler extends Handler {
    private final WeakReference<sampleactivity> mActivity;
 
    public MyHandler(SampleActivity activity) {
      mActivity = new WeakReference<sampleactivity>(activity);
    }
 
    @Override
    public void handleMessage(Message msg) {
      SampleActivity activity = mActivity.get();
      if (activity != null) {
        // ...
      }
    }
  }
 
  private final MyHandler mHandler = new MyHandler(this);
  private static final Runnable sRunnable = new Runnable() {
      @Override
      public void run() {
      //do nothing
      }
  };
 
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
 
    // Post a message and delay its execution for 10 minutes.
    mHandler.postDelayed(sRunnable, 1000 * 60 * 10);

    // Go back to the previous Activity.
    finish();
  }
}
```

####HashMap can be replaced with SparseArray

针对Android这种移动平台，也推出了更符合自己的api，比如SparseArray、ArrayMap用来代替HashMap在有些情况下能带来更好的性能提升。
假设数据量都在千级以内的情况下：

1、如果key的类型已经确定为int类型，那么使用SparseArray，因为它避免了自动装箱的过程，如果key为long类型，它还提供了一个LongSparseArray来确保key为long类型时的使用

2、如果key类型为其它的类型，则使用ArrayMap

####Inefficient layout weight

例如:
```
Use a 'layout_height' of '0dp' instead of '1px' for better performance
```
```
<View
   android:layout_width="match_parent"
   android:layout_height="1px"
   android:layout_weight="1"
   android:layout_marginRight="@dimen/common_padding_16"
   android:background="#8A000000"
   />
```

####Memory allocations within drawing code
绘制时内存分配. 避免绘制的时候创建对象,否则就提示下面的警告信息：因为onDraw()调用频繁，不断进行创建和垃圾回收会影响UI显示的性能

```
Avoid object allocations during draw/layout operations (preallocate and reuse instead)
```

####Missing baselineAligned attribute
当LinerLayout的子View都是ViewGroup（自定义控件除外）时，Lint认为它的子View已经不需要基准线对齐了，为了不让LinerLayout去做无用的计算对齐的操作，提出了如上警告，修改掉之后就可以提高性能。

####Missing recycle() calls

- `This 'Cursor' should be freed up after use with '#close()'`

- `This 'TypedArray' should be recycled after use with '#recycle()'`

####Nested layout weights
权值会被计算两次, 改变时, 会按照比例进行改变.
```
<ImageButton
    android:clickable="false"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    android:layout_weight="1"
    android:background="@color/cor6"
    android:src="@drawable/classmate_selector" 
    />
```
解决方法: 使用RelativeLayout或GridLayout代替LinearLayout.

[stackoverflow](http://stackoverflow.com/questions/21387626/android-nested-weight)

####Node can be replaced by a TextView with compound drawables
使用TextView来代替
```
<LinearLayout
        android:id="@+id/txl_word_layout"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <TextView
            android:id="@+id/txl_word"
            android:layout_width="fill_parent"
            android:layout_height="32dp"
            android:background="@color/cor5"
            android:gravity="center_vertical"
            android:paddingLeft="16dp"
            android:textColor="@color/cor8"
            android:textSize="@dimen/size3"
            android:textStyle="bold" />

        <ImageView
            android:layout_width="fill_parent"
            android:layout_height="@dimen/dividerHeight"
            android:src="@color/cor4"
            tools:ignore="ContentDescription" />
    </LinearLayout>
```

####Obsolete layout params
废弃或无效的布局参数

####Overdraw: Painting regions more than once
过度绘制
```
Possible overdraw: Root element paints background '@color/background_color' with a theme that also paints a background (inferred theme is '@style/AppTheme')
```
一般布局根背景不需要在设置bg了

####Should use valueOf instead of new
[案例](http://stackoverflow.com/questions/25545718/method-invokes-inefficient-new-integerint-constructor-use-integer-valueofint)

####Static Field Leaks
静态变量泄露

####Unused resources
未使用的资源

####Useless parent layout
多余的父布局

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <LinearLayout
        android:id="@+id/ly_load_more"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:orientation="horizontal"
        android:paddingBottom="10dp"
        android:paddingTop="10dp">

        <ProgressBar
            android:id="@+id/video_item_image"
            android:layout_width="@dimen/padding_30"
            android:layout_height="@dimen/padding_30"
            android:layout_marginRight="5dp"
            android:indeterminateDrawable="@drawable/inner_animation" />

        <TextView
            android:id="@+id/video_item_label"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginLeft="7dp"
            android:text="@string/pull_to_refresh_refreshing_label"
            android:textColor="@color/cor13"
            android:textSize="15sp" />
    </LinearLayout>
</LinearLayout>
```
####Using FloatMath instead of Math

貌似Android 6.0 SDK FloatMath不能用

###Android > Lint > Security(安全性)

####addJavascriptInterface Called
addJavascriptInterface 能帮助调用你的JavaScript函数中的任意活动方式。

- addJavaScriptInterface方式帮助我们从一个网页传递值到Android XML视图（反之亦然）。
- 你可以从网页调用你的活动类方式（反之亦然）。
- 这是一个非常有用的功能，而当WebView中的HTML是不能信赖的，这则是一个非常危险的安全问题，因为攻击者可以注入HTML执行你的代码。
- 除非WebView所有HTML都是你写的，否则不要使用addJavascriptInterface()。

####Cipher.getInstance with ECB

```
'Cipher.getInstance' should not be called without setting the encryption mode and padding
```
####Content provider does not require permission
provider如果exported设置为true,那么需要权限控制来保证安全。
```
Exported content providers can provide access to potentially sensitive data
```

####Exported service does not require permission
同provider。

####Hardware Id Usage
```
Using 'getDeviceId' to get device identifiers is not recommended.
```
可以无视。

####Insecure HostnameVerifier
```
Using the ALLOW_ALL_HOSTNAME_VERIFIER HostnameVerifier is unsafe because it always returns true, which could cause insecure network traffic due to trusting TLS/SSL server certificates for wrong hostnames
```
PS：小弟不才，不太理解，请理解了的同学附上。谢谢

####Insecure TLS/SSL trust manager
```
'checkClientTrusted' is empty, which could cause insecure network traffic due to trusting arbitrary TLS/SSL certificates presented by peers

'checkServerTrusted' is empty, which could cause insecure network traffic due to trusting arbitrary TLS/SSL certificates presented by peers
```

####openFileOutput() or similar call passing MODE_WORLD_WRITEABLE
android有一套自己的安全模型，当应用程序(.apk)在安装时系统就会分配给他一个userid，当该应用要去访问其他资源比如文件的时候，就需要userid匹配。默认情况下，任何应用创建的文件，sharedpreferences，数据库都应该是私有的（位于/data/data/<package name>/files），其他程序无法访问。除非在创建时指定了Context.MODE_WORLD_READABLE或者Context.MODE_WORLD_WRITEABLE ，只有这样其他程序才能正确访问

####Receiver does not require permission
同provider。

####Unsafe Protected BroadcastReceiver
```
This broadcast receiver declares an intent-filter for a protected broadcast action string, which can only be sent by the system, not third-party applications. However, the receiver's onReceive method does not appear to call getAction to ensure that the received Intent's action string matches the expected value, potentially making it possible for another actor to send a spoofed intent with no action string or a different action string and cause undesired behavior.
```
大概意思是 没在onReceive中判断Action的有效性。别的应用可以模拟攻击。



	




