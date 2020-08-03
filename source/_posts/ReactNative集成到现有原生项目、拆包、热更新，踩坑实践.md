---
title: ReactNative集成到现有原生项目、拆包、热更新
---

如果你正准备从头开始制作一个新的应用，那么 React Native 会是个非常好的选择。但是大多数项目我们已经有一个原生应用，React Native 作为业务 module，集成到原生应用中。我们一个应用中可能有很多 RN 业务模块，那 module 拆包、热更新就尤为重要。

## 集成到现有原生项目

### 集成到原生项目，首先要安装 RN 开发环境。

1. [搭建 RN 开发环境](https://reactnative.cn/docs/getting-started.html)
2. 由于是集成到现有原生项目，不需要 init 一个新的 RN 项目。
   1. 获取 RN 项目库，并执行`npm install`，安装 RN 项目所需要依赖。
   2. 如果没有 RN 项目库，需要同步当前需要集成的 RN 项目的 package.json，并执行`npm install`

### 1 Android

#### 1. project gradle 添加配置

在项目的 build.gradle 文件中为 React Native 添加一个 maven 依赖的入口，必须写在 "allprojects" 代码块中。

```
allprojects {
    repositories {
         //rn
        maven {
            url "$rootDir/../node_modules/react-native/android"

        }
        maven {
            url("$rootDir/../node_modules/jsc-android/dist")
        }
        //qtt maven-> codepush
         maven {
            url "http://nexus.qutoutiao.net/repository/android/"
        }
        //rn end
        ...
    }
    ...
}

```

#### 2. app gradle 添加配置

在 app 中 build.gradle 文件中添加 React Native 依赖:

```
//rn
project.ext.react = [
        entryFile: "index.js",
        enableHermes: false,  // clean and rebuild if changing
        jsBundleDirRelease: "$buildDir/intermediates/merged_assets/release/out/index"
]

apply from: "../../node_modules/react-native/react.gradle"

def enableHermes = project.ext.react.get("enableHermes", false);

def jscFlavor = 'org.webkit:android-jsc:+'

def safeExtGet(prop, fallback) {
    rootProject.ext.has(prop) ? rootProject.ext.get(prop) : fallback
}
//rn end
```

添加 gradle 依赖

```
//rn
    if (enableHermes) {
        def hermesPath = "../../node_modules/hermesvm/android/";
        debugImplementation files(hermesPath + "hermes-debug.aar")
        releaseImplementation files(hermesPath + "hermes-release.aar")
    } else {
        implementation jscFlavor
    }
    implementation "com.facebook.react:react-native:+" // From node_modules
    implementation "androidx.swiperefreshlayout:swiperefreshlayout:1.0.0"

    // 这里使用手动引入依赖，所以要手动添加。如果auto-link这里不用引用。
    implementation project(':@react-native-community_masked-view')
    implementation project(':react-native-gesture-handler')
    implementation project(':react-native-reanimated')
    implementation project(':react-native-safe-area-context')
    implementation project(':react-native-screens')

    // --> codepush
    implementation project(':@innotechx_react-native-code-push')
    //rn end
```

最下面添加 codepush 引用。

```
apply from: file("../../node_modules/@innotechx/react-native-code-push/android/codepush.gradle")
```

注意确认 node_modules 所在的目录位置。

#### 3. settings.gradle 添加配置

```
include ':@innotechx_react-native-code-push'
project(':@innotechx_react-native-code-push').projectDir = new File(rootProject.projectDir, '../node_modules/@innotechx/react-native-code-push/android/app')

include ':react-native-gesture-handler'
project(':react-native-gesture-handler').projectDir = new File(rootProject.projectDir, '../node_modules/react-native-gesture-handler/android')

include ':react-native-reanimated'
project(':react-native-reanimated').projectDir = new File(rootProject.projectDir, '../node_modules/react-native-reanimated/android')

include ':react-native-safe-area-context'
project(':react-native-safe-area-context').projectDir = new File(rootProject.projectDir, '../node_modules/react-native-safe-area-context/android')

include ':react-native-screens'
project(':react-native-screens').projectDir = new File(rootProject.projectDir, '../node_modules/react-native-screens/android')

include ':@react-native-community_masked-view'
project(':@react-native-community_masked-view').projectDir = new File(rootProject.projectDir, '../node_modules/@react-native-community/masked-view/android')

```

#### 4. AndroidManifest.xml

1 声明网络权限<br>
`<uses-permission android:name="android.permission.INTERNET" />`

2 添加开发者菜单<br>
`<activity android:name="com.facebook.react.devsupport.DevSettingsActivity" />`

3 network_security_config.xml(API level 28+)<br>

```
<application
  android:networkSecurityConfig="@xml/network_security_config">
</application>
```

在`/res/xml`中添加`network_security_config.xml`

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true" />
</network-security-config>
```

#### 4. 将兜底包放到 assets 目录下

#### 5. 根据 demo 改造原生代码

主要为加载 bundle、拆包、热更新的相关实现。

- JsLoaderUtil
- BaseReactActivity
- SpConfig

对 Application 的改造: 继承 ReactApplication。

```

public class MainApplication extends Application implements ReactApplication {

    public static Application app;

    private SpConfig sc;

    private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {

        private String moduleId = "basic";

        @Override
        public boolean getUseDeveloperSupport() {
            JsLoaderUtil.jsState.isDev = sc.getLoadRuntime();
            return sc.getLoadRuntime();
        }


        @Override
        protected List<ReactPackage> getPackages() {
        	// 这里要添加进去RNPackage
            return new ArrayList<>(Arrays.<ReactPackage>asList(
                    new MainReactPackage(),
                    new RNCMaskedViewPackage(),
                    new RNGestureHandlerPackage(),
                    new ReanimatedPackage(),
                    new SafeAreaContextPackage(),
                    new RNScreensPackage(),
                    new CodePush("Xzy1V0cWXWDuh7Bb7HogAraIAJLT4c266bc0c0", MainApplication.this, this.moduleId, BuildConfig.DEBUG, "http://code-push-server.qutoutiao.net/")
            ));
        }

        @Nullable
        @Override
        protected String getJSBundleFile() {
            return CodePush.getJSBundleFile(this.moduleId, "basic.android.bundle");
        }

        @Nullable
        @Override
        protected String getBundleAssetName() {
            return "basic/basic.android.bundle";
        }

        @Override
        protected String getJSMainModuleName() {
            return "index";
        }
    };

    @Override
    public ReactNativeHost getReactNativeHost() {
        return mReactNativeHost;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        sc = new SpConfig(this);
        app = this;
        SoLoader.init(this, /* native exopackage */ false);
    }
}

```

当前实现是基于基类`BaseReactActivity`，这样加载 RN module 的业务组件只需要继承`BaseReactActivity`即可。

```
public class Business1Activity extends BaseReactActivity {
    @Override
    protected String getBundleName() {
        return "act_618.android.bundle";
    }

    @Nullable
    @Override
    protected String getMainComponentName() {
        return "act_618";
    }

    @Override
    protected String getModuleId() {
        return "buz_act_618";
    }
}

```

**注意**： 如果 Android 项目的根目录名就是 android，才可以使用自动引用，就不需要 settings.gradle 与 app build.gradle 里做手动引用依赖，相应的 MainApplication 的 add package 也需要修改。

**CodePush 相关**：[相关配置参考](https://developer.qutoutiao.net/wiki/#/8/README?id=android)

### 2 IOS

#### 1. Podfile 修改

将以下 Podfile 内容配置到你的 Podfile 文件里，`pod install`

```
platform :ios, '9.0'
require_relative '../node_modules/@react-native-community/cli-platform-ios/native_modules'

# target的名字一般与你的项目名字相同
target 'reactnative_multibundler' do

  # 'node_modules'目录一般位于根目录中
  # 但是如果你的结构不同，那你就要根据实际路径修改下面的`:path`
  pod 'FBLazyVector', :path => "../node_modules/react-native/Libraries/FBLazyVector"
  pod 'FBReactNativeSpec', :path => "../node_modules/react-native/Libraries/FBReactNativeSpec"
  pod 'RCTRequired', :path => "../node_modules/react-native/Libraries/RCTRequired"
  pod 'RCTTypeSafety', :path => "../node_modules/react-native/Libraries/TypeSafety"
  pod 'React', :path => '../node_modules/react-native/'
  pod 'React-Core', :path => '../node_modules/react-native/'
  pod 'React-CoreModules', :path => '../node_modules/react-native/React/CoreModules'
  pod 'React-Core/DevSupport', :path => '../node_modules/react-native/'
  pod 'React-RCTActionSheet', :path => '../node_modules/react-native/Libraries/ActionSheetIOS'
  pod 'React-RCTAnimation', :path => '../node_modules/react-native/Libraries/NativeAnimation'
  pod 'React-RCTBlob', :path => '../node_modules/react-native/Libraries/Blob'
  pod 'React-RCTImage', :path => '../node_modules/react-native/Libraries/Image'
  pod 'React-RCTLinking', :path => '../node_modules/react-native/Libraries/LinkingIOS'
  pod 'React-RCTNetwork', :path => '../node_modules/react-native/Libraries/Network'
  pod 'React-RCTSettings', :path => '../node_modules/react-native/Libraries/Settings'
  pod 'React-RCTText', :path => '../node_modules/react-native/Libraries/Text'
  pod 'React-RCTVibration', :path => '../node_modules/react-native/Libraries/Vibration'
  pod 'React-Core/RCTWebSocket', :path => '../node_modules/react-native/'

  pod 'React-cxxreact', :path => '../node_modules/react-native/ReactCommon/cxxreact'
  pod 'React-jsi', :path => '../node_modules/react-native/ReactCommon/jsi'
  pod 'React-jsiexecutor', :path => '../node_modules/react-native/ReactCommon/jsiexecutor'
  pod 'React-jsinspector', :path => '../node_modules/react-native/ReactCommon/jsinspector'
  pod 'ReactCommon/jscallinvoker', :path => "../node_modules/react-native/ReactCommon"
  pod 'ReactCommon/turbomodule/core', :path => "../node_modules/react-native/ReactCommon"
  pod 'Yoga', :path => '../node_modules/react-native/ReactCommon/yoga'

  pod 'DoubleConversion', :podspec => '../node_modules/react-native/third-party-podspecs/DoubleConversion.podspec'
  pod 'glog', :podspec => '../node_modules/react-native/third-party-podspecs/glog.podspec'
  pod 'Folly', :podspec => '../node_modules/react-native/third-party-podspecs/Folly.podspec'

  pod 'CodePush', :path => '../node_modules/@innotechx/react-native-code-push'

  pod 'SSZipArchive'

  use_native_modules!
end
```

#### 2. 将兜底包放到项目里

#### 3. 根据 demo 改造原生代码

主要为加载 bundle、拆包、热更新的相关实现。

- ScriptLoadUtil
- ReactController
- RctBridge

```
1、暴露RCTBridge的executeSourceCode方法，做法为将本项目中的RCTBridge添加到自己的工程

2、事先加载基础包：

jsCodeLocation = [[NSBundle mainBundle] URLForResource:@"platform.ios" withExtension:@"bundle"];
bridge = [[RCTBridge alloc] initWithBundleURL:jsCodeLocation
                             moduleProvider:nil
                              launchOptions:launchOptions];

3、加载业务包：

NSURL *jsCodeLocationBuz = [[NSBundle mainBundle] URLForResource:bundleName withExtension:@"bundle"];
  NSError *error = nil;
  NSData *sourceBuz = [NSData dataWithContentsOfFile:jsCodeLocationBuz.path
                                         options:NSDataReadingMappedIfSafe
                                           error:&error];
  [bridge.batchedBridge executeSourceCode:sourceBuz sync:NO];

4、创建RCTRootView，并绑定业务代码

RCTRootView* view = [[RCTRootView alloc] initWithBridge:bridge moduleName:moduleName initialProperties:nil];

```

**CodePush 相关**：[相关配置参考](https://developer.qutoutiao.net/wiki/#/8/README?id=ios)

---

## ReactNative 拆包与多业务开发

1. 创建业务<br>
   `npm run create`会在`/src/modules`里创建业务模块。

2. 开发与 debug<br>
   `npm run start`开启 metro packager，然后在集成好的 Native 项目查看效果（或者打开 RNHybrid Demo 查看效果）

3. 打包<br>
   通过`npm run bundler`打包基础包与业务包。 1. 基础包配置<br>


    ```
    	基础包入口 - basic.js
    	import React from 'react'
    	import { Text } from 'react-native'
    	import 'react-native'
    	import 'mobx'
    	import 'mobx-react'
    	import 'mobx-react-lite'
    	import 'react-native-gesture-handler'
    	import 'react-native-safe-area-context'
    	import 'react-native-reanimated'
    	import 'react-native-screens'

    	import '@innotechx/react-native-code-push'

    	import '@react-native-community/masked-view'
    	import '@react-navigation/stack'
    	import '@react-navigation/native'

    	const wrap = require('lodash.wrap')

    	Text.render = wrap(Text.render, function(func, ...args) {
    	  const originText = func.apply(this, args)
    	  return React.cloneElement(originText, { allowFontScaling: false })
    	})


    ```

    基础包会将在入口配置的node_modules相关的引用与全局公共配置到basic.bundle.js

    ```
    {
      name: 'basic.js', // 模块入口文件名称
      isBase: true, // 是否是基础模块
      isAll: false,
      metroConfig: 'metro_basic.config.js', // 模块metro 配置文件，这个必须在项目根目录，rn 的 metro 只读取根目录文件
      bundleName: {
        android: 'basic.android.bundle', // 输出名称
        ios: 'basic.bundle',
      },
    },
    ```

    2. 业务包

    业务包配置

    ```
    {
      name: 'buz_act_618.js', // 模块入口文件名称
      isBase: false, // 是否是基础模块
      metroConfig: 'metro_buz.config.js',
      bundleName: {
        android: 'act_618.android.bundle',
        ios: 'act_618.bundle',
      },
    },
    ```

通过 npm run bundler 打包会在 bundles 路径生成相应 bundle.

4.  上传包到平台
    1.  `npm run bundler:upload`
    2.  CI/CD（待完善）

**大前端 RN 平台**
[测试](http://fe-qa.qttcs3.cn/)
[生产](http://fe.qutoutiao.net/)

---
