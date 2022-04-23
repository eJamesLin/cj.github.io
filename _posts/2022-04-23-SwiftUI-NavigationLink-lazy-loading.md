---
layout: post
title:  "SwiftUI NavigationLink lazy loading"
date:   2022-04-23 +0800
tags:   [iOS, SwiftUI]
---

紀錄一個 SwiftUI 的坑及解法。

`NavigationLink` 的 destination 會立即被創建，在還沒有實際按下按鈕之前。
此時若有做些複雜的計算則會被立即執行... (依照 SwiftUI 的思維邏輯應該延後此複雜操作)。

```swift
struct ContentView: View {
    var body: some View {
        NavigationView {
            NavigationLink {
                // Create immediately.
                // What if need some expensive computation here...
                Text("detail view")
            } label: {
                Text("show detail view")
            }
        }
    }
}
```

### 解法 1 : onAppear
在 destination 出現時才執行複雜的操作。
另外要注意 View 反覆多次出現的情況。
```swift
struct ContentView: View {
    var body: some View {
        NavigationView {
                NavigationLink(
                    // Destination created immediately.
                    destination: {
                        Text("Detail View")
                            .onAppear {
                                // do expensive computation at onAppear
                            }
                    },
                    label: { Text("show detail view") }
                )
        }
    }
}
```

### 解法 2 : Lazy loading
View 的 body 在被顯示的時候才會執行，
所以可透過另一個 Wrapper View 把 destination 的創建，用 closure 傳入，延後到 Wrapper View 的 body 被執行時。

```swift
struct ContentView: View {
    var body: some View {
        NavigationView {
            NavigationLink(
                destination: { LazyView(Text("Detail View")) },
                label: { Text("show detail view") }
            )
        }
    }
}

struct LazyView<Content: View>: View {
    let build: () -> Content
    init(_ build: @autoclosure @escaping () -> Content) {
        self.build = build
    }
    var body: Content {
        build()
    }
}
````

使用 `@autoclosure`，將 destination 作為參數傳入，使用起來非常簡便。

### Reference
- <https://twitter.com/chriseidhof/status/1144242544680849410>
