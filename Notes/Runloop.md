Runloop是一种事件循环的模型，是一种很常见的模式。正常来说一个线程在执行完它的代码之后就会推出，简单来看Runloop就是<u>一个无限循环，让**线程**能够随时处理事件并且结束后不会退出</u>。它和线程是一一对应的，主线程的Runloop由系统负责创建，<u>子线程默认不会创建Runloop</u>。

## API

系统在`CoreFoundation`框架中提供了`CFRunLoopRef`对象，这是纯C语言的API，它是线程安全的。在`Foundation`框架中提供了`NSRunLoop`对象，这是对`CFRunLoopRef`的面向对象封装，但是它不是线程安全的。

系统不允许用户直接创建Runloop，而是提供了两个自动获取的函数：`CFRunLoopGetMain()`和`CFRunLoopGetCurrent()`。上层公开的两个函数非常简单：

```objective-c
CFRunLoopRef CFRunLoopGetMain(void) {
    return _CFRunLoopGet0(pthread_main_thread_np()); // 获取主线程的Runloop
}

CFRunLoopRef CFRunLoopGetCurrent(void) {
    return _CFRunLoopGet0(pthread_self()); // 获取当前线程的Runloop
}
```

可以看到对于我们的应用代码来说，除了主线程的Runloop，<u>一个线程只能获取自身的Runloop；</u>

在源码中这两个函数都是通过一个名为`_CFRunLoopGet0`的函数来创建Runloop的，简化后如下：

```c
static CFMutableDictionaryRef __CFRunLoops = NULL;

// 接受一个 pthread_t 类型的线程ID作为参数，返回（不存在则创建）该线程的Runloop
CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    // 1. 创建字典，Key为线程id，Value为CFRunloopRef对象
    if (!__CFRunLoops) {
        __CFRunLoops = ...;
    }
    
    // 2. 尝试从字典中找到该线程的Runloop对象
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    // 3. 不存在则创建一个新的Runloop
    if (!loop) {
        CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        // ...缓存到字典中
    }
    
    // ...
   	// 4. 注册一个回调，当线程销毁的时候，在 _CFFinalizeRunLoop 函数中销毁它的 Runloop
    if (_CFGetTSD(__CFTSDKeyRunLoopCntr) == 0) {
        _CFSetTSD(__CFTSDKeyRunLoopCntr, ..., _CFFinalizeRunLoop);
    }

    return loop;
}
```

整体逻辑很简单，<u>从这里可以看到Runloop和线程是一一对应的，只有当你第一次尝试获取 Runloop 的时候才会创建</u>。创建Runloop对象的代码实现在`__CFRunLoopCreate`函数中

## RunLoop Mode

简单来说 **Runloop Mode** 中封装了这一次循环中需要处理的各种事件，Runloop 每一次循环只能运行在一个 Mode 上，从该 Mode 上接受需要处理的事件。

Mode的类型是`CFRunLoopModeRef`:

```c
typedef struct __CFRunLoopMode *CFRunLoopModeRef;

struct __CFRunLoopMode {
	...
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
};
```

Source/Timer/Observer 这些统称为`Runloop Mode Item`，它们分别表示：

- **Source**

  `CFRunLoopSourceRef`类型，用来接受应用中的各种事件，分为`source0`和`source1`两种不同的Source：

  - **Source0：**

    事件源，Source0 在 CoreFoundation 中有公开的接口供开发者使用，Observer 也可以监听到对应的事件。

  - **Source1：**

    系统使用的基于`port`的 Source，port 是一种进程间通信的方式，应用程序通过 Source1 来接收系统事件（如触摸等），一些系统事件从 Source1 接收到之后会通过 Source0 回调给开发者。

    这里接收事件主要通过`mach_msg`函数来实现。

- **Timer**

  `CFRunLoopTimerRef`类型，用来接收定时触发的事件，实际上`Foundation`层的`NSTimer`就是对这个的封装。这可以解释为什么`NSTimer`的触发时机不是绝对准确：

  因为`Timer Source`是保存在`Runloop Mode`中的，<u>如果到了 Timer 所设定的时间，但是当前运行的并不是保存这个 Timer 的 Mode，那 Timer 是不会触发的</u>，直到下一次运行这个 Mode 的时候才会触发这个 Timer 的事件，而此时已经超过 Timer 最初所设定的时间间隔了。

  <u>Timer只会在在每一次 Runloop 中进行判断</u>，比如说有一个1秒触发一次的 Timer，中间有一次 Runloop 耗时超过了2秒，那这个 Timer 只会被执行一次，中间那一次触发机会就永远的错过了（不会重发）。

  同理，不能为一个没有Runloop的线程添加Timer。

- **Observer**

  `CFRunLoopObserverRef`类型，这是 Runloop 的观察者，包含了一个回调函数，在每一次 Runloop 循环的特定时间点都会发送相应的通知，类型有：

  ```c
  typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
      kCFRunLoopEntry = (1UL << 0), 		  // 即将进入Runloop
      kCFRunLoopBeforeTimers = (1UL << 1),  // 即将处理Timer
      kCFRunLoopBeforeSources = (1UL << 2), // 即将处理Source(Source0)
      kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠(等待事件唤醒)
      kCFRunLoopAfterWaiting = (1UL << 6),  // 休眠中唤醒(进行下一次循环)
      kCFRunLoopExit = (1UL << 7),		  // 这一次的Runloop循环结束(然后结束或进入下一次循环)
      kCFRunLoopAllActivities = 0x0FFFFFFFU
  };
  ```

  `CoreFoundation`框架提供了相应的C语言接口用来添加 Runloop 观察器。

### Mode的分类

既有对开发者可见的`Common Mode`，也有系统私有的`Private Mode`。

像默认情况下的`NSDefaultRunLoopMode`和页面滚动时的`UITrackingRunLoopMode`都属于 Common Mode，我们常常用的`NSRunLoopCommonModes`其实并不是指某一个特定的 Mode，它代表的是所有的 Common Mode。

而 Private Mode 的种类就非常多了，系统没有为开发者开放这类 Mode 的权限，我们在监听 Runloop 动作或是添加 NSTimer 的时候常常会使用`NSRunLoopCommonModes`，而当系统运行在 Private Mode 下的时候我们是无法得通知的。

## RunLoop

`CFRunLoopRef`是 CoreFoundation 提供的 Runloop 类型，其中跟 Mode 相关的成员变量如下：

```c
typedef struct __CFRunLoop * CFRunLoopRef;

struct __CFRunLoop {
    ...
    CFMutableSetRef _modes; // 保存所有的mode，包括private和common的mode
    CFRunLoopModeRef _currentMode; // 当前的Mode
    CFMutableSetRef _commonModes; // 保存公共的Mode
    CFMutableSetRef _commonModeItems; // Source/Timer/Observer等，会自动添加到所有的common mode中
};
```

- **_modes**

  这个字段保存了 Runloop 中的所有 Mode，包括 Common Mode（下面的`_commonModes`） 和 Private Mode。

- **_currentMode**

  Runloop 当前所运行的 Mode，每次 Runloop 循环只能处理跟这个 Mode 相关的事件。

- **_commonModes** 

  保存 Common Mode，系统为开发者公开的 Mode 和开发者自己创建的 Mode 都保存在这里。

- **_commonModeItems**

  我们在为 Runloop 添加 NSTimer、Observer，或是 performSelector 的时候，常常会将 Mode 设定为`NSRunLoopCommonModes`，<u>实际上这个时候我们的 Timer 和 Observer 等都是添加到了这个集合中，在 Runloop 运行的时候集合的内容会被添加到`_commonModes`中的 Mode 上。</u>

### 执行流程

`CFRunLoopRun()`用来启动当前线程的 Runloop，关键逻辑如下：

```objective-c
void CFRunLoopRun(void) {
    do {
        // 1. 通知Entry动作
        __CFRunLoopDoObservers(kCFRunLoopEntry); 
        do {
			// 2. 即将处理Timer事件
            __CFRunLoopDoObservers(kCFRunLoopBeforeTimers); 
            // 3. 即将处理Source0事件
            __CFRunLoopDoObservers(kCFRunLoopBeforeSources);
            // 4. 执行被加入到Runloop中的Block
            __CFRunLoopDoBlocks(); 
            // 5. 处理Source0事件
            __CFRunLoopDoSources0();
            // 6. 如果有消息则直接处理，否则休眠
            if (!ready) {
                // 6.2. 通知BeforeWaiting动作，即将进入休眠
                __CFRunLoopDoObservers(kCFRunLoopBeforeWaiting);
                // 6.3. 通过系统调用 mach_msg 进入休眠直到被事件唤醒
                __CFRunLoopServiceMachPort();
                // 6.3. 通知AfterWaiting，从休眠中唤醒
                __CFRunLoopDoObservers(kCFRunLoopAfterWaiting);
            }
            // 6.1. 分情况处理Source1、Timer事件或Main Queue的Block
            DoSource1();
            DoTimer();
            DoMainQueueBlock();
        } while (ret == 0) // 没有结束、切换mode、超时则继续
        // 7. 通知Exit动作，结束这一次Runloop
        __CFRunLoopDoObservers(kCFRunLoopExit); 
    } while (not finish);
}
```

<u>可以看到`kCFRunLoopBeforeSources`这个动作所指的其实是 Source0 事件</u>，而对于 Source1 事件的执行时机上层应用是无从得知的。

从源码中可以看到，Runloop 内部实现有两个循环，内层的循环结束时会检查Mode Item是否为空、是否超时、是否需要切换 Mode 等，如果有则发送 Exit 事件，退出当前循环。所以在观察主线程 Runloop 的时候会看到，当滚动 ScrollView 的时候会触发 Exit 和 Entry，这是因为 Mode 发生改变触发了这两个事件。

## Runloop和自动释放池

在应用启动之后，在LLDB中通过`po CFRunLoopGetMain()`可以查看主线程 Runloop 中的状态，可以看到两个跟`AutoreleasePool`相关的`Observer`，回调的函数都是`_wrapRunLoopWithAutoreleasePoolHandler`，这两个 Observer 的作用分别是：

1. 第一个 Observer 监听`kCFRunLoopEntry`事件，也就是创建完 Runloop 准备启动的时候调用，这个回调中会通过`_objc_autoreleasePoolPush()`函数创建一个自动释放池。这一次 Runloop 循环中的创建对象都通过这个自动释放池来管理。
2. 第二个 Observer 监听了`kCFRunLoopBeforeWaiting`和`kCFRunLoopExit`事件。在准备休眠的时候调用`_objc_autoreleasePoolPop()`来释放自动释放池，然后在创建一个新的自动释放池；在 Runloop 退出的时候释放自动释放池。

<u>子线程在创建的时候也会自动添加一个AutoreleasePool</u>（可以通过`_objc_autoreleasePoolPrint`方法查看）。

> AutoreleasePool 会 Retain 所有添加到其中的对象，_objc_autoreleasePoolPop() 会调用其中所有对象的 Release，如果此时对象的引用计数为0则被释放。

## 跟Runloop有关的API

### NSTimer

`NSTimer`实际上就是CoreFoundation中的`CFRunLoopTimerRef`（参考上面的介绍），`NSTimer`中以`scheduled`开头的方法只会将 Timer 添加到`DefaultMode`上，这样当列表滚动的时候就会失效，如果希望滚动的时候也能触发，需要手动将其添加到 Runloop 的`CommonMode`上：

```objective-c
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

当然，添加 Timer 的线程必须要有 Runloop。

### performSelector

`NSObject`中`performSelector`系列的API也是跟 Runloop 有关的，它可以将方法派发到指定的线程和 Mode 中去执行。`performSelector`系列的API大致有三种类型：

1. `- [performSelector: withObject:]`

   这个跟直接调用方法是一样的，不涉及到 Runloop 的操作

2. `- [performSelector: onThread: withObject: waitUntilDone:NO];`

   在指定线程中执行方法，并且不等待方法执行结束。这类API所做的事情是<u>将一个`Source0`（CFRunLoopSource）添加到了指定线程的`Common Mode`上</u>，方法在下一次 Runloop 循环的时候被执行。所以<u>如果目标线程没有 Runloop 的话这个方法永远得不到执行</u>。

   需要注意的是`waitUntilDone`参数必须为`NO`才会被添加到下一次Runloop中。

3. `- [performSelector: withObject: afterDelay:];`

   这类`performSelector`带有一个`afterDelay`参数用来设置时间，这一类API会向**当前线程**的 Runloop 中添加一个 Timer，在 Timer 的回调函数中执行目标方法。

第2和第3类的API都是涉及到 Runloop 的操作，目标线程必须要有 Runloop 才能生效。

> 可以通过对比 performSelector 方法执行前后 CFRunLoopRef 对象的数据来验证

### CFRunLoopPerformBlock

这是 CoreFoundation 中的一个纯C函数，<u>Runloop 循环中除了 Source/Timer/Observer 事件以外，还会处理添加到 Runloop 中的 Block</u>，Block 是在 Runloop 的`__CFRunLoopDoBlocks()`函数中处理的。

`CFRunLoopPerformBlock`可以将一个 Block 添加到指定的 Runloop 中：

```objective-c
CFRunLoopPerformBlock(CFRunLoopGetCurrent(), kCFRunLoopCommonModes, ^{
    NSLog(@"perform block"); 
});
```

Block 会在下一次的 Runloop 循环中执行。

## Runloop的运用

### 在子线程中创建Runloop

Foundation 层提供了`NSRunLoop`作为 CoreFoundation 层的封装，Runloop 中必须存在事件源才能运行，所以可以添加一个`NSPort`作为`Input Source`，防止 Runloop 直接退出：

```objective-c
// 主线程中调用
self.thread = [[NSThread alloc] initWithTarget:self selector:@selector(runThread) object:nil];
[self.thread start];

// 子线程中执行
- (void)runThread {
    NSRunLoop *runloop = [NSRunLoop currentRunLoop];
    // 添加的这个Port不处理任何事件，仅仅用来防止Runloop退出
    [runloop addPort:[NSMachPort port] forMode:NSRunLoopCommonModes];
    [runloop run]; // 或[runloop runUntilDate:[NSDate distantFuture]]
}
```

如果使用 CoreFoundation 的话，`runThread`中的代码可以这样写：

```objective-c
CFRunLoopSourceContext context = {};
CFRunLoopSourceRef source = CFRunLoopSourceCreate(NULL, 0, &context);
CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopCommonModes);
CFRunLoopRun();
```

向 Source0 中添加一个自定义的 Source。

如果要结束这个线程的话，可以在线程中执行`[NSThread exit]`结束当前线程：

```objective-c
// perform到目标线程中执行
[self performSelector:@selector(killThread) onThread:self.thread withObject:nil waitUntilDone:NO];

// 目标线程
- (void)killThread {
    [NSThread exit];
}
```

### 创建Observer

有两个方法可以创建 Observer：`CFRunLoopObserverCreate`和`CFRunLoopObserverCreateWithHandler`，后者可以通过 Block 来创建 Observer：

```objective-c
// 1. 通过Block创建一个Observer
CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(NULL, kCFRunLoopAllActivities, true, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
	// 接收事件activity
});
// 2. 将Observer添加到当前的Runloop上
CFRunLoopAddObserver(CFRunLoopGetCurrent(), observer, kCFRunLoopCommonModes);
```

### 创建自定义事件源

CoreFoundation 中提供了方法`CFRunLoopSourceCreate`，可以创建一个自定义的 Source0 事件源并添加到 Runloop 上：

```objective-c
// Source触发时的回调函数
void perform() {
    NSLog(@"do something");
}

// 创建 Source0
CFRunLoopSourceContext context = {};
context.perform = &perform; // 指定回调函数地址
self.source = CFRunLoopSourceCreate(NULL, 0, &context);
CFRunLoopAddSource(CFRunLoopGetCurrent(), self.source, kCFRunLoopCommonModes);
```

当 Runloop 处理该 Source 上的事件时会调用 Context 中的 perform 函数，这一步是通过源码中的`__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__`方法来完成的，在 perform 中打个断点就能看到这个函数。

当我们想要唤醒该线程的时候需要执行：

```objective-c
// 1. 向自定义的source添加事件
CFRunLoopSourceSignal(self.source);
// 2. 唤醒线程处理事件(调用perform)
CFRunLoopWakeUp(self.runloop);
```

当然，如果要做的仅仅是唤醒线程执行某个方法的话直接用`performSelect:onThread:`就可以了。Facebook 的 [Texture](https://github.com/TextureGroup/Texture) 就利用了类似的机制，将处理任务的时机放到了`kCFRunLoopBeforeWaiting`的时候。