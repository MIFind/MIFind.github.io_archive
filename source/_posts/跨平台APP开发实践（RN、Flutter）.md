# 跨平台APP开发实践（RN、Flutter）

----


## 0. 前言

``跨平台的优势``：

> * ``真正的原生应用``：产生的不是网页应用，不是混合应用，而是一个原生的移动应用。
> * ``快速开发应用``：相比原生漫长的编译过程，Hot Reload简直不要太爽。
> * ``可随时呼叫原生外援``：完美兼容Java/Swift/OC的组件，一部分用原生一部分用RN来做完全可以。
> * ``跨平台``：一套业务逻辑代码可以稳定运行在两个平台。
> * ``节省劳动力``：为企业节省劳动力。。。（不知道算不算好事儿）。
> 


* ##### 使用Flutter的应用：闲鱼、美团B端、阿里（FlutterGo、AliFlutter、淘宝特价版）、字节（今日头条、西瓜视频、皮皮虾）

* ##### 使用React Native的应用：携程、美团C端、字节（今日头条）、手机QQ、FaceBook

可以看出RN和Flutter还是呈五五开的发展态势。
github：

* [react-native](https://github.com/facebook/react-native)
* [flutter](https://github.com/flutter/flutter)

但是Flutter是在18年底才发行了以第一个稳定版，而React Native是15年就已经推出。这么一看，Flutter突然🔥起来，就1年的时间就挤掉了RN的大半市场，今天我们一起看一下，这两个跨平台的框架究竟有什么神奇的地方。

----


## 1. React Native的入门与实践
React Native是带着React的光环出生的一个跨平台框架，具备React的一切新特性，让从Ionic与HBuilder的时代走过的Hybrid的开发欲罢不能。因为他能通过React的代码与通用的业务逻辑，编写一套完全原生的App应用，而且APP的使用感受与OC/JAVA编写的Native APP完全一致。

### 1. 搭建环境
* node
* watchman: 监视文件并且记录文件的改动情况。
* Android环境：
	* JDK
	* Android Studio → Android SDK/Gradle/Android SDK Build-Tools
* IOS环境:
	* Xcode
	* CocoaPods

### 2. 创建应用
``react-native init demo``

### 3. 写法与代码结构
React Native和React的基本业务逻辑与项目结构是相通的，除了组件是从react-native的包里引用，样式是css的子集，其他的都是页面的生命周期，渲染逻辑，diff都与React无异。

[萌推商家APP代码](http://git.innotechx.com/projects/X/repos/mms-rn/browse)

Android/IOS目录分别承载这各自应用架构与bundle入口，src部分会打包成jsbundle，然后通过Native的入口注入。

* 打开AS与Xcode查看一下对应平台代码

### 4.RN的运行机制

看到这里会有这样的一个疑问``为什么js代码可以运行在APP中？``

> 是因为RN有两个核心
> 
*  JSC引擎：1 → 因为RN的包里一个有JS执行引擎（WebKit的内核JavaScriptCore），所以它可以运行js代码。（前期是JSC的环境，在0.60.x之后添加了Hermes作为js引擎）。
>
[干货 | 加载速度提升15%，携程对RN新一代JS引擎Hermes的调研](https://cloud.tencent.com/developer/article/1492194)
>
[React Native JSC源码](https://sourcegraph.com/github.com/facebook/react-native@0.30-stable/-/blob/ReactCommon/cxxreact/JSCExecutor.cpp#L24:11)
>
*  JSI通信：其实就是JSBridge，作为JS与Native的桥梁，运行在JSC环境下，通过C++实现的Native类的代理对象，这样就可以实现JS与Native通信。

所以：JSC/Hermes会将作为JS的运行环境（解释器），JS层通过JSI获取到对应的C++层的module对象的代理，最终通过JNI回调Java层的module，在通过JNI映射到Native的函数。


[RN Native Android Module源码](https://sourcegraph.com/github.com/facebook/react-native@v0.63.0-rc.1/-/tree/ReactAndroid/src/main/java/com/facebook/react/views/text)

[RN Native IOS Module源码](https://sourcegraph.com/github.com/facebook/react-native@v0.63.0-rc.1/-/blob/React/Views/RCTView.m)

所以，RN中所有的标签其实都不是真是的控件，js代码中所有的控件，都是一个“Map对中的key”，JS通过这个key组合的DOM，放到VDOM的js数据结构中，然后通过JSBridge代理到Native，Native端会解析这个DOM，从而获得对应的Native的控件。

### 5.怎么实现一个Native Bridge的功能。（先不讲）
> 例子：实现判断应用是否开启通知，如果未打开通知则进入设置页面开启通知。
> 

5.1 ``IOS端 ``

* IOS在 React Native 中，一个“原生模块”就是一个实现了“RCTBridgeModule”协议的 Objective-C 类。

``` 先定义头文件
#import <Foundation/Foundation.h>
#import <React/RCTEventEmitter.h>

@interface RNDataTransferManager : RCTEventEmitter <RCTBridgeModule>

@end

```

```  实现
#import "RNDataTransferManager.h"

@implementation RNDataTransferManager

RCT_EXPORT_MODULE();
// 判断notification是否开启
RCT_EXPORT_METHOD(isNotificationEnabled:(RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject) {
  BOOL isEnable = NO;
  UIUserNotificationSettings *setting = [[UIApplication sharedApplication] currentUserNotificationSettings];
  isEnable = (UIUserNotificationTypeNone == setting.types) ? NO : YES;
  return resolve(@(isEnable));
}

// 进入设置开启Notification
RCT_EXPORT_METHOD(gotoOpenNotification) {
  [self goToAppSystemSetting];
}
```

注意两个宏：

``RCT_EXPORT_METHOD ``：用来设置给JS导出的Native Module名字。

``RCT_EXPORT_MODULE ``：给JS提供的方法通过``RCT_EXPORT_METHOD()``宏实现，必须明确的声明要给 JavaScript 导出的方法，否则 React Native 不会导出任何方法。



5.2 ``Android端 ``

首先新建一个JavaModule类继承ReactContextBaseJavaModule。

```
public class RNDataTransferManager extends ReactContextBaseJavaModule {

    private static ReactApplicationContext reactContext;

    public static RNDataTransferManager rnDataTransferManager;

    public static String currentBindAlias = "";

    public RNDataTransferManager(@Nonnull ReactApplicationContext reactContext) {
        super(reactContext);
        this.reactContext = reactContext;
    }

    public static RNDataTransferManager getInstance() {
        if (null == rnDataTransferManager) {
            rnDataTransferManager = new RNDataTransferManager(reactContext);
        }
        return rnDataTransferManager;
    }

    @Nonnull
    @Override
    public String getName() {
        return "RNDataTransferManager";
    }

        @ReactMethod
    public void isNotificationEnabled(Promise promise) {
        if (promise != null) {
            if (MainApplication.getContext() != null) {
                if (NotificationManagerCompat.from(MainApplication.getContext())
                        .areNotificationsEnabled()) {
                    Log.e("push", "推送开启 isNotificationEnabled -> true");
                    promise.resolve(true);
                } else {
                    Log.e("push", "推送未开启 isNotificationEnabled -> false");
                    promise.resolve(false);
                }
            } else {
                promise.resolve(false);
            }
        }
    }

    @ReactMethod
    public boolean gotoOpenNotification() {
        if (MainApplication.getContext() == null) {
            return false;
        }
        Intent intent = getSetIntent(MainApplication.getContext());
        PackageManager packageManager = MainApplication.getContext().getPackageManager();
        List<ResolveInfo> list = packageManager.queryIntentActivities(intent, 0);
        if (list != null && list.size() > 0) {
            try {
                MainApplication.getContext().startActivity(intent);
                return true;
            } catch (Exception e) {
                return false;
            }
        }
        return false;
    }

}

```

写好了Native Module之后需要注册模块。

1）首先通过ReactPackage的createNativeModules来注册模块。

```
package com.mengtuiapp.mms.bridge;

import com.facebook.react.ReactPackage;
import com.facebook.react.bridge.NativeModule;
import com.facebook.react.bridge.ReactApplicationContext;
import com.facebook.react.uimanager.ViewManager;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import javax.annotation.Nonnull;

public class DataTransferPackage implements ReactPackage {

    private RNDataTransferManager transferModule;

    @Nonnull
    @Override
    public List<NativeModule> createNativeModules(@Nonnull ReactApplicationContext reactContext) {
        List<NativeModule> nativeModules = new ArrayList<>();
        transferModule = new RNDataTransferManager(reactContext);
        RNDataTransferManager.rnDataTransferManager = transferModule;
        nativeModules.add(transferModule);
        return nativeModules;
    }

    @Nonnull
    @Override
    public List<ViewManager> createViewManagers(@Nonnull ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }
}

```

2）然后让你的应用拿到注册到的package，需要在Application的getPackages方法中提供。

```
   @Override
   protected List<ReactPackage> getPackages() {
                    List<ReactPackage> packages = new PackageList(this).getPackages();
                    packages.add(new DataTransferPackage());
                    packages.add(new RNInstallApkPackage());
                    packages.add(new RNUserAgentPackage());
                    packages.add(new RNKeyboardAdjustPackage());
                    packages.add(new CodePush(mContext.getString(R.string.InnotechCodepushKey), mContext, this.moduleId, BuildConfig.DEBUG, mContext.getString(R.string.InnotechCodepushServerUrl)));
                    return packages;
   }
```

5.3 JS端调用

```
NativeModules.RNDataTransferManager.gotoOpenNotification()
就可以前往应用设置页面打开通知。
```


* 当然我们实际开发中不会围绕这么多Native Module来做文章，但是可以看出，做RN是需要一个基本的原生的操作能力。


### 6. 这样做的优势和问题

#### 优势：
> * 相比Hybrid性能更高、因为都是原生组件的渲染。
> * 从render到virtual dom的过程都是React驱动，具备React的一切优秀特性，可以使用React的社区优秀工具。
> * 项目搭建起来了，用JS写APP又具有原生的渲染效率简直爽，一份代码Android、IOS、web都可以适配（毕竟vDom层是一样的，jsbridge就随你魔改了）。
> * 相比原生的编译速度，开发JS使用HotReload简直太爽了。

#### 问题：
> * 跨平台，但是Android、IOS毕竟是不同的系统与生态，组件与功能都有一些跨平台的差异，RN的原生组件的平台差异性很大。
> * 性能问题：动画性能不好、列表数据量大性能不好，主要集中在低端机，大数据列表快速滑动会有白屏，动画层级多在Android低于30fps的情况频繁。
> * 白屏问题，加载bundle的时间会有一个白屏出现，需要手动改Native代码。
> * 开发业务功能不需要原生能力，但是开发一个完整的跨平台项目，是需要具备一定的双端原生能力（有很多要写Native的，许多功能和组件也需要自己封装）
> * 这也是RN做的不好的地方，版本迭代太慢，不痛不痒的迭代了5年了，很多问题还是没有解决。这也是Flutter为什么这么火的原因。

### 7. 那么Flutter怎么做的
Flutter使用Dart作为开发语言，作为一个AOT框架，Flutter是在运行时直接将Dart转化成客户端的可执行文件，然后通过Skia渲染引擎直接渲染到硬件平台。如果说RN是为开发者做了平台兼容，那Flutter更像是为开发者屏蔽了平台的概念。RN需要被转译为本地对应的组件，而Flutter是直接编译成可执行文件并渲染到设备。Flutter可以直接控制屏幕上的每一个像素，这样就可以避免由于使用JSBridge导致的性能问题。

>
三要素：
>
* Dart语言开发。
* 任何Dart代码都是AOT运行前编译成本地可执行文件，使用Skia（渲染引擎）直接渲染到本机。
* 不使用原生的组件，具有自己的widget库，开发时构建自己的widget树来画页面。

#####  如果是页面级的应用来说，Flutter是不需要任何原生代码来写组件，所有组件和页面都可以通过Flutter直接写好。

#### 7.1 Flutter代码结构
[萌推商家APP Flutter版本](https://gitlab.innotechx.com/xuanjiawei/flutter_mms_app)

#### 7.2 Flutter运行与调试
直接演示。

### 8. 总结
今天我们主要看了一下两个框架开发时的代码结构，与代码书写形式，还有简单了解了一下它是怎么运行。那之后如果小伙伴想继续去学习，或做一个自己的应用，还有以下几个方面需要注意：

	1. APP初始化与生命周期状态。
	2. 数据持久化 - 数据管理、SP、本地数据库。
	3. 碎片化处理。
	4. 打包三要素：Android（混淆、签名、加固），IOS（生成证书、导入证书、使用证书）。
	5. 拆包、热更新、原生集成。

Flutter因为自带了渲染引擎，理论上是要比RN渲染效率要高，但是其实实际使用上，在性能过剩的移动端设备中，并没有出现特别大的差异，而Facebook的团队在Flutter的持续施压之下也决定重构底层，并在最近几个版本有了一些进步，所以大家有兴趣的都可以研究一下。


