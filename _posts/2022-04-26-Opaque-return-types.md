---
layout: post
title:  "Opaque return types"
date:   2022-04-26 +0800
tags:   [iOS, SwiftUI]
---

紀錄 Opaque types 的一些資訊。

```swift
struct ContentView: View {
    var body: some View {
        Text("hello world")
    }
}
```
```swift
public protocol View {
    /// The type of view representing the body of this view.
    associatedtype Body : View

    /// The content and behavior of the view.
    @ViewBuilder var body: Self.Body { get }
}
```

此 ContentView 定義 body 的 [Read-Only computed properties](https://docs.swift.org/swift-book/LanguageGuide/Properties.html)，getter 回傳 `some View`，某個符合 View protocol 的型別，稱為 Opaque type。

官網定義：
> A function or method with an opaque return type hides its return value’s type information. Instead of providing a concrete type as the function’s return type, the return value is described in terms of the protocols it supports. Hiding type information is useful at boundaries between a module and code that calls into the module, because the underlying type of the return value can remain private. Unlike returning a value whose type is a protocol type, opaque types preserve type identity—the compiler has access to the type information, but clients of the module don’t.

由以上說明可知，Opaque return type 可以隱藏型別資訊。

我們可以藉著以下方法來得知真實的 body 型別資訊。在這個例子中，真實回傳的型別是 `VStack<Button<Text>>`。
```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Button("print my type") {
                print(type(of: self.body))
                // VStack<Button<Text>>
            }
        }
    }
}
```

<!--more-->

將 `VStack<Button<Text>>` 換掉原本的 some View，回傳真正的 concrete type，也可以成功編譯。
```swift
struct ContentView: View {
    var body: VStack<Button<Text>> {
        VStack {
            Button("print my type") {
                print(type(of: self.body))
            }
        }
    }
}
```

如果在此嘗試將 some View 換成 View，則會撞到 PAT (Protocol with Associated Type) 的限制。
```swift
// Compile Error: Type 'ContentView' does not conform to protocol 'View'
// Cannot infer 'Body' = 'View' because 'View' as a type cannot conform to protocols;
// did you mean to use an opaque result type?
struct ContentView: View {
    var body: View {
        // Compile Error: 
        // Protocol 'View' can only be used as a generic constraint 
        // because it has Self or associated type requirements
        Text("hello")
    }
}
```
```swift
public protocol View {
    /// The type of view representing the body of this view.
    associatedtype Body : View

    /// The content and behavior of the view.
    @ViewBuilder var body: Self.Body { get }
}
```

### Difference Between Opaque Type and Generic Type
- Generic Type: Function Caller 決定回傳型別
- Opaque Type: Function 自己決定回傳型別

#### Generic Type:
> Generic types let the code that calls a function pick the type for that function’s parameters and return value in a way that’s abstracted away from the function implementation.

```swift
func max<T>(_ x: T, _ y: T) -> T where T: Comparable { ... }

// usage example
// max(Int(1), Int(2))
// max(Float(1), Float(2))
```

#### Opaque Type
> You can think of an opaque type like being the reverse of a generic type. 

再此使用前面的例子，其真實的回傳型別是 `VStack<Button<Text>>`
```swift
struct ContentView: View {
    var body: some View {
        VStack {
            Button("print my type") {
                print(type(of: self.body))
                // VStack<Button<Text>>
            }
        }
    }
}
```

### Difference Between Opaque Type and Protocol Type
官方定義：
> Returning an opaque type looks very similar to using a protocol type as the return type of a function, but these two kinds of return type differ in whether they preserve type identity. 

> An opaque type refers to one specific type, although the caller of the function isn’t able to see which type; 

> A protocol type can refer to any type that conforms to the protocol.

- Opaque type 規定回傳的型別，在所有的執行路徑中，都必須回傳相同型別。
- Protocol type 則可以回傳任意 conforms to 該 protocol 的型別。

```swift
protocol Shape {}
struct Triangle: Shape {}
struct Rectangle: Shape {}

enum ShapeEnum {
    case triangle
    case rectangle
}

// Compile Success: Could return any type confirms to protocol
func createProtocolType(shapeEnum: ShapeEnum) -> Shape {
    switch shapeEnum {
    case .triangle: return Triangle()
    case .rectangle: return Rectangle()
    }
}

// Compile Success: All path return the same type
func createOpaqueType(shapeEnum: ShapeEnum) -> some Shape {
    switch shapeEnum {
    case .triangle: return Triangle()
    default: return Triangle()
    }
}

// Compile Error: Function declares an opaque return type, 
// but the return statements in its body do not have matching underlying types
func createOpaqueTypeCompileError(shapeEnum: ShapeEnum) -> some Shape {
    switch shapeEnum {
    case .triangle: return Triangle()
    case .rectangle: return Rectangle()
    }
}
```

### Closure cannot return Opaque Type
若在 closure 中使用 Opaque Type，會遇到 compile error: 
> 'some' types are only implemented for the declared type of properties and subscripts and the return type of functions

所以在 [前一篇的 LazyView](../SwiftUI-NavigationLink-lazy-loading/) 中，要將 View 的建立放到 closure 中，從而在 LazyView 的 body 中再去建立 View 的話，就無法使用 Opaque Type。所以此處使用了 generic 的方法，來限定 closure return type。

```swift
struct LazyView<Content: View>: View {
    let build: () -> Content
    init(_ build: @autoclosure @escaping () -> Content) {
        self.build = build
    }
    var body: Content {
        build()
    }
}
```

### ViewBuilder
前面提到 Opaque Type 限定回傳的 type 是同一種，但為何 ContentView 可以有 if-else 的寫法，回傳不同 type?

```swift
struct ContentView: View {
    @State var isLoggedIn = false

    @ViewBuilder
    func createView() -> some View {
        if isLoggedIn {
            Text("Logged In")
        }
        else {
            Button("Click to Login") {
                isLoggedIn.toggle()
            }
        }
    }

    var body: some View {
        let view = createView()

        // print _ConditionalContent<Text, Button<Text>>
        print(type(of: view))

        return view
    }
}
```

從以上寫法可以觀察到，透過 `@ViewBuilder` 修飾 function 之後，回傳的 concrete type 是 `_ConditionalContent<Text, Button<Text>>`。

而 View protocol 有定義 body 使用 `@ViewBuilder`，所以在 body 中可使用多行的 command，並回傳單一的 concrete type。

```swift
public protocol View {
    /// The type of view representing the body of this view.
    associatedtype Body : View

    /// The content and behavior of the view.
    @ViewBuilder var body: Self.Body { get }
}
```

ViewBuilder 的定義如下

```swift
/// You typically use ``ViewBuilder`` as a parameter attribute for child
/// view-producing closure parameters, allowing those closures to provide
/// multiple child views. For example, the following `contextMenu` function
/// accepts a closure that produces one or more views via the view builder.
///
///     func contextMenu<MenuItems: View>(
///         @ViewBuilder menuItems: () -> MenuItems
///     ) -> some View
///
/// Clients of this function can use multiple-statement closures to provide
/// several child views, as shown in the following example:
///
///     myView.contextMenu {
///         Text("Cut")
///         Text("Copy")
///         Text("Paste")
///         if isSymbol {
///             Text("Jump to Definition")
///         }
///     }
///
@resultBuilder public struct ViewBuilder {

    /// Builds an empty view from a block containing no statements.
    public static func buildBlock() -> EmptyView

    /// Passes a single view written as a child view through unmodified.
    ///
    /// An example of a single view written as a child view is
    /// `{ Text("Hello") }`.
    public static func buildBlock<Content>(_ content: Content) -> Content where Content : View
}

extension ViewBuilder {

    /// Provides support for “if” statements in multi-statement closures,
    /// producing an optional view that is visible only when the condition
    /// evaluates to `true`.
    public static func buildIf<Content>(_ content: Content?) -> Content? where Content : View

    /// Provides support for "if" statements in multi-statement closures,
    /// producing conditional content for the "then" branch.
    public static func buildEither<TrueContent, FalseContent>(first: TrueContent) -> _ConditionalContent<TrueContent, FalseContent> where TrueContent : View, FalseContent : View

    /// Provides support for "if-else" statements in multi-statement closures,
    /// producing conditional content for the "else" branch.
    public static func buildEither<TrueContent, FalseContent>(second: FalseContent) -> _ConditionalContent<TrueContent, FalseContent> where TrueContent : View, FalseContent : View
}
```

### Reference
- <https://docs.swift.org/swift-book/LanguageGuide/OpaqueTypes.html>
- <https://www.swiftbysundell.com/articles/opaque-return-types-in-swift/>
- <https://www.hackingwithswift.com/forums/swift/possible-to-return-opaque-type-from-closure/3334>

