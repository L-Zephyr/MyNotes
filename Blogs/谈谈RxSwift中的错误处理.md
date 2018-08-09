RxSwift中提供了多种不同的错误处理操作符，它们可以在链式操作中相互组合以实现复杂的处理逻辑，下面先简单介绍一下RxSwift提供的错误处理操作，然后通过一些具体的例子来看看如何在实际项目中应用。这里不会详细介绍RxSwift，阅读前需要对Rx的基础有一定了解。

## 错误处理操作符

### throw

Rx中许多操作符中的闭包签名都是带有`throws`修饰符的，比如说这是`map`方法的签名：

```swift
func map<R>(_ transform: @escaping (E) throws -> R) -> Observable<R>
```

我们可以在这样的操作中抛出错误，抛出的错误会沿着链式操作一步步向下传递，比如下面的代码：

```swift
Observable.of(3, 2, 1) // 创建一个包含3个事件的Observable
    .map { (n) -> Int in
        if n < 2 {
            throw CustomError.tooSmall // 1. 抛出一个自定义的错误
        } else {
            return n * 2
        }
    }
	.subscribe { event in
		// 2. 这里会收到一个 CustomError.tooSmall 类型的error
    }
	...
```

当数字小于2时，在`map`中`throw`一个自定义的错误类型，这个错误会被传递到下面的`subscribe`中。

### catchError

RxSwift可以在链式操作中捕获错误，不管是`Observable`自己产生的错误还是用户throw的错误，都可以在`catchError`操作中进行处理，接着上面`map`的代码：

```swift
Observable.of(3, 2, 1)
    .map { 
        ...
    }
    .catchError({ (error) -> Observable<Int> in
        if case CustomError.tooSmall = error {
            return .just(2) // 1. 在捕获到tooSmall错误后返回一个2
        }
        return .error(error) // 2. 其他错误不处理，继续沿着操作链传递
    })
	.subscribe { event in
		// 3. 当发生tooSmall错误时这里会收到2，最终的结果是 3, 2, 2
    }
	...
```

这样的处理方式接近于语言本身的`try…catch`机制，使用起来十分方便。

### retry

`retry`提供了出错重试的操作，在一个`Observable`后面加上`retry`，当这个`Observable`出错的时候订阅方不会收到`.error`事件，而是重新订阅这个Observable，这通常意味着这个Observable创建事件相关的操作都会重新执行一次，比如说这是一个网络请求相关的Observable，“重新订阅”会重发相关的网络请求：

```swift
moyaProvider.rx.request(.customData)
	.retry() // 1. 当发生错误时重试请求
	.subscribe { 
    	// 2. 重试的过程对于订阅者来说是不可见的，请求成功后这里才会接收到事件
    }
```

`retry`方法还可以带一个`Int`类型的参数，表示重试的最大次数。

### retryWhen

`retryWhen`就是带条件的`retry`，可以在特定条件下才进行重试。`retryWhen`的方法签名比较特别，它的闭包中接受的参数不是一个简单的`Error`类型，而是一个`Observable<Error>`类型，使用方法如下：

```swift
Observable.of(3, 2, 1)
    .map { 
        ...
    }
	// 1. retryWhen不关心返回的是什么类型，只关心事件本身，所以直接用Observable<()>即可
	.retryWhen({ (errorObservable) -> Observable<()> in
       	// 2. 用flatMap将其转换成其他类型的Observable
        return errorObservable.flatMap({ (error) -> Observable<()> in
            if case CustomError.tooSmall = error {
                return .just(()) // 3. 返回一个next事件表示重试
            }
            return .error(error) // 4. 继续返回error表示不处理
        })
    })
```

闭包返回的`Observable`可以是任意类型，因为`retryWhen`只关心`Observable`中的事件本身，不关心其中承载的数据类型，所以这里直接用一个空类型即可，如果需要重试的话就将一个带有`.next`事件的`Observable`返回。

`retryWhen`这样设计的一个优点是在出错的时候可以将它重试的逻辑跟另外一个`Observable`事件流关联起来（后面我会演示一个例子）。但是在上面这样一个简单的场景中，使用起来未免过于麻烦了，这里可以做一个简单的封装，提供一个`(Error) -> Bool`类型的闭包来处理判断逻辑：

```swift
extension ObservableType {
    public func retryWhen<Error: Swift.Error>(_ shouldRetry: @escaping (Error) -> Bool) -> Observable<E> {
        return self.retryWhen({ (errorObserver: Observable<Error>) -> Observable<()> in
            return errorObserver.flatMap({ (error) -> Observable<()> in
                if shouldRetry(error) {
                    return .just(())
                }
                return .error(error)
            })
        })
    }
    
    public func retryWhen(_ shouldRetry: @escaping (Swift.Error) -> Bool) -> Observable<E> {
        return self.retryWhen({ (errorObserver: Observable<Swift.Error>) -> Observable<()> in
            return errorObserver.flatMap({ (error) -> Observable<()> in
                if shouldRetry(error) {
                    return .just(())
                }
                return .error(error)
            })
        })
    }
}
```

将上面这段代码复制到你的项目中，之前的重试逻辑就变成了：

```swift
...
.retryWhen({ (error) -> Bool in
    if case CustomError.tooSmall = error {
        return true
    }
    return false
})
...
```

这样看起来清楚多了，减轻了思维负担。

## 实际应用

`Moya`是Swift常用的一个网络库，它提供了Rx的接口，下面的例子以Moya作为网络库来演示，Moya的一个核心协议是`TargetType`，不了解Moya的朋友可以看看它的[文档](https://github.com/Moya/Moya)，基本使用就不再详细介绍了。下面来看两个常见的实际应用场景

### 场景一：带交互的出错重试

在很多时候，用户的操作失败时不能直接重试，而是要给一个，让用户来决定下一步的操作。例如有一个文件下载的请求，当下载失败的时候需要弹框来询问是否重试。也就是说在出错到重试之间存在一个**“中断”**，只有当用户做出选择之后操作链才会继续向下执行。

解决方法是使用`retryWhen`，将参数中的的`Observable<Error>`与我们自己业务逻辑的`Observable`关联起来。

首先，我们假定有这样一个确认框的控件，它的签名如下：

```swift
class ConfirmView: UIView {
    /// 在视图中显示一个确认框，callback为点击的回调，点击确认回调true，点击取消回调false
	static func show(_ title: String, _ callback: (Bool) -> Void) {
		...
    }
}
```

实际的项目中通常都会有很多封装好的控件类型，借助于`RxSwift`中所提供的扩展机制，只需要添加一个小小的扩展就可以与Rx的世界无缝对接起来：

```swift
extension Reactive where Base: ConfirmView {
    // 1. 在扩展中定义一个show方法，不同的是没有callback参数，而是返回一个Observable<Bool>
	static func show(_ title: String) -> Observable<Bool> {
        // 2. 创建一个Observable<Bool>
		return Observable<Bool>.create({ (observer) -> Disposable in
            // 3. 调用原始的show方法，并在回调中通过observer发送结果
            ConfirmView.show(title, { (confirm) in
                observer.onNext(confirm)
				observer.onCompleted()
            })
            return Disposables.create { 
            	// do some cleanup
            }
        })
    }
}
```

之后就可以通过`ConfirmView.rx.show(xxx)`的方式来调用这个方法了，这个方法会弹出一个选择框等待用户的选择，选择的结果通过`Observable`的事件来进行通知。之后我们使用`flatMap`将这个`Observable`与`retryWhen`中的`Obverable<Error>`关联起来：

```swift
...
.retryWhen({ (errorO) -> Observable<()> in
    return errorO.flatMap({ (error) -> Observable<()> in
        if case CustomError.tooSmall = error {
            return ConfirmView.rx
                .show("是否重试?")
                .map {
                    if $0 { // 1. 如果选择了重试，则返回.next()表示重试
                        return ()
                    } else {
                        throw error // 2. 否则继续返回error将错误继续向下传递
                    }
                }
        }
        return .error(error)
    })
})
.subscribe {
	// 3. 如果上面选择了重试，这里不会接收到错误事件
}
...
```

类似的，将不同的操作封装成`Observable`这样简单的逻辑流，然后通过RxSwift提供的操作加以组合以实现更加复杂的逻辑，这也是Rx所提倡的函数式思想。

### 场景二：401认证

401错误是一种很常见应用场景，比如说在我们的应用中认证流程是这样的：当服务器需要重新认证用户登录信息时会返回一个`401`状态码，这时客户端将认证信息添加到请求头中并重发当前的请求，这一过程对上层的业务方应该是无感知的。

这跟之前的例子有一些不同的地方：当出错时我们不能直接`retry`整个请求，而是要**修改原始请求**添加自定义的Header，最简单粗暴的方法是在检测到401错误时发送一个通知，外面收到通知之后将Header添加到请求头里：

```swift
moyaProvider.request(target)
    .map({ (response) -> Response in
        if response.statusCode == 401 { // 将401转换成自定义的错误类型
            // 先发送通知，之后再retry
            NotificationCenter.default.post(name: .AddAuthHeader, object: nil)
            throw NetworkError.needAuth
        } else {
            return response
        }
    })
	.retry()
```

这种做法其实并不好，因为Rx中强调的是事件流，原本应该是一个连贯的逻辑却被通知给打断了，当我们阅读到这里的时候还得停下来全局搜索通知的名字以查找响应的位置，这样不利于阅读，同时也违背了Rx的哲学。

我这里所采用的做法是捕获到错误时不进行`retry`，而是返回一个新的网络请求。为了让这个新的网络请求与之前的逻辑无缝连接起来，首先需要定义一个代理TargetType：

```swift
let ProxyProvider = NetworkProvider<ProxyTarget>()

enum ProxyTarget {
    // 添加Header
    case addHeader(target: TargetType, headers: [String: String]) 
    // ...
}

extension ProxyTarget: TargetType {
	var headers: [String: String]? {
        switch self {
        // 1. 将新增的Header添加到被代理的Target上
        case let .addHeader(target: target, headers: headers):
            return headers.merging(target.headers ?? [:], uniquingKeysWith: { (first, second) -> String in
                return first
            })
        }
    }
    
    // 2. 不需要吹的地方直接返回被代理Target的属性
    var task: Task {
        switch self {
        case let .addHeader(target: target, headers: _):
            return target.task
        }
    }
    
    // ...
}
```

`ProxyTarget`并没有定义新的网络请求，而是用来代理另外一个`TargetType`，这里我们只定义了一个`addHeader`操作，用来修改请求的Header。

最终的实现如下：

```swift
provider.request(target)
    .map({ (response) -> Response in
        if response.statusCode == 401 { // 1. 将401转换成自定义的错误类型
            throw NetworkError.needAuth
        } else {
            return response
        }
    })
	.catchError({ (error) -> Single<Response> in
        if case NetworkError.needAuth(let response) = error{
            // 2. 捕获未认证的错误，添加认证头后再次重试
            let authHeader = ... // 计算认证头
            let target = ProxyTarget.addHeader(target: token, headers: authHeader)
            return ProxyProvider.rx.request(target, callbackQueue: callbackQueue)
        }
        return Single.error(error)
    })
	.subscribe {
		// 3. 认证的过程对于上层的业务方是无感知的
    }
	...
```

使用`map`将401转换成自定义的错误类型，之后在`catchError`中捕获这个错误，使用`ProxyTarget`加上认证头之后返回一个新的`Observable`，这样一来所有相关的逻辑都被集中在这一系列的链式调用中了。

当然在实际项目中不仅仅是401这类错误，可能还会有许多其他业务相关的错误类型，将它们全都放在`map`中处理显然不是一个好主意，最好的办法是将这部分逻辑抽离出来放在Moya的`Plugin`中，这里就不再演示了。

## 最后

Rx中对于事件流的抽象十分强大，可以用来描述各种复杂的场景，这里仅仅从错误处理的方面列举了一些简单的例子，可以看到Rx的思想跟我们平常所写的代码有很大不同，思维上的转变才是理解Rx的关键。