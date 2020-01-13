---
layout: post
title: PromiseKit 入门指南
excerpt_separator: "<!--more-->"
categories:
  - iOS
tags:
  - iOS
  - Swift
  - 开源框架
  - PromiseKit
last_modified_at: 2020-01-13T23:27:01-05:00
---

# 开始使用

## then 与 done

以下是一个经典的 promise 链:

```swift
firstly {
    login()
}.then { creds in
    fetch(avatar: creds.user)
}.done { image in
    self.imageView = image
}
```

如果这段代码使用闭包回调的话，会长成这个样子:

```swift
login { creds, error in
    if let creds = creds {
        fetch(avatar: creds.user) { image, error in
            if let image = image {
                self.imageView = image
            }
        }
    }
}
```

`then` 函数是另外一个构建闭包回调的方法，但这种做法要比之前的 `completion handler` 方法好很多。它主要帮助我们更好的阅读这段代码。上面的 promise 链是非常容易阅读和理解的: 一个异步操作引出另一个异步操作。这段伪代码与程序代码非常接近，我们很容易理解这段 Swift 代码的当前含义。

`done` 函数和 `then` 含义是一样的，但是 `done` 没有返回一个承诺链。它常常被用于“成功链”的结尾。以上，我们可以看到在 `done` 函数中接受一个最终的 image 对象，并且被用来初始化我们的 UI 操作。

让我们来对比以下两种方法: 

```swift
func login()->Promise<Creds>

//对比：

func login(completion:(Creds?,Error?)->Void) //额。这边有两个可选值
```

这个两者区别在于使用了前者使用了`Promise`包装了Creds对象，你的方法返回了`Promise`对象而不是一个回调函数。每一个在响应链中的回调函数都会返回一个`Promise`对象。`Promise`对象中定义`then`方法，该方法在继续链之前等待前一个`Promise`任务的完成。`Promise`任务链会以程序化的方式解决问题，一次完成一个任务。

一个 `Promise` 代表了一个未来的异步任务。它用泛型语法包装了一个对象的真实类型。比如，在以上的例子中， `login` 是一个方法返回了一个 `Promise` 对象，但是他的真实类型是一个 `Creds` 的实例。

> *注意*：`done`是在PromiseKit 5中新特性。我们之前定义了`then`的变体，没有要求你返回一个`Promise`对象。不幸的是，这种惯例经常混淆Swift使用者，并且导致了奇怪且难以调试的错误消息。它使用使用PromiseKit是一件痛苦的事情。`done`的引入让你在使用了promise链的时候，编译器可以很便捷的给类型提示信息。

你可能会注意到与回调函数模式不同的是，promise 链模式似乎忽略了错误。不是这种情况！事实上，与此相反: promise 链模式的错误处理更加容易处理，更加难以忽略。

## catch

使用 promise 链模式，错误会沿着链逐级传递，确保你的应用程序代码正常健壮，代码逻辑清晰明确:

```swift
firstly{
        login()
}.then{ creds in
        featch(avatar:creds.user)
}.done{ image in
        self.imageView = image  
}.catch{
        //整个promise链的任何错误都归于此
}
```

> 如果你忘了"抓住"链条，Swift 会发出警告。我们稍后会详细讨论这个问题。

每个 promise 链都表示单个异步任务。如果任务执行失败，其 promise 将被 rejected。包含被拒绝 promise 链将跳出所有后续的 `then` 任务。将执行下一个 `catch` 任务。（严格地说，所有后续 `catch` 回调都被执行。)

为了好玩，让我们将此模式与闭包模式（ `comletion handler` 模式）进行比较:

```swift
func handle(error: Error) {
    //…
}

login { creds, error in
    guard let creds = creds else { return handle(error: error!) }
    fetch(avatar: creds.user) { image, error in
        guard let image = image else { return handle(error: error!) }
        self.imageView.image = image
    }
}
```

这段代码使用 `guard` 和一个统一的错误处理，但是 promise 链的可读性说明了一切。

## ensure

我们已经学会了异步组合。接下来让我们扩展更多的函数操作:

```swift
firstly {
    UIApplication.shared.isNetworkActivityIndicatorVisible = true
    return login()
}.then {
    fetch(avatar: $0.user)
}.done {
    self.imageView = $0
}.ensure {
    UIApplication.shared.isNetworkActivityIndicatorVisible = false
}.catch {
    //…
}
```

不论你的 promise 链输出是什么，成功或者失败，你的 `ensure` 回调永远会被执行的。

我们将此模式与它的闭包回调模式进行对比：

```swift
UIApplication.shared.isNetworkActivityIndicatorVisible = true

func handle(error: Error) {
    UIApplication.shared.isNetworkActivityIndicatorVisible = false
    //…
}

login { creds, error in
    guard let creds = creds else { return handle(error: error!) }
    fetch(avatar: creds.user) { image, error in
        guard let image = image else { return handle(error: error!) }
        self.imageView.image = image
        UIApplication.shared.isNetworkActivityIndicatorVisible = false
    }
}
```

对于有些人来说，修改这段代码或者取消设置活动指示器，非常容易导致出现 bug。使用 promise 链模式，这类错误几乎不可能会发生：在不使用改模式的情况下，Swift 编译器不会给你编译提示。通过使用这种模式，您几乎不需要查看提交的代码。

> 提示：PromiseKit 为了这个函数可能在名字 `always` 和 `ensure` 之间反复无常地切换。 为此表示歉意。 我们做的很糟糕。

你还可以使用 `finally` 作为一种 `ensure` , 用于终止 promise 链并且没有返回值: 

```swift
spinner(visible: true)

firstly {
    foo()
}.done {
    //…
}.catch {
    //…
}.finally {
    self.spinner(visible: false)
}
```

## when

使用闭包回调模式，对对个异步操作做出反应不是写的很慢就是很难写出优雅的代码。

```swift
operation1 { result1 in
    operation2 { result2 in
        finish(result1, result2)
    }
}
```

这种编码方式使得代码目的不够清晰:

```swift
var result1: …!
var result2: …!
let group = DispatchGroup()
group.enter()
operation1 {
    result1 = $0
    group.leave()
}
operation2 {
    result2 = $0
    group.leave()
}
group.notify(queue: .main) {
    finish(result1, result2)
}
```

使用 Promise 模式会变得更简单:

```swift
firstly {
    when(fulfilled: operation1(), operation2())
}.done { result1, result2 in
    //…
}
```

`when` 接受 `Promise` 对象，等待他们解决，并返回包含结果的 `Promise` 对象。

与任何 `Promise` 链一样，如果任何 `Promise` 任务失败了，该链将调用下一个 `catch` 任务回调。

# PromiseKit 扩展工具包

当我们制作 `PromiseKit` 工具包时，我们知道我们只想使用 `Promise` 来实现异步行为。因此，只要有可能，我们就会为苹果的API提供扩展，根据 `Promise` 重新构建 API。例如:

```swift
firstly {
    CLLocationManager.promise()
}.then { location in
    CLGeocoder.reverseGeocode(location)
}.done { placemarks in
    self.placemark.text = "\(placemarks.first)"
}
```

要使用这些扩展，你需要安装如下 pod 库:

```swift
pod "PromiseKit"
pod "PromiseKit/CoreLocation"
pod "PromiseKit/MapKit"
```

所有这些扩展都可以在 PromiseKit 组织上找到。去那里看看有什么可用的，并阅读源代码和文档。每个文件和函数已被大量记录在案。

> 我们还为 `Alamofire` 等公共库提供扩展。

## 创建 Promises

标准扩展会让您走很长的路，但有时您仍然需要创建自己的`Promise`链。如果你使用额第三方库没有提供`Promise`扩展，或者你已经写好了自己的异步程序。不管怎样，添加`Promise`都很容易。如果你看一下标准扩展库，您将看到它使用下面描述的相同方法。

假设我们有以下方法：

```swift
func fetch(completion: (String?, Error?) -> Void)
```

我们怎么才能转换成一个 `Promise` 呢，这很简单：

```swift
func fetch() -> Promise<String> {
    return Promise { fetch(completion: $0.resolve) }
}
```

您可能会发现扩展版本更具可读性：

```swift
func fetch() -> Promise<String> {
    return Promise { seal in
        fetch { result, error in
            seal.resolve(result, error)
        }
    }
}
```

这个`seal`对象是“Promise”初始化器提供的，用于定义很多方法来处理多重闭包回调的问题的。它甚至可以用来处理各种罕见的情况，从而使您很容易对现有代码库的添加`Promise`扩展。

> 注意：我们试图让它只做Promise(fetch)，但是我们不能让这个更简单的模式在不需要额外消除Swift编译器歧义的情况下普遍工作。对不起，我们尝试过但没有成功。

> 注意：在PromiseKit 4中，这个初始化器为闭包提供了两个参数: `fulfill` 和 `reject`。PromiseKit 5和6为您提供了一个对象，该对象具有 `fulfill` 和 `reject` 方法，但也有方法解析的许多变体。通常，您只需传递要解析的回调函数参数给 `resolve`，并让Swift找出应用于特定情况的变体(如上面的示例所示)。

> 注意： `Guarantee` (下面)是一个稍微不同的初始化器(因为它们不能出错)，所以初始化器闭包的参数只是一个闭包。不是 `Resolver` 对象。因此，应该 `seal(value)` 而不是 `seal.fulfill(value)`。这是因为没有什么变化在 `Guarantee` 中是未知的，它们只能 `fulfill`。

## Guarantee<T>

从 PromiseKit 5 开始，我们就提供了 `Guarantee` 作为 `Promise` 的补充类。我们这样做是为了补充 Swift 强大的错误处理系统。

`Guarantee` 永远不会失败，所以它们不能被 `reject` 。一个很好的例子是 `after` :

```swift
firstly {
    after(seconds: 0.1)
}.done {
    // 没有办法添加“catch”，因为after不能失败。
}
```

如果你不终止一个常规的`Promise`链，Swift会对你做出编译警告。（不是一个`Guarantee`链）。您应该通过提供一个`catch`或一个`return`来消除这个警告。(在后一种情况下，你可以获得得到的promise链。)

尽可能使用`Guarantee`，以便代码在需要的地方有错误处理，在不需要的地方没有错误处理。

一般来说，您应该能够交替使用`Guarantee`和`Promise`，我们已经尽了最大的努力来确保这一点，所以如果您发现任何问题请及时给我们提issue。

如果您正在创建自己的`Guarantee`，那么语法将比`Promise`更简单

```swift
func fetch() -> Promise<String> {
    return Guarantee { seal in
        fetch { result in
            seal(result)
        }
    }
}
```

可以归结为：

```swift
func fetch() -> Promise<String> {
    return Guarantee(resolver: fetch)
}
```

## map, compactMap 等等

`then`向您提供前一个承诺的结果，并要求您返回另一个承诺。`map`提供了前面承诺的结果，并要求您返回一个引用类型或值类型。`compactMap`提供了前面承诺的结果，并要求您返回一个可选的。如果返回nil，则该链将会返回`PMKError.compactMap`错误。

> 理由：在PromiseKit 4之前， `then` 会处理所有这些情况，这是非常糟糕的设计。我们希望这些痛苦会随着新版本的 Swift 而逐步消失。然而，很明显，各种各样的痛点都会存在。事实上，作为库的作者，我们应该在API的命名级别消除歧义。因此，我们将当时的三种主要类型分为 `then`、 `map`和 `done`。在使用了这些新函数之后，我们意识到这在实践中要好得多，所以我们也添加了 `compactMap`(以 `Optional.compactMap`为模型)。

`compactMap`有助于快速组合承诺链。例如:

 ```swift
 firstly {
    URLSession.shared.dataTask(.promise, with: rq)
}.compactMap {
    try JSONSerialization.jsonObject($0.data) as? [String]
}.done { arrayOfStrings in
    //…
}.catch { error in
    // Foundation.JSONError if JSON was badly formed
    // PMKError.compactMap if JSON was of different type
}
```

> 提示:我们还为序列提供了您期望的大多数函数方法，例如 `map`、 `thenMap`、 `compactMapValues`、 `firstValue`等。

## get

我们提供了 `get` 函数，类似 `done`, 主要用于获取返回值。

```swift
firstly {
    foo()
}.get { foo in
    //…
}.done { foo in
    // same foo!
}
```

## tap

我们提供 `tap` 用于 debug。它与 `get` 类似，但是提供了 `Promise` 的 `Result<T>`，因此您可以检查此时promise链中的值，而不会产生任何副作用:

```swift
firstly {
    foo()
}.tap {
    print($0)
}.done {
    //…
}.catch {
    //…
}
```

# 补充

## firstly

我们在这一页上已经使用 `firstly` 函数好几次了，但是它到底是什么呢?事实上，它只是个语法糖。您并不真正需要它，但它有助于使您的 promise 链更具可读性。

```swift
firstly {
    login()
}.then { creds in
    //…
}
```

也可以这样做:

```swift
login().then { creds in 
		//...
}
```

这里有一个关键的理解:login()函数返回一个`Promise`，所有`Promise`都有一个`then`函数。`firstly`返回一个`Promise`，然后`then`也返回一个`Promise`！但是不要太担心这些细节。从学习这种模式开始。然后，当您准备好前进时，继续学习底层架构。

## when的变体

`when`是Promisekit中很有用的几个函数之一，因此我们提供了几个变体。

- 默认的`when`是`when(fulfilled:)`，您通常应该使用它。这种变体等待他所有承诺组件完成，但如果有任何一个承诺失败了，`when`也失败，因此promise链将会被“拒绝”继续执行。需要注意的是，所有的promise都在`when`中继续执行。promise无法控制它们所代表的任务。promise只是任务的分装。
- `when(resolved:)` 将会继续等待，即使它的一个或多个承诺组件失败了。when的这种变体产生的值是一个Result<T>的数组。因此，该变体要求其所有组件承诺具有相同的泛型类型。有关此限制，请参阅我们的高级模式指南。
- `race`变体允许您竞逐多个承诺。无论谁先完成都是结果。有关典型用法，请参阅高级模式指南。

# Swift 闭包的用法

Swift自动推断单行闭包的返回和返回类型。以下两种形式是相同的:

```swift
foo.then {
    bar($0)
}

// is the same as:

foo.then { baz -> Promise<String> in
    return bar(baz)
}
```

我们的文档经常为了清晰而省略返回值。

然而，这种简写既是福也是祸。您可能会发现，Swift编译器经常无法正确推断返回类型。如果您需要进一步的帮助，请参阅我们的故障排除指南。

> PromiseKit 5 中添加 `done`函数，我们成功地避免了在使用 PromiseKit 和 Swift 过程中的许多常见痛点。

# 延伸阅读

以上信息是在使用PromiseKit中的90%了。我们强烈建议阅读[API指南](https://links.jianshu.com/go?to=https%3A%2F%2Fmxcl.dev%2FPromiseKit%2Freference%2Fv6%2FClasses%2FPromise.html)。有许多简短的函数可能对您有帮助，上面所有的内容在源代码的中的概述都更加全面。

在Xcode中编码时，单击PromiseKit函数来访问该文档。

这里是一些最近的文章，文档基于PromiseKit 5+:

- [Using Promises - Agostini.tech](https://links.jianshu.com/go?to=https%3A%2F%2Fagostini.tech%2F2018%2F10%2F08%2Fusing-promisekit)

小心一些网上的参考文章，他们中的许多人提到PromiseKit版本都小于5，这里面有些API是不相同的(抱歉，但Swift多年来已经改变了很多，因此我们也不得不这么做)。