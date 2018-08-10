前段时间在RxSwift上做了一些实践，Rx确实是一个强大的工具，但同时也是一把双刃剑，如果滥用的话反而会带来副作用，本文就引入Rx模式之后如何更好的管理应用的状态和逻辑做了一些粗浅的总结。

本文篇幅较长，主要围绕着状态管理这一话题进行介绍，前两个部分介绍了前端领域中React和Vue所采用的状态管理模式及其在Swift中的实现，最后介绍了另一种简化的状态管理方案。不会涉及复杂的Rx特性，阅读前对Rx有一些基本的了解即可。

## 为什么状态管理这么重要

一个复杂的页面通常需要维护大量的变量来表示其运行期间的各种状态，在MVVM中页面大部分的状态和逻辑都通过ViewModel来维护，在常见的写法中ViewModel和视图之间通常用`Delegate`来通讯，比如说在数据改变的时候通知视图层更新UI等等：

![MVVM](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/谈谈RxSwift和状态管理/20df71c9aff74b853d94c085db9ba925.png)

在这种模式中，ViewModel的状态更新之后需要我们调用Delegate手动通知视图层。而在Rx中这一层关系被淡化了，由于Rx是响应式的，设定好绑定关系后ViewModel只需要改变数据的值，Rx会自动的通知每一个观察者：

![Rx](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/谈谈RxSwift和状态管理/d58774b57ce6e58560cb54e267970954.png)

Rx为我们隐藏了通知视图的过程，首先这样的好处是明显的：ViewModel可以更加专注于数据本身，不用再去管UI层的逻辑；但是滥用这个特性也会带来麻烦，大量的可观察变量和绑定操作会让逻辑变得含糊不清，修改一个变量的时候可能会导致一系列**难以预料**的连锁反应，这样代码反而会变得更加难以维护。

想要更好的过渡到响应式编程，一个统一的状态管理方案是不可或缺的。在这一块前端领域有不少成熟的实践方案，Swift中也有一些开源库对其进行了实现，其中的思想我们可以先来参考一下。

下面的介绍中所涉及的示例代码在：[https://github.com/L-Zephyr/MyDemos/tree/master/RxStateDemo](https://github.com/L-Zephyr/MyDemos/tree/master/RxStateDemo)。

## Redux - ReSwift

`Redux`是Facebook所提出的基于Flux改良的一种状态管理模式，在Swift中有一个名为[ReSwift](https://github.com/ReSwift/ReSwift)的开源项目实现了这个模式。

### 双向绑定和单向绑定

要理解Redux首先要明白Redux是为了解决什么问题而生的，Redux为应用提供统一的状态管理，并实现了单向的数据流。所谓的`单向绑定`和`双向绑定`所描述的都是视图（View）和数据（Model）之间的关系：

比方说有一个展示消息的页面，首先需要从网络加载最新的消息，在MVC中我们可以这样写：

```swift
class NormalMessageViewController: UIViewController {
 	var msgList: [MsgItem] = [] // 数据源
    
    // 网络请求
    func request() {
        // 1. 开始请求前播放loading动画
        self.startLoading()
        
        MessageProvider.request(.news) { (result) in
            switch result {
            case .success(let response):
                if let list = try? response.map([MsgItem].self) {
                    // 2. 请求结束后更新model
                    self.msgList = list
                }
            case .failure(_):
                break
            }
            
            // 3. model更新后同步更新UI
            self.stopLoading()
            self.tableView.reloadData()
        }
    }
    // ...
}
```

还可以将不需要的消息从列表中删除：

```swift
extension NormalMessageViewController: UITableViewDataSource {
	func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCellEditingStyle, forRowAt indexPath: IndexPath) {
        if editingStyle == .delete {
            // 1. 更新model
            self.msgList.remove(at: indexPath.row)
            
            // 2. 刷新UI
            self.tableView.reloadData()
        }
    }
    // ...
}
```

在`request`方法中我们通过网络请求修改了数据`msgList`，一旦`msgList`发生改变必须刷新UI，显然视图的状态跟数据是同步的；在tableView上删除消息时，视图层**直接**对数据进行操作然后刷新UI。视图层即会响应数据改变的事件，又会直接访问和修改数据，这就是一个双向绑定的关系：

![双向绑定](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/RxSwift和状态管理/bca6a4c2976ff1f93611ed892bcf4e49.png)

虽然在这个例子中看起来非常简单，但是当页面比较复杂的时候UI操作和数据操作混杂在一起会让逻辑变得混乱。看到这里`单向绑定`的含义就很明显了，它去掉了`View -> Model`的这一层关系，视图层不能直接对数据进行修改，它只能通过某种机制向数据层传递事件，并在数据改变的时候刷新UI。

### 实现

为了构造单向数据流，Redux引入了一系列概念，这是Redux中所描述的数据流：

![redux](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/RxSwift和状态管理/4b60993a35465dbb2a3abb29b6cf24c8.png)

其中的`State`就是应用的状态，也就是我们的Model部分，先不管这里的`Action`、`Reducer`等概念，从图中可以看到State和View是有着**直接**的绑定关系的，而View的事件则会通过`Action`、`Store`等一系列操作**间接**的改变`State`，下面来详细的介绍一下Redux的数据流的实现以及所涉及到的概念：

1. **View**
   顾名思义，View就是视图，用户在视图上的操作事件不会直接修改模型，而是会被映射成一个个`Action`。

2. **Action**
   `Action`表示一个对数据操作的请求，Action会被发送到`Store`中，这是对模型数据进行修改的唯一办法。

   在ReSwift中有一个名为Action的协议（仅作标记用的空协议），对于Model中数据的每个操作，比如说设置一个值，都需要有一个对应的Action：

   ```swift
   /// 设置数据的Action
   struct ActionSetMessage: Action {
       var news: [MsgItem] = []
   }

   /// 移除某项数据的Action
   struct ActionRemoveMessage: Action {
       var index: Int
   }
   ```

   用`struct`类型来表示一个Action，Action所携带的数据保存在其成员变量中。

3. **Store和State**
   就像上面所提到的，`State`表示了应用中的Model数据，而`Store`则是存放State的地方；在Redux中Store是一个**全局**的容器，所有组件的状态都被保存在里面；Store接受一个Action，然后修改数据并通知视图层更新UI。

   如下所示，每一个页面和组件都有各自的状态以及用来储存状态的Store：

   ```swift
   // State
   struct ReduxMessageState: StateType {
       var newsList: [MsgItem] = []
   }

   // Store，直接使用ReSwift的Store类型来初始化即可，初始化时要指定reducer和状态的初始值
   let newsStore = Store<ReduxMessageState>(reducer: reduxMessageReducer, state: nil)
   ```

   `Store`通过一个`dispatch`方法来接收`Action`，视图调用这个方法来向Store传递Action：

   ```swift
   messageStore.dispatch(ActionRemoveMessage(index: 0))
   ```

4. **Reducer**
   `Reducer`是一个比较特殊的函数，这里其实是借鉴了函数式的一些思想，首先Redux强调了数据的不可变性（Immutable），简单来说就是一个数据模型在创建之后就不可被修改，那当我们要修改Model某个属性时要怎么办呢？答案就是创建一个新的Model，`Reducer`的作用就体现在这里：

   `Reducer`是一个函数，它的签名如下：

   ```swift
   (_ action: Action, _ state: StateType?) -> StateType
   ```

   接受一个表示动作的action和一个表示当前状态的state，然后计算并返回一个新的State，随后这个新的State会被更新到Store中：

   ```swift
   // Store.swift中的实现
   open func _defaultDispatch(action: Action) {
       guard !isDispatching else {
           raiseFatalError("...")
       }
       isDispatching = true
       let newState = reducer(action, state) // 1. 通过reducer计算出新的state
       isDispatching = false

       state = newState // 2. 直接将新的state赋值到当前的state上
   }
   ```

   应用中所有数据模型的更新操作最终都通过`Reducer`来完成，为了保证这一套流程可以正常的完成，`Reducer`必须是一个**纯函数**：它的输出只取决于输入的参数，不依赖任何外部变量，同样也不能包含任何异步的操作。

   在这个例子中的`Reducer`是这样写的：

   ```swift
   func reduxMessageReducer(action: Action, state: ReduxMessageState?) -> ReduxMessageState {
       var state = state ?? ReduxMessageState()
       // 根据不同的Action对数据进行相应的修改
       switch action {
       case let setMessage as ActionSetMessage: // 设置列表数据
           state.newsList = setMessage.news
       case let remove as ActionRemoveMessage: // 移除某一项
           state.newsList.remove(at: remove.index)
       default:
           break
       }
       // 最后直接返回修改后的整个State结构体
       return state
   }
   ```

最后在视图中实现`StoreSubscriber`协议接收State改变的通知并更新UI即可。详细的代码请看Demo中的`Redux`文件夹。

### 分析

Redux将`View -> Model`这一层关系分解成了`View -> Action -> Store -> Model`，每一个模块只负责一件事情，数据始终沿着这条链路单向传递。

- **优点**：

  - 在处理大量状态的时候单向数据流更加容易维护，所有事件都通过唯一的入口`dispatch`手动触发，数据的每一个处理过程都是透明的，这样就可以追踪到每一次的状态变更操作。在前端中Redux的配套工具[redux-devtools](https://github.com/reduxjs/redux-devtools)就提供了一个名为`Time Travel`的功能，能够回溯应用的任意历史状态。

  - 全局Store有利于在多个组件之间共享状态。

- **缺点**：

  - 首先Redux为它的数据流指定了大量的规则，无疑会带来更高的学习成本。

  - 在Redux的核心模型中并没有考虑异步（Reducer是纯函数），所以如网络请求这样的异步任务还需要通过`ActionCreator`之类的机制间接处理，进一步提升了复杂度。

  - 另一个被广为诟病的缺点是，Redux会引入大量**样板代码**，在上面这个简单的例子中我们需要为页面创建Store、State、Reducer、Action等不同的结构：

    ![样板代码](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/RxSwift和状态管理/182431076bf9ed8afcd91df9a69de7a4.png)

    即便是修改一个状态变量这样简单的操作都需要经过这一套流程，这无疑会大大增加代码量。

综上所述，Redux模式虽然有许多优点，但它带来的成本也无法忽视。如果你的页面和交互极其复杂或是多个页面之间有大量的共享状态的话可以考虑Redux，但是对于大部分应用来说，Redux模式并不太适用。

## Vuex - ReactorKit

`Vue`也是近年来十分热门的前端框架之一，`Vuex`则是其专门为`Vue`提出的状态管理模式，在Redux之上进行了一些优化；而`ReactorKit`是一个Swift的开源库，它的一些设计理念与Vuex十分相似，所以这里我将它们放在一起来讲。

### 实现

与`ReSwift`不同的是`ReactorKit`的实现本身便于基于`RxSwift`，所以不必再考虑如何与Rx结合，下面是`ReactorKit`中数据的流程图：

![Reactor](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/RxSwift和状态管理/526522b22e86d9a8fec01d82947e5c74.png)

大体流程与Redux类似，不同的是`Store`变成了`Reactor`，这是`ReactorKit`引入的一个新概念，它不要求在全局范围统一管理状态，而是每个组件管理各自的状态，所以每个视图组件都有各自所对应的`Reactor`。

具体的代码请看Demo中的`ReactorKit`文件夹，各个部分的含义如下：

1. **Reactor：**

   现在用`ReactorKit`来重写上面的那个例子，首先需要为这个页面创建一个实现了`Reactor`协议的类型`MessageReactor`：

   ```swift
   class MessageReactor: Reactor {
       // 与Redux中的Action作用相同，可以是异步
       enum Action {
           case request
           case removeItem(Int)
       }
       // 表示修改状态的动作(同步)
       enum Mutation {
           case setMessageList([MsgItem])
           case removeItem(Int)
       }
       // 状态
       struct State {
           var newsList: [MsgItem] = []
       }
       ...
   }
   ```
   一个Reactor需要定义`State`、`Action`、`Mutation`这三个部分，后面会一一介绍。

   首先比起Redux这里多了一个`Mutation`的概念，在Redux中由于Action直接与Reducer中的操作对应，所以Action只能用来表示同步的操作。`ReactorKit`将这个概念更加细化，拆分成了两个部分：`Action`和`Mutation`：

   - `Action`：视图层触发的动作，可以表示同步和异步（比如网络请求），它最终会被**转换成Mutation**再被传递到Reducer中；
   - `Mutation`：只能表示同步操作，相当于Redux模式中的Action，最终被传入Reducer中参与新状态的计算；

2. **mutate()：**

   `mutate()`是Reactor中的一个方法，用来将用户触发的`Action`转换成`Mutation`，`mutate()`的存在使得Action可以表示异步操作，因为无论是异步还是同步的Action最后都会被转换成同步的Mutation：

   ```swift
   func mutate(action: MessageReactor.Action) -> Observable<MessageReactor.Mutation> {
       switch action {
       case .request:
           // 1. 异步：网络请求结束后将得到的数据转换成Mutation
           return service.request().map { Mutation.setMessageList($0) }
       case .removeItem(let index):
           // 2. 同步：直接用just包装一个Mutation
           return .just(Mutation.removeItem(index))
       }
   }
   ```

   值得一提的是，这里的`mutate()`方法返回的是一个`Observable<Mutation>`类型的实例，得益于Rx强大的描述能力，我们可以用**一致**的方式来处理同步和异步代码。

3. **reduce()：**

   `reduce()`方法这里就没太多可说的了，它扮演的角色与Redux中的Reducer一样，唯一不同的是这里接受的是一个`Mutation`类型，但本质是一样的：

   ```swift
   func reduce(state: MessageReactor.State, mutation: MessageReactor.Mutation) -> MessageReactor.State {
       var state = state
   
       switch mutation {
       case .setMessageList(let news):
           state.newsList = news
       case .removeItem(let index):
           state.newsList.remove(at: index)
       }
   
       return state
   }
   ```

4. **Service**

   图中还有一个与`mutate()`产生交互的`Service`对象，Service指的是实现具体业务逻辑的地方，`Reactor`会通过各个`Service`对象来执行具体的业务逻辑，比如说网络请求：

   ```swift
   protocol MessageServiceType {
       /// 网络请求
       func request() -> Observable<[MsgItem]>
   }
   
   final class MessageService: MessageServiceType {
       func request() -> Observable<[MsgItem]> {
           return MessageProvider
           	.rx
           	.request(.news)
           	.mapModel([MsgItem].self)
           	.asObservable()
       }
   }
   ```

   看到这里`Reactor`的本质基本上已经明了：`Reactor`实际上是一个中间层，它负责管理视图的状态，并作为视图和具体业务逻辑之间通讯的桥梁。


此外`ReactorKit`希望我们的所有代码都通过**函数响应式**（FRP）的风格来编写，这从它的API设计上可以看出：`Reactor`类型中没有提供如`dispatch`这样的方法，而是只提供了一个`Subject`类型的变量`action`：

```swift
var action: ActionSubject<Action> { get }
```

在Rx中`Subject`既是观察者又是可观察对象，常常扮演一个中间桥梁的角色。视图上所有的`Action`都通过Rx绑定到`action`变量上，而不是通过手动触发的方式：比方说我们想在`viewDidLoad`的时候发起一个网络请求，常规的写法是这样的：

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    service.request() // 手动触发一个网络请求动作
}
```

而`ReactorKit`所推崇的函数式风格是这样的：

```swift
// bind是统一进行事件绑定的地方
func bind(reactor: MessageReactor) {
    self.rx.viewDidLoad // 1. 将viewDidLoad作为一个可观察的事件
        .map { Reactor.Action.request } // 2. 将viewDidLoad事件转成Action
        .bind(to: reactor.action) // 3. 绑定到action变量上
        .disposed(by: self.disposeBag)
    // ...
}
```

`bind`方法是视图层进行事件绑定的地方，我们将VC的`viewDidLoad`作为一个事件源，将其转换成网络请求的Action之后绑定到`reactor.action`上，这样当VC的viewDidLoad被调用时该事件源就会发出一个事件并触发`Reactor`中网络请求的操作。

这样的写法是更加FRP，一切都是事件流，但是实际用起来并不是那么完美。首先我们需要为用到的所有UI组件提供Rx扩展（上面的例子使用了RxViewController这个库）；其次这对reactor实例初始化的时机有更加严格的要求，因为`bind`方法是在reactor实例初始化的时候自动调用的，所以不能在`viewDidLoad`中初始化，否则会错过`viewDidLoad`事件。

### 分析

- **优点**：
  - 相比ReSwift简化了一些流程，并且以组件为单位来管理各自的状态，相比起来更容易在现有工程中引入；
  - 与`RxSwfit`很好的结合在了一起，能提供较为完善的函数响应式（FRP）开发体验；
- **缺点**：
  - 因为核心思想还是Redux模式，所以模板代码过多的问题还是无法避免；

## 另一种简化方案

Redux模式对于大部分应用来说还是过于沉重了，而且Swift的语言特性也不像JavaScript那样灵活，很多样板代码无法避免。所以这里总结了另一套简化的方案，希望能在享受单向数据流优势的同时减轻使用者的负担。

详细的代码请看Demo中的`Custom`文件夹：

![Demo](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/谈谈RxSwift和状态管理/0c91d73ec06c1a0836e24270945b2e48.png)

实现非常简单，核心是一个`Store`类型：

```swift
public protocol StateType { }

public class Store<ConcreteState>: StoreType where ConcreteState: StateType {
    public typealias State = ConcreteState

    /// 状态变量，一个只读类型的变量
    public private(set) var state: State
    
    /// 状态变量对应的可观察对象，当状态发生改变时`rxState`会发送相应的事件
    public var rxState: Observable<State> {
        return _state.asObservable()
    }
    
    /// 强制更新状态，所有的观察者都会收到next事件
    public func forceUpdateState() {
        _state.onNext(state)
    }
    
    /// 在一个闭包中更新状态变量，当闭包返回后一次性应用所有的更新，用于更新状态变量
    public func performStateUpdate(_ updater: (inout State) -> Void) {
        updater(&self.state)
        forceUpdateState()
    }
    ...
}
```

其中`StateType`是一个空协议，仅作为类型约束用；`Store`作为一个基类，负责保存组件的状态，以及管理状态更新的数据源，核心代码非常简单，下面来看一下实际应用。

### ViewModel

在实际开发中我让`ViewModel`来处理状态管理和变更的逻辑，再来实现一次上面的那个例子，将一个业务方的`ViewModel`分成三个部分：

```swift
// <1>
struct MessageState: StateType {
    ...
}

// <2>
extension Reactive where Base: MessageViewModel {
    ...
}

// <3>
class MessageViewModel: Store<MessageState> {
    required public init(state: MessageState) {
        super.init(state: state)
    }
    ...
}
```

各个部分的含义如下：

- **定义页面的状态变量**

  描述一个页面所需的所有状态变量都需要定义在一个单独的实现了`StateType`协议的`struct`中：

  ```swift
  struct MessageState: StateType {
      var msgList: [MsgItem] = [] // 原始数据
  }
  ```

  从前面的代码中可以看到`Store`中有一个只读的`state`属性：

  ```swift
  public private(set) var state: State
  ```

  业务方的ViewModel直接通过`self.state`来访问当前的状态变量。而修改状态变量则通过一个`performStateUpdate`方法来完成，方法签名如下：

  ```swift
  public func performStateUpdate(_ updater: (inout State) -> Void)
  ```

  ViewModel在修改状态变量的时候通过`updater`闭包中的参数直接进行修改：

  ```swift
  performStateUpdate { $0.msgList = [...] } // 修改状态变量
  ```

  执行完毕后页面的状态会被更新，所绑定的UI组件也会接受到状态更新的事件。这样一来能避免为每一个状态变量创建一个Action，简化了流程，同时所有更新状态的操作都由经过同一个入口，有利于之后的分析。

  >统一管理状态变量有以下几个优点：
  >- *逻辑清晰：*在浏览页面的代码时只要查看这个类型就能知道哪些变量是需要特别关注的；
  >- *页面持久化：*只需序列化这个结构体就能够保存这个页面的全部信息，在恢复时只需要将反序列化出来的State赋值给`ViewModel`的`state`变量即可：`self.state = localState`；
  - *便于测试：*单元测试时可以通过检查State类型的变量来进行测试；  

- **定义对外暴露的可观察变量（Getter）**

  ViewModel需要暴露一些能让视图进行绑定的可观察对象（Observable），`Store`中提供了一个名为`rxState`的`Observable<State>`类型对象作为状态更新的统一事件源，但是为了更加便于视图层使用，我们需要将其进一步细化。

  这部分逻辑定义在ViewModel的`Rx扩展`中，对外提供可观察的属性，这里定义了视图层需要绑定的所有状态。这部分的作用相当于`Getter`，是视图层从ViewModel中获取数据源的接口：

  ```swift
  extension Reactive where Base: MessageViewModel {
      var sections: Observable<[MessageTableSectionModel]> {
          return base
          	.rxState // 从统一的事件源rxState中分流
              .map({ (state) -> [MessageTableSectionModel] in
                  // 将VM中的后端原始模型类型转换成UI层可以直接使用的视图模型
                  return [
                      MessageTableSectionModel(items: state.msgList.map { MessageTableCellModel.news($0) })
                  ]
              })
      }   
  }
  ```

  这样一来视图层不需要关心`State`中的数据类型，直接通过`rx`属性来获取自己需要观察的属性即可：

  ```swift
  // 视图层直接观察sections，不需要关心内部的转换逻辑
  vm.rx.sections.subscribe(...)
  ```

  > 为什么要将视图层使用的接口定义在扩展中，而不是直接观察基类中的`rxState`：
  > - 定义在Rx扩展中的变量可以直接通过ViewModel的rx属性访问到，便于视图层使用；
  > - State中的原始数据可能需要一定**转换**才能让视图层使用（比如上面将原始的`MsgItem`类型转换成TableView可以直接使用的SectionModel模型），这部分的逻辑适合放在扩展的计算属性中，让视图层更加纯粹；  

- **对外提供的方法（Action）**

  ViewModel还需要接收视图层的事件以触发具体的业务逻辑，如果这一步通过Rx绑定的方式来完成的话，会对业务层代码的编写方式带来很多限制（参考上面的ReactorKit）。所以这部分不做过多的封装，还是通过方法的形式来对外暴露接口，这部分就相当于Action，不过这样的代价是Action无法再通过统一的接口来派发:

  ```swift
  class MessageViewModel: Store<MessageState> { 
      // 请求
      func request() {
          state.loadingState = .loading
          MessageProvider.rx
              .request(.news)
              .map([MsgItem].self)
              .subscribe(onSuccess: { (items) in
                  // 请求完成后改变state中响应的变量，UI层会自动响应
                  self.performStateUpdate {
                      $0.msgList = items
                      $0.loadingState = .normal
                  }
              }, onError: { error in
                  self.performStateUpdate { $0.loadingState = .normal }
              })
              .disposed(by: self.disposeBag)
      }
  }
  ```

  我们之前已经将状态和UI完全分离开来了，所以在ViewModel的逻辑中只需要关心`state`中的状态即可，不需要关心与视图层的交互，所以以这种方式编写的代码同样也是十分清晰的。

### View

视图层需要实现一个名为`View`的协议，这里主要参考了`ReactorKit`中的设计：

```swift
/// 视图层协议
public protocol View: class {
    /// 用于声明该视图对应的ViewModel的类型
    associatedtype ViewModel: StoreType
    
    /// ViewModel的实例，有默认实现，视图层需要在合适的时机初始化
    var viewModel: ViewModel? { set get }
    
    /// 视图层实现这个方法，并在其中进行绑定
    func doBinding(_ vm: ViewModel)
}
```

对于视图层来说，它需要做两件事：

- 实现一个`doBinding`方法，所有的Rx事件绑定都放在这个方法中完成：

  ```swift
  func doBinding(_ vm: MessageViewModel) {
      vm.rx.sections
          .drive(self.tableView.rx.items(dataSource: dataSource))
          .disposed(by: self.disposeBag)   
  }
  ```

- 在合适的时机初始化`viewModel`属性：

  ```swift
  override func viewDidLoad() {
      super.viewDidLoad()
      // 初始化ViewModel
      self.viewModel = MessageViewModel(state: MessageState())
  }
  ```

  当`viewModel`初始化完成后会自动调用`doBinding`方法进行绑定，并且在实例的生命周期中只会被执行一次。

在视图层中对于各种状态的绑定是很重要的一个环节，`View`协议存在的意义在于将视图层的事件绑定**规范化**，防止绑定操作的代码散落在各处降低可读性。

### 数据流

按照以上流程实现的页面数据流如下：

![数据流](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/谈谈RxSwift和状态管理/cc2b85b1a7ec787dcd49ab6877a6e135.png)

1. 视图（View）中的事件触发时，直接调用相应的方法触发`ViewModel`中的逻辑；
2. `ViewModel`中执行具体的业务逻辑，并通过`performStateUpdate`修改保存在State中的状态变量；
3. 状态变量发生改变之后，通过Rx的绑定自动通知视图层更新UI；

这样能保证一个页面的数据始终按照预期的方式来变化，而且单向数据流的特点使得我们可以像Redux这样追踪所有状态的变更，比如说我们可以简单的利用Swift的反射（`Mirror `）来将所有状态的变更打印到控制台中：

```swift
public func performStateUpdate(_ updater: (inout State) -> Void) {
    updater(&self.state)

    #if DEBUG
    StateChangeRecorder.shared.record(state, on: self) // 记录状态的变更
    #endif

    forceUpdateState()
}
```

实现的代码在`StateChangeRecorder.swift`文件中，非常简单只有不到100行。每当有状态发生改变的时候就会在控制台中打印一条Log：

![States Change Log](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/谈谈RxSwift和状态管理/da702144b42557e0a86d9b1c0eae56a1.png)

如果你为所有`StateType`类型实现序列化和反序列化的操作，甚至可以实现类似[redux-devtools](https://github.com/reduxjs/redux-devtools)这样的`Time Travel`功能，这里就不再继续引申了。

## 总结

引入Rx模式需要多方面的考虑，本文仅针对状态管理这一点作了介绍，上面介绍的三种方案各有特点，最终的选择还是要结合项目的实际情况来判断。