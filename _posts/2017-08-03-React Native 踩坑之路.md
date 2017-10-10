---
layout:     post
title:      "React Native 踩坑之路"
subtitle:   ""
date:       2017-07-18 12:00:00
author:     "Shinefy"
header-img: ""
---


## RN环境篇
---

- 使用[Homebrew](https://brew.sh/index_zh-cn.html)作为mac的主要包管理器。
- 使用Homebrew安装[Yarn](https://yarn.bootcss.com/)，Yarn用于替代npm。[这里有Yarn的常用命令](https://yarn.bootcss.com/docs/usage.html)。
 
 > Yarn是Facebook提供的替代npm的工具，可以加速node模块的下载。使用Yarn执行创建、初始化、更新项目、运行打包服务（packager）等任务。
 
 > 设置Yarn国内镜像源：
 yarn config set registry https://registry.npm.taobao.org --global
 yarn config set disturl https://npm.taobao.org/dist --global
 
 
- 安装node版本管理工具nvm,[这里有nvm的正确安装方式及常用命令](http://www.imooc.com/article/14617)。

- 使用nvm安装[node](http://nodejs.cn/)。

- 使用Yarn安装React Native的命令行工具（react-native-cli）:
  
  `yarn global add react-native-cli`

- 基本的环境搭建还应参考[RN中文网搭建环境教程](http://reactnative.cn/docs/0.45/getting-started.html)。


## TypeScript篇
---

- 使用[WebStorm](https://www.jetbrains.com/webstorm/)作为编写RN的主要IDE。

 > IDE Preferences:
 > 将JavaScript language version 设置为 React JSX。
 > 勾选enable TypeScript Compiler
 
	- 发现WebStorm的几个缺点：
 	 1. native语言IDE竟然都不支持高亮
 	 2. 不能在IDE里debug，不知道为什么，还得打开chrome调试
 
- VSCode编写RN的几个tips

  - [ React Native 开发准备的 VS Code 插件](http://blog.csdn.net/asce1885/article/details/71075432)
  - [安装react-native-tools插件代替chromDevTools调试的方法](http://blog.csdn.net/guoer9973/article/details/54669885) 
  - [自动编译ts文件的方法](https://github.com/zlumer/typescript-node-basic)：
     1. tsconfig内设置 `"watch": true`
     2. `cmd+shift+b`编译，选择tsconfig文件。
     3.  终端内找到tsconfig build 任务查看，done。

- [TypeScript官方教程的中文翻译](https://www.gitbook.com/book/zhongsp/typescript-handbook/details)

- [TypeScript优秀的入门教程](https://www.gitbook.com/book/xcatliu/typescript-tutorial/details)

- [TypeScript的type definitions
](https://segmentfault.com/a/1190000009129234)
 
 
## 项目构建篇
---

- RN的ios部分使用Swift构建方法：
	1. 删除RN项目ios目录下的所有文件
	2. 使用Xcode创建一个新的swift项目
	3. 将该项目内所有文件复制到RN的ios目录下
	4. `pod init  &&  pod install`
	5.  Podfile文件添加RN项目中node_modules目录下的React相关模块
	
		```java
			pod "Yoga", :path => "../node_modules/react-native/ReactCommon/yoga"
			pod "React", :path => "../node_modules/react-native", :subspecs => [
			'Core',
			'ART',
			'RCTActionSheet',
			'RCTAdSupport',
			'RCTGeolocation',
			'RCTImage',
			'RCTNetwork',
			'RCTPushNotification',
			'RCTSettings',
			'RCTText',
			'RCTVibration',
			'RCTWebSocket',
			'RCTLinkingIOS',
			'DevSupport',
			'BatchedBridge'
			]				
		```
		
	6. `pod init  &&  pod install`
	7.  入口文件AppDelegate.swift内添加以下代码，以加载jsbundle以及将index.ios.js作为根视图：
	
		```swift
		//import React
		func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
			let moduleName:String = "CommonDemo"
			let jsCodeLocation: NSURL
			
			jsCodeLocation = NSURL(string: "http://127.0.0.1:8081/index.ios.bundle?platform=ios&dev=true")!
			
			//jsCodeLocation = NSBundle.mainBundle().URLForResource("main", withExtension: "jsbundle")!
			
			let rootView: RCTRootView = RCTRootView(bundleURL: jsCodeLocation as URL!, moduleName: moduleName, initialProperties: nil, launchOptions: launchOptions)
			
			rootView.backgroundColor = UIColor.white
			
			let rootViewController = UIViewController()
			rootViewController.view = rootView
			
			window = UIWindow(frame: UIScreen.main.bounds)
			window?.rootViewController = rootViewController
			window?.makeKeyAndVisible()
			
			return true
    }
	```
	
 8. 在xcode内build，build succeeded。
 9. 切换至WebStorm，`yarn start` , `react-native run-ios` 
 
 > 参考资料
 [ReactNative-Cocoapods-Swift-Project项目搭建](http://blog.csdn.net/u011690583/article/details/51700662)
 [RN报错：":CFBundleIdentifier", Does Not Exist](https://github.com/facebook/react-native/issues/7308)
 [Xcode报错:double-conversion-1.1.5、boost_1_63_0文件被墙](https://github.com/facebook/react-native/issues/14423)，单独下载后覆盖至~/.rncache目录
 
 
- 更改React Native版本：
	- 在项目目录下使用`react-native -v`命令查看当前的版本
	- [安装react-native-git-upgrade工具模块](http://reactnative.cn/docs/0.45/upgrading.html#content),最快捷的方式更新RN的版本
	
 

## 链接原生库
---

 > 安装一个带原生依赖的RN库时，必须要将原生部分链接到iOS项目内，否则直接运行会报错。
 
 - [官方链接原生库教程](http://reactnative.cn/docs/0.46/linking-libraries-ios.html#content)
 	1. yarn add 某个带有原生依赖的库
 	2. react-native link 
 
 - 但是在实际使用的时候,每次使用`react-native link `命令自动link后，在xcode下运行项目直接就报错，提示`React/RCTBridgeModule.h not found` 等奇奇怪怪的问题，so必须得考虑手动链接然后再解决问题了。
  
   - 对于像[react-native-vector-icons](https://github.com/oblador/react-native-vector-icons)这样的带有原生依赖的库，因为依赖部分有`.podspec`格式的pod依赖库描述文件，那么可以直接在podfile内加入`pod 'RNVectorIcons', :path => "../node_modules/react-native-vector-icons"`,执行`pod install && pod update`，done。

   - 对于像[react-native-picker](https://github.com/beefe/react-native-picker)这样的带有原生依赖的库，依赖部分没有.podspec文件，那么我们可以自己创建，[创建podspec文件，为自己的项目添加pod支持](http://www.jianshu.com/p/d7d1942dd3f1):
     
     1. 在ios部分根目录下执行`pod spec create RCTBEEPickerManager(该proj名称)`
     2. 编辑内容，删除不必要的注释. (source_files条目一定要填对)
     3. 执行`pod lib lint` 修改Warning或者Error的地方，直到验证通过。
     4. podfile内加入`pod 'RCTBEEPickerManager', :path => "../node_modules/react-native-picker`,执行`pod install && pod update`，done。
     5.  发现每次`yarn add`以后自己建的podspec文件就没了，so，对yarn做什么设置呢？？     

		> pod成功后，在在XCode内的Development Pods下就能看到刚刚pod进去的原生依赖。编译xcode项目，发现还是提示错误`React/RCTBridgeModule.h not found`，这个时候再研究了一下，这个依赖的RCTBEEPickerManager.xcconfig配置文件内加入`FRAMEWORK_SEARCH_PATHS = $(inherited) "$PODS_CONFIGURATION_BUILD_DIR/React" "$PODS_CONFIGURATION_BUILD_DIR/Yoga"`，再build一下，duang~成功。	
 
 
## 创建iOS原生模块
---
	
- [这里是官方关于iOS原生模块的教程](http://reactnative.cn/docs/0.46/native-modules-ios.html#content)
- [ React Native iOS原生模块开发实战心得](http://blog.csdn.net/fengyuzhengfan/article/details/54691432)
- 创建Siwft iOS原生模块的步骤：
	1. 创建混编的桥接文件，引入RCTBridgeModule.h：`#import <React/RCTBridgeModule.h>`
	2. 创建原生模块KeychainHelper.swift，
		
		```typescript
			objc(KeychainHelper)
			public class KeychainHelper : NSObject {
		
				@objc
				private func initUUID(){
					let keychain = Keychain();
					if Keychain()["uuid"] == nil {
						if let uuidString = UIDevice.current.identifierForVendor?.uuidString {
						keychain["uuid"] = uuidString;
						}else{
						ToastManager.showShortToast("无法初始化保存UUID！")
						}
					}
				}
		
				@objc
				private func getUUID( _ resolver:RCTPromiseResolveBlock , rejecter :RCTPromiseRejectBlock){
						resolver([Keychain()["uuid"] ?? ""])
					}
			}
		```
	3. 创建KeychainHelperBridge.m文件，将模块注册到RN中。
		
		```objective-c
			
			#import <Foundation/Foundation.h>
			#import <React/RCTBridgeModule.h>


			@interface RCT_EXTERN_MODULE(KeychainHelper, NSObject)

			RCT_EXTERN_METHOD(initUUID)
			RCT_EXTERN_METHOD(getUUID :  (RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject);
			@end
			
		```
	4. 在RN中调用：
		
		```typescript

		const KeychainHelper = require("react-native").NativeModules.KeychainHelper;
		
		KeychainHelper.initUUID();
		
		async getuuid() {
			var uuid = await KeychainHelper.getUUID();
			this.setState({"uuid":uuid});
 	 	}

		```
	
- 几个必须注意的地方：

  - 注册模块时因为是使用oc导出的，导出的oc方法的第一个参数是忽略外部参数的，所以实现的swift部分需要对应`_`,否则RN调用会找不到这个方法报错。
  - 在所有的情况下RN和原生模块之前进行通信都是在异步的情况下进行的，所以如果想要RN调原生模块方法时有返回值，即原生向RN发送数据，需要通过Callbacks,更先进的，我们也可以通过Promises。具体可以看开头的教程。
	
