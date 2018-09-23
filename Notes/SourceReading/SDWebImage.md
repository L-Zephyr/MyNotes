# SDWebImage

## 代码结构

![工程结构](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/SDWebImage/75b7d4ce97ed656a41c95cf17c82b763.png)

-   **Downloader**

    用来执行网络下载任务，主要有作为单例提供全局控制的`SDWebImageDownloader`类型，以及实现具体下载逻辑的`SDWebImageDownloaderOperation`类型；

-   **Cache**

    管理图片的本地缓存，主要通过一个`SDImageCache`类型的单例来实现；

-   **Decoder**

    图片解码器，封装了各种格式图片的解码逻辑，其中`SDWebImageCoder`和`SDWebImageProgressiveCoder`这两个协议定义了接口，然后针对不同格式的图片实现了多个 Decoder，这些 Decoder 又通过一个单例对象`SDWebImageCodersManager`进行统一的管理；

-   **Utils**

    主要是`SDWebImageManager`类型，作为一个[Facade 模式](https://zh.wikipedia.org/wiki/%E5%A4%96%E8%A7%80%E6%A8%A1%E5%BC%8F)中的高层接口，负责组织`Downloader`、`Cahce`和`Decoder`层中的代码，并向下面`WebCache Categories`的实现提供统一的高层接口；

    另外还有`SDWebImagePrefetcher`类型，也是调用的`SDWebImageManager`中的接口，不同的是它只进行加载和缓存。

-   **Categories**

    在 UIKit 上实现了一些辅助的方法，主要是为了下面的`WebCache Categories`服务；

-   **WebCache Categories**

    提供 UIKit 上的扩展方法，如`UIImageView`、`UIButton`等等，实现了加载图片的逻辑，业务方主要关心的接口都在这里；

-   **FLAnimatedImage**

## 上层接口

`WebCache Categories`文件夹中为`UIButton`、`UIImageView`等 UI 控件提供了分类扩展，实现了一些快捷的方法供业务方使用，其中开发中最常用的接口就是`UIImageView`上的

```objective-c
-[sd_setImageWithURL: placeholderImage: completed:]
```

`UIImageView`中定了很多辅助方法，它们的调用逻辑如下（只是一部分）：

![SD1](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/SDWebImage/2cfd5812c10ec7fc740220eb15098329.png)

最终都会调用到一个`sd_internalSetImageWithURL`方法上面，这个方法定义在`UIView+WebCache`这个扩展中，事实上，SD 支持的所有 UI 控件（如`UIImageView`和`UIButton`）都是通过这个方法来加载图片的：

```objective-c
- (void)sd_internalSetImageWithURL:(NSURL *)url
                  placeholderImage:(UIImage *)placeholder
                           options:(SDWebImageOptions)options
                      operationKey:(NSString *)operationKey
                     setImageBlock:(SDSetImageBlock)setImageBlock
                          progress:(SDWebImageDownloaderProgressBlock)progressBlock
                         completed:(SDExternalCompletionBlock)completedBlock
                           context:(NSDictionary<NSString *, id> *)context;
```

大部分参数的含义从名字中就可以知道了，其中：

-   **setImageBlock**：

    封装了下载完成后设置为控件设置图片的逻辑；

-   **context**：

    字典中保存了一些附加的参数，目前只用到了一个：`SDWebImageExternalCustomManagerKey`：

    ```objective-c
    SDWebImageManager *manager = [context objectForKey:SDWebImageExternalCustomManagerKey];
    if (!manager) {
        manager = [SDWebImageManager sharedManager];
    }
    ```

    外部可以通过这个来提供自定义的`SDWebImageManager`对象，如果没有设置则使用默认的单例。

-   **operationKey**：

    用来标识一个特定的请求，一个控件的实例会保存它自己的所有请求，在开始加载之前会根据`operationKey`退出上一次的相同请求。在默认情况下会使用当前实例的类名作为`operationKey`的值：

    ```objective-c
    NSString *validOperationKey = operationKey ?: NSStringFromClass([self class]);
    ```

    这意味着默认情况下一个实例**只能有一个请求**，但是也会有一些不同的情况，比如说`UIImageView`还提供了设置`highlightedImage`的一系列方法：`-[sd_setHighlightedImageWithURL:]`，它们用了另外一个不同的`operationKey`：

    ```objective-c
    [self sd_internalSetImageWithURL:url
                        placeholderImage:nil
                                 options:options
                            operationKey:@"UIImageViewImageOperationHighlighted"
                           setImageBlock:^(UIImage *image, NSData *imageData) {
                               weakSelf.highlightedImage = image;
                           }
                                progress:progressBlock
                               completed:completedBlock];
    ```

    所以我们可以同时在一个 UIImageView 上并发请求普通的图片和高亮的图片。

`sd_internalSetImageWithURL`方法中通过一个`SDWebImageManager`实例来创建下载任务，下载任务是一个实现了`SDWebImageOperation`协议的对象，控件实例本身会保存它自己的`Operation`，这部分的逻辑封装在一个分类`UIImageView+WebCacheOperation`中：

```objective-c
@interface UIView (WebCacheOperation)
// 保存Operation和Key的映射关系
- (void)sd_setImageLoadOperation:(nullable id<SDWebImageOperation>)operation forKey:(nullable NSString *)key;
// 退出一个指定的下载任务
- (void)sd_cancelImageLoadOperationWithKey:(nullable NSString *)key;
// 删除一个指定的下载任务
- (void)sd_removeImageLoadOperationWithKey:(nullable NSString *)key;

@end
```

在`sd_setImageLoadOperation`时会将`Operation`保存在一个`NSMapTable`的类型的表中，并且它的值是`weak`类型的，也就是说控件本身并**不会持有**`Operation`，实际上`Operation`是由创建它的那个`SDWebImageManager`来持有的。

`sd_internalSetImageWithURL`中主要的操作就是通过`SDWebImageManager`中的：

```objective-c
- (id<SDWebImageOperation>)loadImageWithURL:(NSURL *)url
									options:(SDWebImageOptions)options
									progress:(SDWebImageDownloaderProgressBlock)progressBlock
									completed:(SDInternalCompletionBlock)completedBlock;
```

这个方法来加载图片，在`completedBlock`中拿到`UIImage`，并通过：

```objective-c
- (void)sd_setImage:(UIImage *)image
  		  imageData:(NSData *)imageData
basedOnClassOrViaCustomSetImageBlock:(SDSetImageBlock)setImageBlock
		 transition:(SDWebImageTransition *)transition
		  cacheType:(SDImageCacheType)cacheType
  		   imageURL:(NSURL *)imageURL;
```

这个方法将结果图片设置到 UI 控件上，其中`SDWebImageTransition`类型的参数`transition`是 SD 提供的在设置图片时提供自定义动画的类型，可以通过设置`UIView`上的这个属性来进行自定义：

```objective-c
@property (nonatomic, strong, nullable) SDWebImageTransition *sd_imageTransition;
```

UIKit 上的这一层并不需要关心图片加载和缓存的具体策略，相关的逻辑都封装到了`SDWebImageManager`这个类型中了。

## SDWebImageManager

`SDWebImageManager`提供了一系列加载和缓存相关的接口给上层使用，基本上用户层面只需要关心这里的接口就够了。实际上`SDWebImageManager`整合了底层的网络、解码还有缓存相关组件的工作。

暴露给上层的接口最重要的就是这个：

```objective-c
- (id <SDWebImageOperation>)loadImageWithURL:(NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(SDInternalCompletionBlock)completedBlock;
```

`WebCache Categories`中的 UIKit 扩展都是通过这个方法来加载图片的。

这个方法返回的`id<SDWebImageOperation>`其实是一个`SDWebImageCombinedOperation`类型的对象，这个对象保存了查询缓存和网络请求对应的`Operation`。

加载图片的基本逻辑的伪代码如下

```objective-c
// 1. 创建一个operation
SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
// 2. 先通过SDImageCache来查找缓存
operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(...) {
    // 3. 判断是否需要下载
    if (shouldDownload) {
		operation.downloadToken = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(...) {
            // 4. 缓存下载结果
            if (needCache) {
				[self.imageCache storeImage:... forKey:... toDisk:... completion:...];
            }
        }];
    }
}];
return operation;
```

1. 首先创建一个`SDWebImageCombinedOperation`对象，这是一个复合的 Operation 对象，它保存了下面查找缓存以及网络请求的 Operation，并在`cancel`的时候一起退出；
2. 通过`SDImageCache`中的`-[queryCacheOperationForKey: options: done:]`方法来查询缓存，根据`options`参数的不同，这个过程可能是同步的也可能是异步的，返回的`NSOperation`保存在第一步的`operation`对象中；
3. 查询缓存完成后会结合查找的结果和用户设置的 options 来判断是否需要下载，如果要下载的话会通过`SDWebImageDownloader`的实例来创建一个下载任务，下载任务返回的 Operation 对象同样也保存在第一步的`operation`上；
4. 下载完成后再次调用`SDImageCache`来缓存结果；
5. 最后将第一步创建的`operation`返回，上层业务方可以通过这个来取消请求；

## 缓存-SDImageCache

`SDImageCache`管理了 SD 中的图片缓存逻辑，提供了查询和缓存的接口，在接口设计上又分为同步接口和异步接口，以图片的 URL 作为 Key 值，缓存位置有**内存缓存**和**磁盘缓存**两个部分。

### 异步查询

异步查询主要是`-[queryCacheOperationForKey: options: done:]`这个方法，`SDWebImageManager`正是通过这个方法来查找缓存的，它涉及到的主要方法如图：

![异步查找](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/SDWebImage/226d15544e73af2b11b86de096f522f4.png)

1. 首先会调用`-[imageFromMemoryCacheForKey:]`方法在内存缓存中查找指定的图片，Key 就是图片的 URL，这一步是**同步**的；

2. 如果没有找到，则调用`-[diskImageDataBySearchingAllPathsForKey:]`方法在磁盘中查找缓存文件，这个过程是**异步**的（如果没有设置`SDImageCacheQueryDiskSync`选项的话；

    值得注意的是这里不仅会查找 SD 自己的缓存，用户还可以通过`-[addReadOnlyCachePath:]`方法提供一个自定义的缓存目录，如果 SD 的缓存**未命中**的话最后还会到用户提供的目录下查找，当然文件必须遵循 SD 的命名规则才能被找到，具体可查看`-[cachedFileNameForKey:]`方法；

3. 缓存在磁盘中的图片资源都是原始的格式，所以在取出来之后会通过`-[diskImageForKey: data: options:]`方法对图片进行一个**解码**的操作，减少 UI 渲染的压力，解压后的结果会被**保存到内存缓存**中；

最后这个方法虽然返回了一个`NSOperation`的对象，但它的异步操作其实并不是通过`NSOperation`来实现的，而是通过 GCD 来实现，事实上这里面**所有**的 IO 操作都会被派发到`ioQueue`这个**串行队列**中；返回的`NSOperation`唯一的作用就是用户可以通过设置它的`isCancelled`来取消操作；

### 同步查询

`-[imageFromCacheForKey:]`这个方法会在内存缓存和磁盘缓存中**同步**查找图片，涉及到的方法如下：

![同步查询](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/SDWebImage/05b5765b1f316a14716f69397966040b.png)

还是先在内存中查找，再到磁盘中查找，可以看到调用的方法跟异步查询也是一样的，不过有一个细节在于同步查找也会将查询操作派发到`ioQueue`队列中来执行：

```objective-c
dispatch_sync(self.ioQueue, ^{
    imageData = [self diskImageDataBySearchingAllPathsForKey:key];
});
```

`dispatch_sync`会等待 Block 执行完毕后再返回，所以如果之前有大量的查询操作未完成的话，同步查询会消耗更多的时间，所以尽量还是要用异步接口来查找；

### 保存缓存

保存缓存的接口主要是以下这些：

![储存](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/SDWebImage/83eb5b6df7a64e74178db044a3f90a53.png)

`-[storeImage: imageData: forKey: toDisk: completion:]`是最主要的方法，先将原始的`UIImage`类型图片直接**同步**保存到内存缓存中，然后**异步**派发到`ioQueue`中进行磁盘缓存。

在进行磁盘缓存之前会先通过`SDWebImageCodersManager`来将图片数据 Encode，然后再通过`-[_storeImageDataToDisk: forKey:]`方法将 Encode 后的数据写到磁盘中，这两个过程都是异步的。

另外可以看到还有一个`-[storeImageDataToDisk: forKey:]`方法，这个方法直接将图片的数据保存到磁盘中，**不会**进行 Encode，另外这个方法是同步的（但是磁盘操作也会派发到`ioQueue`中）。

### 内存缓存-SDMemoryCache

在内存缓存上 SD 并没有做太多的封装，只是实现了一个继承自`NSCache`的类型`SDMemoryCache`，设置和读取缓存用的都是`NSCache`中的方法。

不过这里 SD 也做了一个优化，`SDMemoryCache`中保存了一个`NSMapTable`实例，这个表的 Key 是`Strong`的，Value 是`Weak`的，在设置和移除缓存的时候同时也会同步反应到这个`NSMapTable`中：

```objective-c
- (void)setObject:(id)obj forKey:(id)key cost:(NSUInteger)g {
    [super setObject:obj forKey:key cost:g];
    if (!self.config.shouldUseWeakMemoryCache) {
        return;
    }
    if (key && obj) {
        // 将obj同步保存到这个表中
        LOCK(self.weakCacheLock); // 线程同步是通过信号量来实现的
        [self.weakCache setObject:obj forKey:key];
        UNLOCK(self.weakCacheLock);
    }
}
```

但是当收到内存警告的时候，只会从`NSCache`中移除所有的数据，不会清楚`NSMapTable`中的数据：

```objective-c
- (void)didReceiveMemoryWarning:(NSNotification *)notification {
    // Only remove cache, but keep weak cache
    [super removeAllObjects];
}
```

这个设计有两个出发点：

1. MapTable 的值是`Weak`的，所以不会影响内存的释放；

2. 从`NSCache`中移除一个对象，这个对象并不一定会释放，可能还会被`UIImageView`所持有，这种情况下后面再查找时还能通过`NSMapTable`找到这个对象，避免了后续耗时的磁盘查找，这点可以从`setObject`方法上看到：

    ```objective-c
    - (id)objectForKey:(id)key {
        id obj = [super objectForKey:key];
        if (!self.config.shouldUseWeakMemoryCache) {
            return obj;
        }
        // 如果没找到的话会到NSMapTable中查找
        if (key && !obj) {
            LOCK(self.weakCacheLock);
            obj = [self.weakCache objectForKey:key];
            UNLOCK(self.weakCacheLock);
            if (obj) {
                // Sync cache
                NSUInteger cost = 0;
                if ([obj isKindOfClass:[UIImage class]]) {
                    cost = SDCacheCostForImage(obj);
                }
                [super setObject:obj forKey:key cost:cost];
            }
        }
        return obj;
    }
    ```

### 缓存清理

-   **内存缓存**：在发生内存警告的时候进行清理，具体的操作上面已经介绍了；

-   **磁盘缓存**：可以设置缓存文件的最长生命期限以及最大缓存大小，SD 会根据这两个条件来进行缓存清理的判断，这是磁盘清理的相关方法：

    ![清理磁盘](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/SDWebImage/0821ead8a5bcaab59564cde35099b694.png)

    这两个方法分别在两个不同的时机被触发进行磁盘清理：其中`-[deleteOldFiles]`是在 App 即将被终止（`UIApplicationWillTerminateNotification`）的时候调用；`-[backgroundDeleteOldFiles]`会在应用进入后台的时候调用，申请一个 TaskId 并在清理结束后释放。

    `-[deleteOldFilesWithCompletionBlock:]`实现了具体的**异步**清理逻辑，首先将过期的文件删除（默认的最大保存期限是一周），如果缓存文件的总大小超过限制的话，会按照**修改时间**排序，依次删除最老的文件直到小于限制为止。

## 下载-SDWebImageDownloader

前面已经提到了，`SDWebImageManager`通过调用`SDWebImageDownloader`来实现下载的逻辑，这是一个单例对象，管理着 SD 中所有的网络请求，`SDWebImageDownloader`通过下面这两个方法创建下载任务：

![创建下载任务](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/SDWebImage/ff4ec7ba3efe1748a4821159d4ab0027.png)

-   `-[downloadImageWithURL: options: progress: completed:]`：

    这里的异步管理是通过`NSOperationQueue`来实现的，最大并发量为`6`。首先通过下面的`createDownloader...`方法创建一个`NSOperation`，然后将其保存在一个字典`URLOperations`中，以 URL 作为 Key。这里 SD 做了优化，当重复请求同一个 URL 时，重复的请求并不会发出去，而是仅将`callback`保存到已有网络请求`operation`里面，在请求成功之后统一回调：

    ```objective-c
    LOCK(self.operationsLock);
    // 1. 先看看是否正在请求这个URL
    NSOperation *operation = [self.URLOperations objectForKey:url];
    if (!operation || operation.isFinished) {
        // 2. 没有则为该请求创建一个Operation...
    }
    UNLOCK(self.operationsLock);
    // 3. 最后将callback添加到这个operation中
    id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
    ```

    最后这个方法会创建并返回一个`SDWebImageDownloadToken`类型的实例，这个实例中保存了该请求相关的一些数据，用户可以通过这个实例来取消请求：

    ```objective-c
    SDWebImageDownloadToken *token = [SDWebImageDownloadToken new];
    token.downloadOperation = operation;
    token.url = url;
    token.downloadOperationCancelToken = downloadOperationCancelToken;
    return token;
    ```

-   `-[createDownloaderOperationWithUrl: options:]`：

    这个方法创建并返回了一个实现了`SDWebImageDownloaderOperationInterface`协议的`NSOperation`类型的对象：

    ```objective-c
    NSOperation<SDWebImageDownloaderOperationInterface> *operation = [[self.operationClass alloc] initWithRequest:request inSession:self.session options:options];
    ```

    这里所用的`operationClass`可以由用户自定义，只要继承自`NSOperation`并实现`SDWebImageDownloaderOperationInterface`协议就可以了。默认情况下使用的是`SDWebImageDownloaderOperation`这个类型。

    另外创建网络请求用的`NSURLSession`也是保存在 Downloader 里面的，Downloader 实现了`NSURLSessionTaskDelegate`协议，并将相应的方法**转发**到相应的`Operation`实例中。

    转发方法时，会通过`-[operationWithTask:]`这个方法来找到 Task 所对应的 Operation：

    ```objective-c
    - (NSOperation*)operationWithTask:(NSURLSessionTask *)task {
        NSOperation *returnOperation = nil;
        // 遍历downloadQueue中的所有operation
        for (NSOperation *operation in self.downloadQueue.operations) {
            if ([operation respondsToSelector:@selector(dataTask)]) {
                if (operation.dataTask.taskIdentifier == task.taskIdentifier) {
                    returnOperation = operation;
                    break;
                }
            }
        }
        return returnOperation;
    }
    ```

    这里就是一个 O(n)的线性遍历，看起来性能会比较差，但是考虑到`downloadQueue`只是一个并发量最高为**6**的队列，所以实际上对性能基本不会有什么影响。

### SDWebImageDownloaderOperation

`SDWebImageDownloaderOperation`是实现具体下载逻辑的地方，实现了`NSURLSessionTaskDelegate`相关的方法（由`SDWebImageDownloader`调用）。

负责请求后台 ID、保存下载过程的数据、对结果进行解码、回调业务方等等逻辑。

### SDWebImageOperation

`SDWebImageOperation`是一个协议，里面只包含了一个`cancel`方法：

```objective-c
@protocol SDWebImageOperation <NSObject>

- (void)cancel; // 用来退出异步操作

@end
```

`SDWebImageOperation`用来封装 SD 中的**异步操作**，提供退出异步操作的接口，网络和文件的异步操作都被封装成`SDWebImageOperation`，在 SD 中有三个不同类型的`Operation`实现了这个协议：

![SD的Operation](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/SDWebImage/aa1275779c22a15fbe96842f664ed58d.png)

-   **SDWebImageDownloaderOperation：**

    之前已经接介绍了`SDWebImageDownloaderOperation`是实现图片下载的类型，`cancel`会将 operation 的状态置为结束，并移除掉所有的上层请求的回调。

    它还提供了另外一个 cancel 方法：`- (BOOL)cancel:(nullable id)token;`。像前面介绍的，多个上层的相同请求会共用一个 operation 对象，`token`标识了某个具体的上层请求，这个方法仅将对应请求的回调移除，并不会真正的结束这个请求（所有的回调都被移除了才会结束）。

-   **SDWebImageDownloadToken：**

    在使用`SDWebImageDownloader`创建一个请求的时候会返回这个类型的对象，它里面保存了`operation`和这一次请求相关的`token`，它的`cancel`方法是这样实现的：

    ```objective-c
    - (void)cancel {
        if (self.downloadOperation) {
            SDWebImageDownloadToken *cancelToken = self.downloadOperationCancelToken;
            if (cancelToken) {
                [self.downloadOperation cancel:cancelToken]; // 将当前请求的callback从operation中移除
            }
        }
    }
    ```

    它并没有调用`operation`的全量`cancel`方法，只是取消了指定请求的回调而已，所以说`SDWebImageDownloadToken`是面向上层请求的类型。

-   **SDWebImageCombinedOperation：**

    回到最上层的`SDWebImageManager`中的接口`-[loadImageWithURL:...]`，它的方法签名中返回值的类型是`id<SDWebImageOperation>`，实际上返回的是`SDWebImageCombinedOperation`类型的实例。`SDWebImageManager`中整合了缓存和网络相关的操作，查询磁盘缓存和网络请求的异步操作分别封装到了不同的`Operation`中，这两个 Operation 被保存到一个`SDWebImageCombinedOperation`的实例中返回，它的`cancel`方法是这样的：

    ```objective-c
    - (void)cancel {
        @synchronized(self) {
            self.cancelled = YES;
            // 1. 退出缓存查询
            if (self.cacheOperation) {
                [self.cacheOperation cancel];
                self.cacheOperation = nil;
            }
            // 2. 退出网络请求
            if (self.downloadToken) {
                [self.manager.imageDownloader cancel:self.downloadToken];
            }
            [self.manager safelyRemoveOperationFromRunning:self];
        }
    }
    ```

    用户调用`cancel`后会将涉及到的 Operation 全部退出。

## 编码/解码

SD定义了一个`SDWebImageCoder`协议来描述图片加解码的接口：

```objective-c
@protocol SDWebImageCoder <NSObject>

@required
- (BOOL)canDecodeFromData:(nullable NSData *)data;
- (UIImage *)decodedImageWithData:(nullable NSData *)data;
- (UIImage *)decompressedImageWithImage:(UIImage *)image
                                   data:(NSData **)data
                                options:(NSDictionary<NSString*, NSObject*>*)optionsDict;

- (BOOL)canEncodeToFormat:(SDImageFormat)format;
- (NSData *)encodedDataWithImage:(UIImage *)image format:(SDImageFormat)format;

@end
```

其中`-[decodedImageWithData:]`方法的作用是将数据解析成`UIImage`类型的图像，但这样生成的图像通常还是带压缩的格式，并不能表示实际的位图，这时可以`decompressedImageWithImage`用于对这个`UIImage`进行解压转换成实际的位图，避免显示的时候在主线程解码耗时。

另外还有一个**继承自**`SDWebImageCoder`的协议：`SDWebImageProgressiveCoder`，用来描述渐进式解码的接口：

```objective-c
@protocol SDWebImageProgressiveCoder <SDWebImageCoder>

@required
- (BOOL)canIncrementallyDecodeFromData:(NSData *)data;
- (UIImage *)incrementallyDecodedImageWithData:(NSData *)data finished:(BOOL)finished;

```

SD中提供了三个不同的Coder：`SDWebImageWebPCoder`、`SDWebImageImageIOCoder`和`SDWebImageGIFCoder`，另外还有一个`SDWebImageCodersManager`类型的单例来管理所有的Coder。

### SDWebImageCodersManager

`SDWebImageCodersManager`同样实现了`SDWebImageCoder`协议，不过不同的是它并没有实现具体的解码逻辑，而是将方法转发到它保存的`Coders`上，比如说`decodedImageWithData:`：

```objective-c
- (UIImage *)decodedImageWithData:(NSData *)data {
    LOCK(self.codersLock);
    NSArray<id<SDWebImageCoder>> *coders = self.coders;
    UNLOCK(self.codersLock);
    // 找到一个可以解析的Coder
    for (id<SDWebImageCoder> coder in coders.reverseObjectEnumerator) {
        if ([coder canDecodeFromData:data]) {
            return [coder decodedImageWithData:data];
        }
    }
    return nil;
}
```

属性`coders`是一个数组，保存了不同的Coder，从后往前找到第一个可以处理当前图片数据的Coder，然后通过该Coder的方法来解码，注意这里用的是**逆序遍历**，所以后面添加的Coder会先进行处理，也就是说你可以添加一个自定义的Coder来覆盖SD默认的实现。

Manager中默认加载了`SDWebImageImageIOCoder`和`SDWebImageWebPCoder`这两个。

- **SDWebImageWebPCoder**:

  仅处理`WebP`格式图片的加解码，并且支持渐进式解码；

- **SDWebImageImageIOCoder**：

  支持除了`WebP`以外的所有类型，包括`PNG`、`JPG`、`GIF`、`HEIF`、`HEIC`等等，`HEIF`和`HEIC`是高效率图像格式的缩写，也就是iOS相机默认采用的文件格式，生成的图像文件较小、质量高，并且还可以储存声音。这种格式并不是所有版本的iOS系统都支持，SD通过`CGImageSourceCopyTypeIdentifiers()`获取系统支持的图片类型来进行判断。

  这些类型的图片都是`UIImage`支持的格式，所以`decodeImageWithData:`方法的实现非常简单：

  ```objective-c
  - (UIImage *)decodedImageWithData:(NSData *)data {
      if (!data) {
          return nil;
      }
      
      UIImage *image = [[UIImage alloc] initWithData:data]; // 直接生成图片
      image.sd_imageFormat = [NSData sd_imageFormatForImageData:data];
      
      return image;
  }
  ```

  但是这种方式加载`GIF`图片只能显示第一帧的**静态图**。

- **SDWebImageGIFCoder**：

  `SDWebImageGIFCoder`利用`ImageIO`提供了对GIF动态图的支持，这个Coder在默认情况下**并没有**添加到`CoderManager`中，如果想要为**所有**的图片添加GIF支持可以手动将它添加到Manager中。

  但是对于GIF还是推荐使用`FLAnimatedImage`来实现，因为它的性能更高，SD中同样为`FLAnimatedImageView`类型提供了加载图片的上层接口。