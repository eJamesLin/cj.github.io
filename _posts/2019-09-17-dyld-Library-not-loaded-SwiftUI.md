---
layout: post
title:  SwiftUI dyld Library not loaded
date:   2019-09-17 +0800
tags:   [iOS, SwiftUI]
---
紀錄一個踩到的 Apple Known Issue。

因為 SwiftUI 是 iOS 13 以上才支援，所以使用在既有的專案要加上版本的判斷: `@available`。

```swift
@available(iOS 13.0, *)
struct ContentView: View {
    var body: some View {
        Text("Hello")
    }
}
```

即使加上了這個保護，但卻在 iOS 12 的實機遇到了 Crash，
查了以後才知道這是個已列在 Release Notes 的問題。

解法為加上 `-weak_framework SwiftUI` flag to the `Other Linker Flags` setting in the `Build Settings` tab。

![](/assets/2019/SwiftUI-weak-framework.png)

<https://developer.apple.com/documentation/ios_ipados_release_notes/ios_13_release_notes>

> Apps containing SwiftUI inside a Swift package might not run on versions of iOS earlier than iOS 13. (53706729)
> Workaround: When back-deploying to an OS which doesn't contain the SwiftUI framework, add the -weak_framework SwiftUI flag to the Other Linker Flags setting in the Build Settings tab. See Frameworks and Weak Linking for more information on weak linking a framework. This workaround doesn't apply when using dynamically linked Swift packages which import SwiftUI.

