# Android】热修复——Tinker（入门）
## 前言
不知你是否遇到这样的情况？千辛万苦上开发了一个版本，好不容易上线了，突然发现了一个严重bug需要进行紧急修复，怎么办？难道又要重新打包App、测试，发布新个版本？就为了修改一两行的代码？
莫慌，这种问题其实可以分分钟解决。如果你学会了这项黑科技——热修复。
在用户使用App的时候，不知不觉，这个Bug就被修复了。
![莫慌](http://upload-images.jianshu.io/upload_images/1638147-b6a03e2bd2224d96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


> 热修复:**热修复**（也称**热补丁**、**热修复补丁**，英语：**hotfix**）是一种包含信息的独立的累积更新包，通常表现为一个或多个文件。这被用来解决软件产品的问题（例如一个程序错误）。——维基百科

本文介绍了Tinker的接入方式，更加详细的内容可以查阅[官方文档](http://tinkerpatch.com/Docs/SDK)

## 介绍
Tinker是微信官方的Android热补丁解决方案，它支持动态下发代码、So库以及资源，让应用能够在不需要重新安装的情况下实现更新。当然，你也可以使用Tinker来更新你的插件。
Tinker所支持的功能如下
![来自官方Github](http://upload-images.jianshu.io/upload_images/1638147-a413068d4fb48dc5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Tinker热补丁方案·不仅支持类、So以及资源的替换，它还是2.X－7.X的全平台支持。

## 接入
进入正题吧
![额，我是说进入正题](http://upload-images.jianshu.io/upload_images/1638147-445b88427fbc65f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在项目的`build.gradle`中
```java
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        // TinkerPatch 插件
        classpath "com.tinkerpatch.sdk:tinkerpatch-gradle-plugin:1.1.4"
    }
}
```
然后在`app`的`gradle`文件`app/build.gradle`中
```java
dependencies {
    // 若使用annotation需要单独引用,对于tinker的其他库都无需再引用
    provided("com.tencent.tinker:tinker-android-anno:1.7.7")
    compile("com.tinkerpatch.sdk:tinkerpatch-android-sdk:1.1.4")
}
```
在app目录下，创建`tinkerpatch.gradle`(可以去后面的链接下载源码，把这个文件拷进去)
![tinkerpatch.gradle](http://upload-images.jianshu.io/upload_images/1638147-f67458db1f8256a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
将 TinkerPatch 相关的配置都放于`tinkerpatch.gradle`中,然后在app的`gradle`文件`app/build.gradle`中还添加
```java
apply from: 'tinkerpatch.gradle'
```
完整的`app/buidl.gradle`:
```java
apply plugin: 'com.android.application'
apply from: 'tinkerpatch.gradle'
android {
    ...
}
dependencies {
    .....
    // 若使用annotation需要单独引用,对于tinker的其他库都无需再引用
    provided("com.tencent.tinker:tinker-android-anno:1.7.7")
    compile("com.tinkerpatch.sdk:tinkerpatch-android-sdk:1.1.4")
}
```

## 配置参数
打开之前创建的`tinkerpatch.gradle`添加
```java
apply plugin: 'tinkerpatch-support'

/**
 * TODO: 请按自己的需求修改为适应自己工程的参数
 */
def bakPath = file("${buildDir}/bakApk/")
def baseInfo = "app-1.0.0-0330-20-37-31"
def variantName = "release"

/**
 * 对于插件各参数的详细解析请参考
 * http://tinkerpatch.com/Docs/SDK
 */
tinkerpatchSupport {
    /** 可以在debug的时候关闭 tinkerPatch **/
    tinkerEnable = true
    reflectApplication = true

    autoBackupApkPath = "${bakPath}"

    appKey = "你的appkey"//申请的appkey

    /** 注意: 若发布新的全量包, appVersion一定要更新 **/
    appVersion = "1.0.0"

    def pathPrefix = "${bakPath}/${baseInfo}/${variantName}/"
    def name = "${project.name}-${variantName}"

    baseApkFile = "${pathPrefix}/${name}.apk"
    baseProguardMappingFile = "${pathPrefix}/${name}-mapping.txt"
    baseResourceRFile = "${pathPrefix}/${name}-R.txt"
    /**
     *  若有编译多flavors需求, 可以参照： https://github.com/TinkerPatch/tinkerpatch-flavors-sample
     *  注意: 除非你不同的flavor代码是不一样的,不然建议采用zip comment或者文件方式生成渠道信息（相关工具：walle 或者 packer-ng）
     **/
}

/**
 * 用于用户在代码中判断tinkerPatch是否被使能
 */
android {
    defaultConfig {
        buildConfigField "boolean", "TINKER_ENABLE", "${tinkerpatchSupport.tinkerEnable}"
    }
}
/**
 * 一般来说,我们无需对下面的参数做任何的修改
 * 对于各参数的详细介绍请参考:
 * https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97
 */
tinkerPatch {
    ignoreWarning = false
    useSign = true
    dex {
        dexMode = "jar"
        pattern = ["classes*.dex"]
        loader = []
    }
    lib {
        pattern = ["lib/*/*.so"]
    }

    res {
        pattern = ["res/*", "r/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]
        ignoreChange = []
        largeModSize = 100
    }
packageConfig {
    }
    sevenZip {
        zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
//        path = "/usr/local/bin/7za"
    }
    buildConfig {
        keepDexApply = false
    }
}
```
- 参数说明
 - bakPath：基包路径
 - baseInfo：基包文件夹名（打补丁包的时候，需要修改）
 - appKey：进入[官网](http://tinkerpatch.com/ )注册一个账号，新增APP，得到对应的appKey。
![官方参数说明](http://upload-images.jianshu.io/upload_images/1638147-a2cc88fe29fa6248.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 使用
创建`FetchPatchHandler`用于检测是否更新（刚打开时会检测一次）
```java
public class FetchPatchHandler extends Handler {
    public static final long HOUR_INTERVAL = 3600 * 1000;
    private long checkInterval;

    /**
     * 通过handler, 达到按照时间间隔轮训的效果
     * @param hour
     */
    public void fetchPatchWithInterval(int hour) {
        //设置TinkerPatch的时间间隔
        TinkerPatch.with().setFetchPatchIntervalByHours(hour);
        checkInterval = hour * HOUR_INTERVAL;
        //立刻尝试去访问,检查是否有更新
        sendEmptyMessage(0);
    }
    @Override
    public void handleMessage(Message msg) {
        super.handleMessage(msg);

        //这里使用false即可
        TinkerPatch.with().fetchPatchUpdate(false);
        //每隔一段时间都去访问后台, 增加10分钟的buffer时间
        sendEmptyMessageDelayed(0, checkInterval + 10 * 60 * 1000);
    }
}
```
创建`MyApplication`初始化(reflectApplication = true 时)
```java
public class MyApplication extends Application{
    private ApplicationLike tinkerApplicationLike;
    @Override
    public void onCreate() {
        super.onCreate();
        if (BuildConfig.TINKER_ENABLE) {

            // 我们可以从这里获得Tinker加载过程的信息
            tinkerApplicationLike = TinkerPatchApplicationLike.getTinkerPatchApplicationLike();

            // 初始化TinkerPatch SDK, 更多配置可参照API章节中的,初始化SDK
            TinkerPatch.init(tinkerApplicationLike)
                    .reflectPatchLibrary()
                    .setPatchRollbackOnScreenOff(true)
                    .setPatchRestartOnSrceenOff(true);

            // 每隔3个小时去访问后台时候有更新,通过handler实现轮训的效果
            new FetchPatchHandler().fetchPatchWithInterval(3);
        }
    }
}
```
**注意：初始化的代码建议紧跟 super.onCreate(),并且所有进程都需要初始化，已达到所有进程都可以被 patch 的目的
如果你确定只想在主进程中初始化 tinkerPatch，那也请至少在 :patch 进程中初始化，否则会有严重的 crash 问题**

### 打生产包
每次开发完成后，开始打包。
打开`Studio`右侧的`Gradle`，选择`assemableRelease`打正式包
![Gradle](http://upload-images.jianshu.io/upload_images/1638147-543425aa2190aad9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

完成后可以在文件夹`build`中找到生成的文件（这里称为基包）
![结果](http://upload-images.jianshu.io/upload_images/1638147-93041baf24c717e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

打开`build` -> `bakApk`  -> `app-1.0.0-0330-21-40-52 `  (根据时间命名)
`release`文件夹中会出现我们刚打完的包。一个`apk`，对应一个txt文件。
将apk安装到手机上，这是一个刚创建的项目，里面只有Hello World
![](http://upload-images.jianshu.io/upload_images/1638147-0bdf8cbbfd9542f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>将`app-1.0.0-0330-21-40-52`备份，打补丁包的时候需要用到。 
多渠道打包时会根据渠道名分包，目录结构相似。

### 生成补丁包
这里模拟一个修补bug的场景，发一个热补丁，弹一个Toast。
**注意：在打生产包的代码上做修改**
```java
Toast.makeText(this, "这是个补丁", Toast.LENGTH_SHORT).show();
```
- 将之前的备份好的基包复制到`build/bakApk`下，打开`tinkerpatch.gradle`修改`baseInfo`成对应的文件名
![修改baseInfo](http://upload-images.jianshu.io/upload_images/1638147-e0f4097237e5bd46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 修改`tinkerpatch.gradle`中的`tinkerpatchSupport` -> `appVersion`
![appVersion](http://upload-images.jianshu.io/upload_images/1638147-61b68a1f4fcac3ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 完成后打开`Gradle`，如下选择`tinkerPatchRelease`
![Gradle](http://upload-images.jianshu.io/upload_images/1638147-8dbf6e35bc04bbcf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
补丁包将位于 build/outputs/tinkerPatch 中，这里只需要用到patch_signed_7zip.apk
![补丁包](http://upload-images.jianshu.io/upload_images/1638147-a7d5502798547c56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 发布
最后，只需要将刚生成的补丁包发布，然后静静等待即可。
- 进入[官方网站](http://tinkerpatch.com/Apps/index)，选择对应的app，添加APP版本
![添加APP版本](http://upload-images.jianshu.io/upload_images/1638147-6a5d654cd0d44c58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![添加版本](http://upload-images.jianshu.io/upload_images/1638147-c002e2d8b9250429.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**版本号对应`tinkerpatch.gradle`中的appVersion**
- 选择`patch_signed_7zip.apk`文件，提交即可（更多下发选项，参考官方文档）
![发布补丁](http://upload-images.jianshu.io/upload_images/1638147-d0e83807d8b01394.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 提交后，查看补丁的下载数量以及成功应用数
![补丁详情](http://upload-images.jianshu.io/upload_images/1638147-1eb91c0e0c1d7a6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这时候重新打开app，等待补丁下载。下载完成后关闭app，再次打开，查看结果。就这样，整个热修复的流程就完成了。
**注意：一定要关闭后打开，热修复才会生效。**
![结果](http://upload-images.jianshu.io/upload_images/1638147-22443e6292f318ed.gif?imageMogr2/auto-orient/strip)


###关于兼容多渠道包
关于渠道包的问题，若使用`flavor`编译渠道包，会导致不同的渠道包由于`BuildConfig`变化导致`classes.dex`差异。这里建议的方式有：
- 将渠道信息写在`AndroidManifest.xml`或文件中，例如`channel.ini`；
- 将渠道信息写在`ap`k文件的`zip comment`中，这种是建议方式，例如可以使用项目[packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin)或者可使用V2 Scheme的[walle](https://github.com/Meituan-Dianping/walle)；
- 若不同渠道存在功能上的差异，建议将差异部分放于单独的dex或采用相同代码不同配置方式实现；

**已通过Walle实现**：[【Android】Walle多渠道打包&Tinker热修复](http://www.jianshu.com/p/0ba717f7385f)

## Tinker已知的问题：
  1. Tinker不支持修改`AndroidManifest.xml`，Tinker不支持新增四大组件；
  2. 由于Google Play的开发者条款限制，不建议在GP渠道动态更新代码；
  3. 在`Android N`上，补丁对应用启动时间有轻微的影响；
  4. 不支持部分三星android-21机型，加载补丁时会主动抛出`TinkerRuntimeException:checkDexInstall failed`；
  5. 由于各个厂商的加固实现并不一致，在1.7.6以及之后的版本，tinker不再支持加固的动态更新；
  6. 对于资源替换，不支持修改`remoteView`。例如`transition`动画，`notification icon`以及桌面图标。

## Tips:
- `dubug`模式下，`tinkerpatch.gradle` —> `tinkerpatchSupport `—>`tinkerEnable`需要改为`false`
- 添加`SD`卡权限，
- 下载补丁后，杀掉进程重新打开，补丁才会生效
- 补丁非即时生效，需要等一会儿

## 源码地址
[Github](https://github.com/Gavin-ZYX/TinkerTest)

## 参考
[官方文档](http://tinkerpatch.com/Docs/SDK)
[接入指南](https://github.com/Tencent/tinker/wiki)

> 以上有错误之处，感谢指出
