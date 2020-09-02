---
layout: post
title:  Flutter Plugins Ordering Issue and Solution
date:   2020-09-03 +0034
tags:   [iOS, Flutter]
---

### About
Flutter plugins 被註冊的順序，會影響到 plugin code 執行的順序，進而可能會造成第三方登入失敗，或無法正確地收到 deep link 的呼叫。
Android 及 iOS 的 Flutter 來自同一個 Engine，所以同樣都有載入順序的問題。
本篇想紀錄問題探詢的過程，以及解決的方法。

### Issue Intro
iOS 上，其中一個處理從外部呼叫 App 的 callback function 是 [application(_:open:options:)](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623112-application)。

常見的實作如下：

```swift
// sample code
func application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey: Any] = [:]) -> Bool {
    if FacebookLogin.handle(url) {
        return true // 處理 Facebook 登入
    } else if LineLogin.handle(url) {
        return true // 處理 LINE 登入
    } else if FirebaseDynamicLinks.handle(url) {
        return true // 處理 Firebase Dynamic Links
    } else {
        return AppRouter.handle(url) // 對這個 deeplink url，在 App 內部使用 webview 或使用新的 native 畫面呈現
    }
}
```

一個呼叫 App 起來的 url，只會屬於一種情況。通常會在處理完登入及特殊的 deeplink (ex: FirebaseDynamicLinks) 之後，才使用 App 內的 Router 來呈現這個 url。
此時 code 的順序有其必要，把呈現 url 放在最後。


而在 Flutter 中，plugin 會實作 FlutterPlugin 的 protocol [registerWithRegistrar:registrar](https://github.com/flutter/flutter/blob/856a90e67c9284124d44d2be6c785bacd3a1c772/packages/flutter_tools/templates/plugin/ios-swift.tmpl/Classes/pluginClass.m.tmpl)。
在其中，plugin 跟 Flutter 使用 `addApplicationDelegate:` 註冊，聲明自己要處理 `UIApplicationDelegate` 的呼叫。此時若多個 plugin 都有聲明，則這個順序是如何被決定的呢？

<!--more-->

Firebase Dynamic Links 的實作如下

```objc
// https://github.com/FirebaseExtended/flutterfire/blob/firebase_dynamic_links-v0.5.0+11/packages/firebase_dynamic_links/ios/Classes/FLTFirebaseDynamicLinksPlugin.m#L52
+ (void)registerWithRegistrar:(NSObject<FlutterPluginRegistrar> *)registrar {
  FlutterMethodChannel *channel =
      [FlutterMethodChannel methodChannelWithName:@"plugins.flutter.io/firebase_dynamic_links"
                                  binaryMessenger:[registrar messenger]];
  FLTFirebaseDynamicLinksPlugin *instance =
      [[FLTFirebaseDynamicLinksPlugin alloc] initWithChannel:channel];
  [registrar addMethodCallDelegate:instance channel:channel];
  [registrar addApplicationDelegate:instance];
  // ...
}
- (BOOL)application:(UIApplication *)application
            openURL:(NSURL *)url
            options:(NSDictionary<UIApplicationOpenURLOptionsKey, id> *)options {
  return [self checkForDynamicLink:url];
}
```

而 `uni_links` 這個幫忙處理 DeepLink 及 Universal Link 的 plugin，會在 native 把 url 接起來，然後傳給 Flutter，Flutter 開發者就可以專心使用 `Stream` 來處理進來的 url。

```objc
// https://github.com/avioli/uni_links/blob/master/ios/Classes/UniLinksPlugin.m#L58
- (BOOL)application:(UIApplication *)application
            openURL:(NSURL *)url
            options:(NSDictionary<UIApplicationOpenURLOptionsKey, id> *)options {
  self.latestLink = [url absoluteString];
  return YES;
}

- (void)setLatestLink:(NSString *)latestLink {
  static NSString *key = @"latestLink";

  [self willChangeValueForKey:key];
  _latestLink = [latestLink copy];
  [self didChangeValueForKey:key];

  if (_eventSink) _eventSink(_latestLink);
}
```

此時，同時有多個 plugin 跟 flutter 註冊要處理 UIApplicationDelegate，此時就會發生執行順序上的問題。很有可能 Firebase Dynamic Links 會無法被呼叫到，被其他 plugin 先攔截走，而沒有成功解析 Dynamic Links。又或許是登入的 url 被攔截走，而造成無法登入的問題。
實際跑了幾次後發現，這個執行順序是不固定的。
那要如何能保證 `Login` 以及 `Firebase Dynamic Links` 的執行順序在 `uni_links` 之前呢？

### Dive into Flutter Source Code
在 Flutter 的 Objective-C++ code 中，對於處理 incoming link 的 `applicationURL:openURL:options` 處理如下：
他會對所有的 `_delegates`，逐一地詢問是否要處理 url，如果已處理完成，則提早 return 跳離 for loop。

```objc++
// https://github.com/flutter/engine/blob/9f650edd14dd0d74acb3d6ad65eb794b1e4b27e3/shell/platform/darwin/ios/framework/Source/FlutterPluginAppLifeCycleDelegate.mm#L312
- (BOOL)application:(UIApplication*)application
            openURL:(NSURL*)url
            options:(NSDictionary<UIApplicationOpenURLOptionsKey, id>*)options {
  for (NSObject<FlutterApplicationLifeCycleDelegate>* delegate in _delegates) {
    if (!delegate) {
      continue;
    }
    if ([delegate respondsToSelector:_cmd]) {
      if ([delegate application:application openURL:url options:options]) {
        return YES;
      }
    }
  }
  return NO;
}
```

而這個 `_delegate` 則是在這個 `addDelegate:` 中被加入的。`_delegate` 的結構只是一個 PointerArray，他的順序是在 plugin 被 register 時，呼叫 `addDelegate:` 所固定下的。

```objc++
// https://github.com/flutter/engine/blob/9f650edd14dd0d74acb3d6ad65eb794b1e4b27e3/shell/platform/darwin/ios/framework/Source/FlutterPluginAppLifeCycleDelegate.mm#L98

@implementation FlutterPluginAppLifeCycleDelegate {
  NSMutableArray* _notificationUnsubscribers;
  UIBackgroundTaskIdentifier _debugBackgroundTask;

  // Weak references to registered plugins.
  NSPointerArray* _delegates;
}

...

- (void)addDelegate:(NSObject<FlutterApplicationLifeCycleDelegate>*)delegate {
  [_delegates addPointer:(__bridge void*)delegate];
  ...
}
```

而 Flutter plugin 註冊的順序則是由自動產生的檔案 `GeneratedPluginRegistrant.m` 所決定的。
```objc
//
//  Generated file. Do not edit.
//
...
+ (void)registerWithRegistry:(NSObject<FlutterPluginRegistry>*)registry {
  [FlutterLineSdkPlugin registerWithRegistrar:[registry registrarForPlugin:@"FlutterLineSdkPlugin"]];
  [UniLinksPlugin registerWithRegistrar:[registry registrarForPlugin:@"UniLinksPlugin"]];
  [FLTFirebaseDynamicLinksPlugin registerWithRegistrar:[registry registrarForPlugin:@"FLTFirebaseDynamicLinksPlugin"]];
}
```

而 Flutter 是如何產生這個 `GeneratedPluginRegistrant.m` 呢？

```
// https://github.com/flutter/flutter/blob/39d7a019c150ca421b980426e85b254a0ec63ebd/packages/flutter_tools/gradle/flutter.gradle#L248
* The plugins are added to pubspec.yaml. Then, upon running `flutter pub get`,
* the tool generates a `.flutter-plugins` file, which contains a 1:1 map to each plugin location.
```
當執行了 `flutter pub get`，去抓取 flutter plugin 時，會產生一份 `.flutter-plugins`，而 `GeneratedPluginRegistrant.m` 即是依照這份文字檔的順序所產生。

```dart
// https://github.com/flutter/flutter/blob/fa4d31b31/packages/flutter_tools/lib/src/plugins.dart#L464

const String _objcPluginRegistryImplementationTemplate = '''//
//  Generated file. Do not edit.
//
#import "GeneratedPluginRegistrant.h"
{{#plugins}}
#import <{{name}}/{{class}}.h>
{{/plugins}}
@implementation GeneratedPluginRegistrant
+ (void)registerWithRegistry:(NSObject<FlutterPluginRegistry>*)registry {
{{#plugins}}
  [{{prefix}}{{class}} registerWithRegistrar:[registry registrarForPlugin:@"{{prefix}}{{class}}"]];
{{/plugins}}
}
@end
''';

// called by `_writeIOSPluginRegistrant`
```

而這個 `.flutter-plugins` 的順序，因為在 `findPlugins` 中對 Map `packages` 使用了 `forEach`，所以會 `plugins` 的順序無法固定。

```dart
List<Plugin> findPlugins(FlutterProject project) {
  final List<Plugin> plugins = <Plugin>[];
  Map<String, Uri> packages;
  try {
    final String packagesFile = fs.path.join(
      project.directory.path,
      PackageMap.globalPackagesPath,
    );
    packages = PackageMap(packagesFile).map;
  } on FormatException catch (e) {
    printTrace('Invalid .packages file: $e');
    return plugins;
  }
  packages.forEach((String name, Uri uri) {
    final Uri packageRoot = uri.resolve('..');
    final Plugin plugin = _pluginFromPubspec(name, packageRoot);
    if (plugin != null) {
      plugins.add(plugin);
    }
  });
  return plugins;
}
```

因為無法固定 plugin 的順序，所以每次重新 run project，在相同的 input 下，有可能會因為不同的 plugin 載入順序，而產生不同的結果。

### Solution
- 寫了一個簡短地 script，在 Xcode compile 前，幫忙排序及指定 `GeneratedPluginRegistrant.m` 裡面的 plugin 註冊順序。
- 程式碼在 github：<https://github.com/eJamesLin/FlutterPluginSort>
- 跑完之後，把 `GeneratedPluginRegistrant.m` 也加入 `git` version-control 裡面即可
- 如果官方已有更新，或各位有更建議的解法，非常歡迎提醒我囉，感謝~
