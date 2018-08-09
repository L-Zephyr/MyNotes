[TOC]


## Observable

在RxSwift中，最关键的一个概念是可观察序列（Observable Sequence），它相当于Swift中的序列（Sequence），可观察序列中的每个元素都是一个事件，我们知道Swift的序列中可以包含任意多个元素，类似的，可观察序列会不断产生新的事件直到发生错误或正常结束为止。订阅者（Observer）通过订阅（subscribe）一个可观察队列来接收序列所产生的新事件，只有在有观察者的情况下序列才可以发送事件。

例如，使用`of`操作创建一个可观察序列（以下简称为Observable）：

```swift
let seq = Observable.of(1, 2, 3) 
```

`of`是一种用来创建Observable的简便操作，在上面的代码中创建了一个类型为`Observable<Int>`的Observable，里面包含了三个元素：1，2，3。

来看看Observable中都提供了哪些操作，可观察序列是一个实现了`ObservableType`协议的类型，`ObservableType`协议的定义非常简单：

```swift
protocol ObservableType : ObservableConvertibleType {
    associatedtype E
    func subscribe<O: ObserverType>(_ observer: O) -> Disposable where O.E == E
}
```

其中`E`是一个关联类型，表示序列中元素的类型，除此之外协议只定义了一个方法：`subscribe`，用于向可观察序列添加一个观察者（`ObserverType`类型）：

```swift
// 接收闭包的subscribe函数是通过协议扩展提供的简便方法
seq.subscribe { (event) in
    print(event)
}
```

`subscribe`相当于Swift序列中的遍历操作（makeIterator），如上，向seq序列添加一个观察者，在序列中有新的事件时调用该闭包，上面的代码会输出1，2，3。

## Observer

观察者是实现了`ObserverType`协议的对象，`ObserverType`协议同样十分简单：

```swift
public protocol ObserverType {
    associatedtype E
    func on(_ event: Event<E>)
}
```

`E`为观察者所观察序列中的元素类型，当序列中有新的事件产生时，会调用`on`方法来接收新的事件。其中事件的类型`Event`是一个枚举，其中包含3个类型：

```swift
enum Event<Element> {
    case next(Element)
    case error(Swift.Error)
    case completed
}
```

1. `.next`：表示序列中产生了下一个事件，关联值Element保存了该事件的值。
2. `.error`：序列产生了一个错误，关联值Error保存了错误类型，在这之后序列会直接结束(不再产生新的next事件)。
3. `.completed`：序列正常结束。

## Dispose

除了产生错误和自然结束以外，还可以手动结束观察，在使用`subscribe`订阅一个可观察序列时，会返回一个`Disposable`类型的对象。这里的`Disposable`是一个协议，只定义了一个方法：

```swift
protocol Disposable {
    func dispose()
}
```

`dispose`方法用来结束此次订阅并释放可观察序列中的相关资源，通常来说你并不需要直接调用该方法，而是通过调用其扩展方法`addDisposableTo`将`Disposable`添加到一个`DisposeBag`对象中。DisposeBag对象会自动管理所有添加到其中的Disposable对象，在DisposeBag对象销毁的时候会自动调用其中所有Disposable的dispose方法释放资源。

也可以使用`takeUntil`来自动结束订阅：

```swift
seq.takeUntil(otherSeq)
	.subscribe({ (event) in
    	print(event)
	})
```

在otherSeq序列发出任意类型的事件之后，自动结束本次订阅。

## 创建序列

通过`Observable`类型提供的方法`create`可以创建一个自定义的可观察序列：

```swift
let seq = Observable<Int>.create { (observer) -> Disposable in
    observer.on(.next(1))
    observer.on(.completed)
    return Disposables.create {
        // do some cleanup
    }
}
```

`create`方法使用一个闭包来创建自定义的序列，闭包接收一个`ObserverType`的参数observer，并通过observer来发送相应的事件。如上面的代码，创建了一个`Observable<Int>`类型的可观察序列，订阅该序列的观察者会收到事件1和一个完成事件。最后`create`方法返回一个自己创建的`Disposable`对象，可以在这里进行一些相关的资源回收操作。

除了`create`方法之外，RxSwift中提供了很多中简便的方法用于创建序列，常用的有：

- **just**：创建一个只包含一个值的可观察序列：

  ```swift
  let justSeq = Observable.just(1)
  justSeq.subscribe { (event) in
      print(event)
  }
  ---- example output ----
  next(1)
  completed
  ```


- **of**：`of`和`just`有点类似，不同的是`of`可以将一系列元素创建成事件队列，该`Observable`依次发送相应事件和结束事件：

  ```swift
  let ofSeq = Observable.of(1, 2, 3)
  ofSeq.subscribe { (event) in
      print(event)
  }
  ---- example output ----
  next(1)
  next(2)
  next(3)
  completed
  ```

- **empty**：这种类型的Observable只发送结束(Completed)事件

  ```swift
  let emptySequence = Observable<String>.empty()
  ```

- **error**：该队列只发送一个`error`事件，传递一个自定义的错误类型。

  ```swift
  let errorSeq = Observable<TestError>.error(TestError.Error1)
  ```


## Share

通常在我们在订阅一个可观察序列的时候，每一次的订阅行为都是独立的，也就是说：

```swift
let seq = Observable.of(1, 2)
// 1
seq.subscribe { (event) in
    print("sub 1: \(event)")
}
// 2
seq.subscribe { (event) in
    print("sub 2: \(event)")
}
---- example output ----
sub 1: next(1)
sub 1: next(2)
sub 1: completed 
sub 2: next(1)
sub 2: next(2)
sub 2: completed 
```

我们连续订阅同一序列两次，每次都会接收到相同的事件，第二次订阅时并没有因为第一次订阅的行为导致元素"耗尽"。有些时候我们希望让所有的观察者都共享同一份事件，这个时候可以使用`share`

- **share**：`share`是`ObservableType`协议的一个扩展方法，它返回一个可观察序列，该序列的所有观察者都会共享同一份订阅，上面的代码加上share之后：

  ```swift
  let seq = Observable.of(1, 2).share()
  // 1
  seq.subscribe { (event) in
      print("sub 1: \(event)")
  }
  // 2
  seq.subscribe { (event) in
      print("sub 2: \(event)")
  }
  ---- example output ----
  sub 1: next(1)
  sub 1: next(2)
  sub 1: completed 
  sub 2: completed 
  ```

  可以看到，在第一次订阅时序列已经将所有的事件发送，后面再进行第二次订阅的时候只收到了一个完成事件。

- **shareReplay**：`shareReplay`的用法与`share`类似，它的方法签名如下：

  ```swift
  func shareReplay(_ bufferSize: Int) -> Observable<Element>
  ```

  不同的地方在于，`shareReplay`接收一个整型参数bufferSize，指定缓冲区大小，订阅该序列的观察者会立即收到最近bufferSize条事件。

## 序列的变换和组合

在Swift的序列`Sequence`中，可以使用map、flatMap和reduce等常见的函数式方法对其中的元素进行变换，RxSwift中的可观察序列同样也支持这些方法。

### 变换

- **map**：这是`map`方法的签名：

  ```swift
  func map<Result>(_ transform: @escaping (E) throws -> Result) -> Observable<Result>
  ```

  在一个自定义的闭包中对序列的每一个元素进行变换，返回一个包含转换后结果的可观察序列，与Swift中`Sequence`的map类似。

  ```swift
  let mappedSeq: Observable<String> = seq.map { (element) -> String in
  	return "value: \(element)"
  }
  ```

- **flatMap**：先来看看`flatMap`的签名：

  ```swift
  func flatMap<O: ObservableConvertibleType>(_ selector: @escaping (E) throws -> O)
          -> Observable<O.E> 
  ```

  关于flatMap的作用同样可以类比`Sequence`，`Sequence`中的`flatMap`闭包遍历每一个元素进行处理后返回一个新的序列，最后会将这些序列"展平"，得到一个包含所有序列元素的新序列：

  ```swift
  let array = [1, 2]
  let res = array.flatMap { (n) -> [String] in
      return ["\(n)a", "\(n)b"]
  }
  // res: ["1a", "1b", "2a", "2b"]
  ```

  RxSwift中的`flatMap`用法与之类似，`flatMap`中的闭包会遍历可观察序列中的所有元素，并返回一个新的可观察序列，最后`flatMap`会返回一个包含所有元素的可观察序列：

  ```swift
  let seq = Observable.of(1, 2)
      .flatMap { (n) -> Observable<String> in
          return Observable.of("\(n)a", "\(n)b") // (1)
      }
      .subscribe { (event) in
          print(event)
      }
  // 得到的seq类型为Observable<String>
  ---- example output ----
  next(1a)
  next(1b)
  next(2a)
  next(2b)
  completed
  ```

  在闭包中创建了若干个可观察序列(1)，这些序列中发送的`next`事件都会被传递到seq序列中，其中任何一个序列发生错误（发送了`error`事件）时，seq序列都会直接结束，不再继续接收事件；但是只有所有序列都完成（发送了`completed`事件）后，seq序列才会正常结束。

- **flatMapLatest**：作用与`flatMap`类似，但是对于闭包中生成的可观察序列，它并不会保留所有的序列的订阅，在遍历结束后，只保留最后创建的序列的订阅，之前创建的Observables都会取消订阅（相应序列的dispose方法也会被调用）：

  ```swift
  // 与上一个例子相同的代码，仅将flatMap改成flatMapLatest
  let seq = Observable.of(1, 2)
      .flatMapLatest { (n) -> Observable<String> in
          return Observable.of("\(n)a", "\(n)b") // (1)
      }
      .subscribe { (event) in
          print(event)
      }
  ---- example output ----
  next(1a)
  next(2a)
  next(2b)
  completed
  ```

  因为订阅关系的改变，现在只有当最后创建的那个Observable正常结束时，seq才会收到`completed`事件。

  在这种情况下，`flatMapLatest`会得到与`flatMap`相同的输出：

  ```swift
  let seq = Observable.of(1, 2)
      .flatMapLatest { (n) -> Observable<String> in
          return Observable<String>.create({ (observer) -> Disposable in
              observer.onNext("\(n)a")
              observer.onNext("\(n)b")

              return Disposables.create { }
          })
      }
      .subscribe { (event) in
          print(event)
      }
  ```

  这是因为在上面的这个例子中所创建的Observable是同步创建元素的，无法被打断。

  类似的方法还有`flatMapFirst`，使用方法可以类比`flatMapLatest`。

- **reduce和scan**：`reduce`的作用与Sequence中定义的一样，它接收一个初始值和一个闭包，在Observable中的每个值上调用该闭包，并将每一步的结果作为下一次调用的输入：

  ```swift
  Observable.of(1, 2, 3).reduce(0) { (first, num) -> Float in
          return Float(first + num)
      }
      .subscribe { (event) in
          print(event)
      }
  // 输出：next(6.0), completed
  ```

  在上面的代码中，提供了一个初始值0，在闭包中计算和，并将结果序列的元素类型改成`Float`，序列的观察者最后接收到所有元素的和。

  `scan`的作用类似于`reduce`，它跟`reduce`之间唯一的区别在于，`scan`会发送每一次调用闭包后的结果：

  ```swift
  Observable.of(1, 2, 3).scan(0) { (first, num) -> Float in
          return Float(first + num)
      }
      .subscribe { (event) in
          print(event)
      }
  // 输出：next(1.0), next(3.0), next(6.0), completed
  ```

### 组合

- **startWith**：在序列的开头加入一个指定的元素

  ```swift
  Observable.of(2, 3).startWith(1).subscribe { (event) in
      print(event)
  }
  ---- example output ----
  next(1)
  next(2)
  next(3)
  completed
  ```

  订阅该序列之后，会立即收到`startWith`指定的事件，即使此时序列并没有开始发送事件。

- **merge**：当你有多个类型相同的Observable，可以使用`merge`方法将它们合并起来，同时订阅所有Observable中的事件：

  ```swift
  let seq1 = Observable.just(1)
  let seq2 = Observable.just(2)
  let seq = Observable.of(seq1, seq2).merge()
  seq.subscribe { (event) in
      print(event)
  }
  ---- example output ----
  next(1)
  next(2)
  completed
  ```

  只有当Observable中的元素也是Observable类型的时候才可以使用`merge`方法，当其中一个序列发生错误的时候，seq都会被终止，同样的只有所有序列都完成之后，seq才会收到完成事件。

- **zip**：`zip`方法也可以将多个Observable合并在一起，与`merge`不同的是，`zip`提供了一个闭包用来对多个Observable中的元素进行组合变化，最后获得一个新的序列：

  ```swift
  let seq1 = Observable.just(1)
  let seq2 = Observable.just(2)
  let seq: Observable<String> = Observable.zip(seq1, seq2) { (num1, num2) -> String in
      return "\(num1 + num2)"
  }
  seq.subscribe { (event) in
      print(event)
  }
  ---- example output ----
  next(3)
  completed
  ```

  `zip`方法按照参数个数的不同有多个版本，最多支持合并8个可观察序列，需要注意的一点是，闭包所接收的参数是各个序列中对应位置的元素。也就是说，如果seq1发送了一个事件，而seq2发送了多个事件，闭包也只会被执行一次，seq中只有一个元素。

  组合的Observable中任意一个发生错误，最后的seq都会直接出错终止，当所有的Observable都发出completed事件后，seq才会正常结束。

- **combineLatest**：`combineLatest`同样用于将多个序列组合成一个，使用方法与`zip`一样，但是它的调用机制跟`zip`不同，每当其中一个序列有新元素时，`combineLatest`都会从其他所有序列中取出最后一个元素，传入闭包中生成新的元素添加到结果序列中。

## Subject

`Subject`对象相当于一种中间的代理和桥梁的作用，它既是观察者又是可观察序列，在向一个`Subject`对象添加观察者之后，可以通过该`Subject`向其发送事件。`Subject`对象并不会主动发送completed事件，并且在发送了error或completed事件之后，`Subject`中的序列会直接终结，无法再发送新的消息。`Subject`同样也分为几种类型：

- **PublishSubject**：`PublishSubject`的订阅者只会收到在其订阅（subscribe）之后发送的事件

  ```swift
  let subject = PublishSubject<Int>()
  subject.onNext(1)
  subject.subscribe { (event) in
      print(event)
  }
  subject.onNext(2)

  ---- example output ----
  next(2)
  ```

  可以看到，观察者只收到了事件2，在订阅之前发送的事件1并没有接收到。

- **ReplaySubject**：`ReplaySubject`在初始化时指定一个大小为n的缓冲区，里面会保存最近发送的n条事件，在订阅之后，观察者会立即收到缓冲区中的事件：

  ```swift
  let subject = ReplaySubject<Int>.create(bufferSize: 2)
  subject.onNext(1)
  subject.subscribe { (event) in
      print(event)
  }
  subject.onNext(2)

  ---- example output ----
  next(1)
  next(2)
  ```

- **BehaviorSubject**：`BehaviorSubject`在初始化时需要提供一个默认值，在订阅时观察者会立刻收到序列上一次发送的事件，如果没有发送过事件则会收到默认值：

  ```swift
  let subject = BehaviorSubject(value: 1)
  subject.subscribe { (event) in
      print(event)
  }

  ---- example output ----
  next(1)
  ```

- **Variable**：`Variable`是对`BehaviorSubject`的一个封装，行为上与`BehaviorSubject`类似。`Variable`没有`on`之类的方法来发送事件，取而代之的是一个`value`属性，向`value`赋值可以向观察者发送next事件，并且访问`value`可以获取最后一次发送的数据：

  ```swift
  let variable = Variable(1)
  variable.asObservable().subscribe { (event) in
      print(event)
  }
  variable.value = 2

  ---- example output ----
  next(1)
  next(2)
  completed
  ```

  与其他Subject类型不同的是，`Variable`在释放的时候会发送completed事件，并且`Variable`对象永远不会发送error事件。

## Scheduler

`Scheduler`是RxSwift中进行多线程编程的一种方式，一个Observable在执行的时候会指定一个`Scheduler`，这个`Scheduler`决定了在哪个线程对序列进行操作以及事件回调。默认情况下，在订阅Observable之后，观察者会在与调用`subscribe`方法时相同的线程收到通知，并且也会在该线程进行销毁（dispose）。

与GCD类似，`Scheduler`分为串行（serial）和并行（concurrent）两种类型，RxSwift中定义了几种Schedular：

- **CurrentThreadScheduler**：这是默认的Scheduler，代表了当前的线程，serial类型。
- **MainScheduler**：表示主线程，serial类型
- **SerialDispatchQueueScheduler**：提供了一些快捷的方法来创建串行Scheduler，内部封装了DispatchQueue
- **ConcurrentDispatchQueueScheduler**：提供了快捷的方法来创建并行Scheduler，同样封装了DispatchQueue

### subscribeOn和observeOn

`subscribeOn`和`observeOn`是其中两个最重要的方法，它们可以改变Observable所在的Scheduler：

```swift
// main thread
let scheduler = ConcurrentDispatchQueueScheduler(qos: .default)
let seq = Observable.of(1, 2)
seq.subscribeOn(scheduler)
    .map {
        return $0 * 2 // 子线程
    }
    .subscribe { (event) in
        print(event) // 子线程
    }
```

在上面的代码中创建了一个并发的Scheduler，并在序列seq上调用`subscribeOn`指定了该Scheduler，可以看到，我们在主线程中订阅该序列，但是`map`方法以及事件的回调都是在创建的子线程中执行。

`subscribeOn`和`observeOn`都可以指定序列的Scheduler，它们之间的区别在于：

- `subscribeOn`设定了整个序列开始的时候所在的Scheduler，序列在创建以及之后的操作都会在这个Scheduler上进行，`subscribeOn`在整个链式调用中只能调用一次，之后再次调用`subscribeOn`没有任何效果。
- `observeOn`指定一个Scheduler，在这之后的操作都会被派发到这个Scheduler上执行，`observeOn`可以在链式操作的中间改变Scheduler

```swift
createObservable().
	.doSomething()
	.subscribeOn(scheduler1) // (1)
	.doSomethingElse()
	.observeOn(scheduler2) // (2)
	.doAnother()
	...
```

如上代码，在(1)处执行了`subscribeOn`之后，之前的操作createObservable()和doSomething()都会在scheduler1中执行，随后的doSomethingElse()同样也在scheduler1中执行，随后用`observeOn`指定了另外一个scheduler2，之后的doAnother()会在scheduler2上执行。

## 为原有代码添加Rx扩展

RxSwift中提供了一种扩展机制，可以很方便的为原有的代码添加上Rx扩展。首先来看一个结构体`Reactive`：

```swift
public struct Reactive<Base> {
    /// base是扩展的对象实例
    public let base: Base
	
    public init(_ base: Base) {
        self.base = base
    }
}
```

`Reactive`是一个泛型结构体，只定义了一个属性`base`，并且在初始化结构体的时候传入该属性的值。

此外还定义了一个协议`ReactiveCompatible`：

```swift
public protocol ReactiveCompatible {
    associatedtype CompatibleType

    static var rx: Reactive<CompatibleType>.Type { get set }
    var rx: Reactive<CompatibleType> { get set }
}
```

该协议中分别为类对象和实例对象定义一个名字相同的属性：`rx`，类型为上面定义的`Reactive`，随后通过协议扩展为其提供了`get`的默认的实现：

```swift
extension ReactiveCompatible {
    public static var rx: Reactive<Self>.Type {
        get {
            return Reactive<Self>.self
        }
        set {
            // this enables using Reactive to "mutate" base type
        }
    }

    public var rx: Reactive<Self> {
        get {
            return Reactive(self)
        }
        set {
            // this enables using Reactive to "mutate" base object
        }
    }
}
```

关联类型`CompatibleType`被自动推导为实现该协议的类本身，使用`self`初始化一个`Reactive`对象。

最后通过协议扩展为所有的NSObject类型实现了`ReactiveCompatible`协议：

```swift
extension NSObject: ReactiveCompatible { }
```

这样一来，代码中所有继承自NSObject的类型实例中都会有一个类型为`Reactive`的属性`rx`，当我们要为自己的类型添加Rx扩展时，只需要通过扩展向`Reactive`中添加方法就可以了，例如向UIButton类型添加扩展：

```swift
extension Reactive where Base: UIButton { // 为Reactive<UIButton>添加扩展
    public var tap: ControlEvent<Void> {
        return controlEvent(.touchUpInside) // 通过base可以访问该实例本身
    }
}
```

由于`Reactive`是一个泛型类型，我们可以通过where语句指定泛型的类型，这样一来，我们就可以在UIButton实例的rx中访问tap属性了：

```swift
let button = UIButton(...)
button.rx.tap
```

类似RxCocoa这样的RxSwift扩展库都是通过这种方式进行Rx扩展的。