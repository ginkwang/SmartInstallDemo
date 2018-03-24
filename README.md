## Android APK 省心安装 —— 眼睁睁地看着它完成一切

> 项目的一个新需求。因为我们现在做的是一个服务于公共的产品，实施人员也不一定及时地去维护。所以为了方便，我们想要 APK 更新时静默更新，就是不需要人去主动触发或者同意，就能够实现软件更新。上网查资料，最终实现方法如下：

网上关于静默更新的文章有很多，包括郭霖大神都介绍过相关技术。所以这块就不多卖关子了，静默安装 APK 有两种实现方式：

- 静默安装
- 省心安装

效果更好的是静默安装，因为他真的是静默——无声无息。就类似于你使用的是小米手机，然后在小米软件商店下载软件，你会发现你点击下载后不用你做任何同意安装的操作，软件就已经下载到你的手机了。这样自然是最好的效果，也正是我们想要的。但是得到好的东西就要付出一些代价，实现静默安装的一个大前提是——Root 权限。这个条件太苛刻了，你不能要求谁的手机都已经 Root。那你可能会问，为啥小米安装软件就能静默？

没为啥，因为 MIUI 就是人家小米自己定制的 Android 系统，自然是想改什么就改什么。而我们普通开发者肯定是没有这么高的权限了。

网上也有一些教程，就是可以通过 Java 的反射机制获取到 IPackageManager 这个对象，然后使用它的 installPackage 方法进行安装。但是这种方法需要配置很多文件，而且在不同机型，需要配置得文件也不一样。适配难度很大。所以只能忍痛挥别这种最好的方案。

那现在只好选择省心安装的方案了，什么是省心安装？

还拿上面的例子说，你使用小米手机，但是你在应用宝中下载软件，这时你就会遇到这种情况，安装软件时会需要你点击是否同意安装，如下：

![应用宝下载](https://github.com/GinkWang/SmartInstallDemo/blob/master/images/%E5%BA%94%E7%94%A8%E5%AE%9D%E4%B8%8B%E8%BD%BD.gif)

按照前文的逻辑来说，这是正常的情况。因为应用宝在 MIUI 中也只是个普通应用啊，没有 Root 就没法使用静默安装。下图是应用宝的设置界面：
![Ss_2018-03-24_22-21-54](https://github.com/GinkWang/SmartInstallDemo/blob/master/images/%E5%BA%94%E7%94%A8%E5%AE%9D%E8%AE%BE%E7%BD%AE%E7%95%8C%E9%9D%A2.png)

但是可以在上图中看到，应用宝有一个备用解决方案——省心装，关于该功能的介绍是 `安装时无须频繁点击下一步与完成` 。看来还行，我们开启一下这个功能看看。

点击开启之后，会提示要授予应用宝辅助功能权限。于是我们就去辅助功能界面去授权：

![Ss_2018-03-24_22-26-16](https://github.com/GinkWang/SmartInstallDemo/blob/master/images/%E8%BE%85%E5%8A%A9%E7%95%8C%E9%9D%A2.png)

这个界面详细介绍了关于省心装的功能，我们点击右上方开启。然后回到应用宝安装一个应用试一下：

![应用宝省心装](https://github.com/GinkWang/SmartInstallDemo/blob/master/images/%E5%BA%94%E7%94%A8%E5%AE%9D%E7%9C%81%E5%BF%83%E8%A3%85.gif)

可以看到，就如同应用宝对于该功能介绍的那样，我在选择安装软件之后，就没用我再点击确认安装什么的了。一切步骤全是应用宝替我搞定的。这样的实现虽然还是显示安装界面，不如静默安装安静，但是也可以满足我的需求，实现我想要的效果。

所以最终选择省心安装这个方案。另外，这个方案在不同软件上有不同的叫法，有叫他自动装的，也有叫智能装的。反正追根溯源，说的都是一个东西。

下面看一下此方案的具体实现。

先新建一个工程，然后在 `res` 下新建一个 `xml` 目录，在`xml`目录内新建一个`accessibility_service_config.xml`文件，内容如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
android:accessibilityEventTypes="typeAllMask"
android:accessibilityFeedbackType="feedbackGeneric"
android:accessibilityFlags="flagDefault"
android:canRetrieveWindowContent="true"                       android:description="@string/accessibility_desc" android:packageNames="com.android.packageinstaller"/>
```

其中，`packageNames`指定我们要监听哪个应用程序下的窗口活动，这里写`com.android.packageinstaller`表示监听Android系统的安装界面。`accessibility_desc`指定在无障碍服务当中显示给用户看的说明信息，上图中应用宝的一大段内容就是在这里指定的。`accessibilityEventTypes`指定我们在监听窗口中可以模拟哪些事件，这里写`typeAllMask`表示所有的事件都能模拟。`accessibilityFlags`可以指定无障碍服务的一些附加参数，这里我们传默认值flagDefault就行。`accessibilityFeedbackType`指定无障碍服务的反馈方式，实际上辅助功能（无障碍服务）这个功能是 Android 提供给一些残疾人士使用的，比如说盲人不方便使用手机，就可以借助无障碍服务配合语音反馈来操作手机，而我们其实是不需要反馈的，因此随便传一个值就可以，这里传入`feedbackGeneric`。最后`canRetrieveWindowContent`指定是否允许我们的程序读取窗口中的节点和内容，必须写`true`。

然后我们还要在代码中新建一个监听安装程序的 Service，继承自 `AccessibilityService`。

```java
public class SmartInstallAPKAccessibilityService extends AccessibilityService {

    private static final String TAG = "[SmartInstallAPKAccessibilityService]";
    private Map<Integer, Boolean> handleMap = new HashMap<>();

    @Override
    public void onAccessibilityEvent(AccessibilityEvent event) {
        AccessibilityNodeInfo nodeInfo = event.getSource();
        if (nodeInfo != null) {
            int eventType = event.getEventType();
            if (eventType == AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED ||
                    eventType == AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED) {
                if (handleMap.get(event.getWindowId()) == null) {
                    boolean handled = iterateNodesAndHandle(nodeInfo);
                    if (handled) {
                        handleMap.put(event.getWindowId(), true);
                    }
                }
            }

        }
    }

    @Override
    public void onInterrupt() {

    }

    //遍历节点，模拟点击安装按钮
    private boolean iterateNodesAndHandle(AccessibilityNodeInfo nodeInfo) {
        if (nodeInfo != null) {
            int childCount = nodeInfo.getChildCount();
            if ("android.widget.Button".equals(nodeInfo.getClassName())) {
                String nodeCotent = nodeInfo.getText().toString();
                Log.d(TAG, "content is: " + nodeCotent);
                if ("安装".equals(nodeCotent)
//                        || "完成".equals(nodeCotent)
                        || "确定".equals(nodeCotent)
                        || "打开".equals(nodeCotent)) {
                    nodeInfo.performAction(AccessibilityNodeInfo.ACTION_CLICK);
                    return true;
                }
            }
            //遇到ScrollView的时候模拟滑动一下
            else if ("android.widget.ScrollView".equals(nodeInfo.getClassName())) {
                nodeInfo.performAction(AccessibilityNodeInfo.ACTION_SCROLL_FORWARD);
            }
            for (int i = 0; i < childCount; i++) {
                AccessibilityNodeInfo childNodeInfo = nodeInfo.getChild(i);
                if (iterateNodesAndHandle(childNodeInfo)) {
                    return true;
                }
            }
        }
        return false;
    }
}
```

这里的代码含义是，每当有窗口活动时，就会触发 `onAccessibilityEvent()` 方法，我们根据传入的 `AccessibilityEvent` 参数来判断当前事件的类型。我们只需要监听`TYPE_WINDOW_CONTENT_CHANGED`和`TYPE_WINDOW_STATE_CHANGED`这两种事件就可以了，因为在整个安装过程中，这两个事件必定有一个会被触发。当然也有两个同时都被触发的可能，那么为了防止二次处理的情况，这里我们使用了一个`Map`集合来过滤掉重复事件。

然后通过`iterateNodesAndHandle`方法来进行当前界面节点的判断，如果是按钮节点，我们就判断按钮上的文字是不是`安装`、`完成`、`确定`、`打开`这几项，如果是，就进行模拟点击。（`完成`和`打开`不能同时在代码中进行判断，如果你安装完软件不想打开软件，就保留`完成`。反之则保留`打开`） 。

之后还要在清单文件中注册该服务，

```xml
<service
    android:name=".SmartInstallAPKAccessibilityService"
    android:label="智能安装 APK"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE"
    >
    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService"/>
    </intent-filter>
    <meta-data
        android:name="android.accessibilityservice"
        android:resource="@xml/accessibility_service_config"
        />
</service>
```

最后在 Activity 中写使用逻辑，

```java
private Button mBtnOpenAssist;
private Button mBtnSmartInstall;

private String mAPKPath;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    initView();
}

private void initView() {
    mBtnOpenAssist = (Button) findViewById(R.id.btn_open_assist);
    mBtnOpenAssist.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View pView) {
            Intent intent = new Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS);
            startActivity(intent);
        }
    });

    mBtnSmartInstall = (Button) findViewById(R.id.btn_smart_install);
    mBtnSmartInstall.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View pView) {
            Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
            //intent.setType(“image/*”);//选择图片
            //intent.setType(“audio/*”); //选择音频
            //intent.setType(“video/*”); //选择视频 （mp4 3gp 是android支持的视频格式）
            //intent.setType(“video/*;image/*”);//同时选择视频和图片
            intent.setType("*/*");//无类型限制
            intent.addCategory(Intent.CATEGORY_OPENABLE);
            startActivityForResult(intent, 1);
        }
    });
}

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (resultCode == Activity.RESULT_OK) {
        Uri uri = data.getData();
        if ("file".equalsIgnoreCase(uri.getScheme())) {//使用第三方应用打开
            mAPKPath = uri.getPath();
            Toast.makeText(this, mAPKPath, Toast.LENGTH_SHORT).show();
            return;
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {//4.4以后
            mAPKPath = getPath(this, uri);
            Toast.makeText(this, mAPKPath, Toast.LENGTH_SHORT).show();
        } else {//4.4以下下系统调用方法
            mAPKPath = getRealPathFromURI(uri);
            Toast.makeText(MainActivity.this, mAPKPath, Toast.LENGTH_SHORT).show();
        }
        doInstallAPK(new File(mAPKPath));
    }
}

//安装程序
private void doInstallAPK(File pFile) {
    Intent intent = new Intent(Intent.ACTION_VIEW);
    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    intent.setDataAndType(Uri.fromFile(pFile), "application/vnd.android.package-archive");
    startActivity(intent);
}
```

逻辑很简单，就是定义两个按钮，一个是去开启辅助功能，一个是选择 APK 文件进行安装（注意，这里只是核心代码，全部代码见文末链接）。

最后，看一下实际效果：
![实际效果](https://github.com/GinkWang/SmartInstallDemo/blob/master/images/%E5%AE%9E%E9%99%85%E6%95%88%E6%9E%9C.gif)

符合我们的预期。

然后这里还有个地方要注意，因为 Android 系统被各家手机厂商定制的花样百出，所以不一定所有手机的安装程序都叫`com.android.packageinstaller`（好像魅族就不是），所以这个还要自己排一下坑。

参考：

https://blog.csdn.net/guolin_blog/article/details/47803149

https://blog.csdn.net/fuchaosz/article/details/51852442