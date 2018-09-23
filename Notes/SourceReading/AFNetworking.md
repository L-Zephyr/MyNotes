# AFNetworking

## AFURLSessionManager

### 类图

![AF1](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/AFNetworking/8a1b103dc7ec75422464fe8d5b77e682.png)

`AFURLSessionManager`是对`NSURLSession`的封装，通过一个`NSURLSessionConfiguration`配置对象来初始化。它是AFNetworking中最**关键**的一个类型，它负责创建实际的网络请求对象，从类图中可以看到它同时实现了三种Task的Delegate方法。

在NSURL Session中有**三种**Task来表示不同的网络请求类型，分别为Data、Upload、Download。`AFURLSessionManager`提供了多个不同的方法来创建这三种的Task：

```objective-c
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request ...;
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request ...;
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request ...;
...
```

这些方法的实现方式都想差不多，精简后如下所示：

```objective-c
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request ... {
	NSURLSessionDataTask *task = self.session...; // 通过NSURLSession实例创建task
    // ...
    [self addDelegateForXXTask:task ...]; // 设置Delegate
    return task;
}
```

`AFURLSessionManager`为这三种Task提供三个方法来设置它们的Delegate，分别为`addDelegateForDownloadTask`、`addDelegateForDataTask`、`addDelegateForUploadTask`。在这些方法中创建了一个类型为`AFURLSessionManagerTaskDelegate`的对象将这个Task实例包装起来，而这个Delegate对象保存在Manager的一个字典中，以Task的`taskIdentifier`做为Key值：

```objective-c
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task {
    // 一个普通的锁来保证多线程的安全
    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [self addNotificationObserverForTask:task];
    [self.lock unlock];
}
```

### AFURLSessionManagerTaskDelegate

`AFURLSessionManagerTaskDelegate`和`AFURLSessionManager`一样都实现了所有Task类型的Delegate，但不同的是`AFURLSessionManager`才是URLSession**真正的代理对象**。当Manager中的代理方法被调用的时候，会在它的字典中通过Task的`taskIdentifier`来找到对应的Delegate对象，并**手动调用**其相应的代理方法。

当Manager中的`- [URLSession: task: didCompleteWithError:]`被调用的时候表示这个Task结束了，此时将该Task对应的Delegate从字典中移除：

```objective-c
- (void)removeDelegateForTask:(NSURLSessionTask *)task {
    [self.lock lock];
    [self removeNotificationObserverForTask:task];
    [self.mutableTaskDelegatesKeyedByTaskIdentifier removeObjectForKey:@(task.taskIdentifier)];
    [self.lock unlock];
}
```

一个请求各个阶段的逻辑是由`AFURLSessionManager`和`AFURLSessionManagerTaskDelegate`互相配合来完成的：

**AFURLSessionManagerTaskDelegate**负责：

- 保存传输过程中的数据，以及Upload Task、Download Task的进度；
- 如果是Download Task，在文件下载成功后将文件从缓存区移动到目标位置；


- 在适当时机调用业务方的Block（progress、download、complete...）进行回调以及发送通知；
- 请求成功后通过Manager的`responseSerializer`对结果进行序列化。其中所有请求成功时**都会派发到同一个并行队列**中进行序列化工作（`url_session_manager_processing_queue()`）。

可以看到，跟请求自身相关的大部分逻辑都被封装到了`AFURLSessionManagerTaskDelegate`这个对象中。

**AFURLSessionManager**负责：

- 请求开始时为每个Task创建Delegate实例并保存在manager中；
- Data Task被转换成Download Task时更新相应的Delegate；
- 挑战/应答的处理；
- Manager提供了一些Block可以对所有的请求进行全局的处理，Manager会优先通过Block来进行处理；如果业务方没有设置才会进一步转发给Delegate处理（比如说`downloadTaskDidFinishDownloading`）;
- 请求结束后移除该Task对应的Delegate实例；

总的来看，Manager的逻辑位于Delegate逻辑的上层，它管理所有的Delegate，负责处理一些**全局**的操作，而**只跟Task自身相关**的操作则被封装到了Delegate中，这一层封装使得逻辑变得更加清晰。

### suspend和consume事件

AF中**Swizzle**了`NSURLSessionDataTask`类型的`resume`和`suspend`方法，在相应方法调用的时候发送通知：

```objective-c
- (void)af_resume {
    NSURLSessionTaskState state = [self state];
    [self af_resume];
    
    // Not Running -> Running
    if (state != NSURLSessionTaskStateRunning) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
    }
}
```

分别为：

- `AFNSURLSessionTaskDidResumeNotification`

- `AFNSURLSessionTaskDidSuspendNotification`

这两个通知，它们在各自Task的**子线程中**发送，Manager会监听所有Task的通知，并分别将其转换成：

- `AFNetworkingTaskDidResumeNotification`
- `AFNetworkingTaskDidSuspendNotification`

通知在**主线程**中发送出去。上层应用可以通过这两个通知监听到所有网络请求的`resume`和`suspend`事件。

> Task中的`suspend`方法会暂停个请求，稍后可以通过`resume`继续，被挂起的请求不会超时；一个Download Task在`resume`的时候可以接着上一次的进度继续下载，其他类型的Task则会重新开始请求。

## AFHTTPSessionManager

`AFHTTPSessionManager`继承自上面提到的`AFURLSessionManager`，提供了HTTP相关方法的封装：GET、POST、PATCH、HEAD、DELETE：

![AFHTTPSessionManager](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/AFNetworking/a707ad283b4065bcc125e859cd533048.png)

对于普通的HTTP方法，`dataTaskWithHTTPMethod`会通过`AFHTTPRequestSerializer`来创建一个URLRequest，然后调用父类的方法创建一个`DataTask`并开始请求。

而对于`multipart`类型的POST请求，则会创建一个`UploadTask`请求用来上传文件，RequestSerializer中对于multipart的处理也不一样。

![AFHTTPSessionManager2](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/AFNetworking/9ef19bfd16e490a27c1ac1eb534dbee8.png)

## 序列化

序列化指的是创建一个Request以及从一个Response中解析出目标数据的过程，AF分别定义了两个协议以及许多个不同的类型来封装序列化的过程。

### AFURLRequestSerialization

这是AF中定义的一个**协议**，只有一个方法：

```objective-c
- (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(nullable id)parameters
                                        error:(NSError **)error;
```

它的作用是用来**创建请求对象**，AF中自带了几种不同的RequestSerializer类型，他们的类图如下：

![AFHTTPSessionManager3](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/AFNetworking/e1993bc49a0f6be102d0d9c508a691fd.png)

#### AFHTTPRequestSerializer

`AFHTTPRequestSerializer`封装了创建一个HTTP请求的所有操作，包括设置超时时间、设置参数、请求头、Body等。

对于`multipart/form-data`格式，首先它是一个POST请求，不同的是它的Body中可以同时包含多个不同的内容（比如说多个文件），所以称为“multipart”，在body中不同部分的数据被`boundary`分割开来，AF中专门封装了一个`AFMultipartFormData`协议和一个`AFStreamingMultipartFormData`类型来创建Multipart数据：

![AFHTTPSessionManager4](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/AFNetworking/8d94c60f525e635894de7ef03ba88c51.png)

`AFMultipartFormData`中定义了一系列用来为HTTP Body添加数据的方法，如：

```objective-c
// 添加一个文件
- (BOOL)appendPartWithFileURL:(NSURL *)fileURL
                         name:(NSString *)name
                        error:(NSError * _Nullable __autoreleasing *)error;
// 添加一段数据
- (void)appendPartWithFormData:(NSData *)data
                          name:(NSString *)name;
// ...等等
```

`AFStreamingMultipartFormData`为这些接口提供了具体的实现。

在`AFHTTPRequestSerializer`中有这样一个方法：

```objective-c
- (NSMutableURLRequest *)multipartFormRequestWithMethod:(NSString *)method
                                              URLString:(NSString *)URLString
                                             parameters:(NSDictionary *)parameters
                              constructingBodyWithBlock:(nullable void (^)(id <AFMultipartFormData> formData))block
                                                  error:(NSError **)error;
```

其中有一个Block`constructingBodyWithBlock`中传入的参数就是`AFStreamingMultipartFormData`类型的对象，**用户通过这个对象自行像请求中添加multipart数据**，最后返回最终的URLRequest。

#### JSONSerializer和PropertyListSerializer

在`AFHTTPRequestSerializer`之上还有两个不同的Serializer：`AFJSONRequestSerializer`和`AFPropertyListRequestSerializer`。他们分别将HTTP Body中的数据格式化成JSON和plist，并设置正确的`Content-Type`。

对于`Content-Type`：

- 普通的HTTP请求：`application/x-www-form-urlencoded`；
- 请求的数据格式为JSON：`application/json`；
- 请求的数据格式为plist：`application/x-plist`；
- multipart请求：`multipart/form-data; boundary=xxxx`；其中boundary是通过`AFCreateMultipartFormBoundary`方法连续取两次随机数创建出来的；

### AFURLResponseSerialization

![AF4](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/AFNetworking/92a2ebb4113bdb320acae8c5de0a03f1.png)

类似Request，AF对于网络请求返回数据的序列化也是通过一个协议`AFURLResponseSerialization`和一个类型`AFHTTPResponseSerializer`来完成的。

该协议同样也只定义了一个方法：

```objective-c
- (id)responseObjectForResponse:(NSURLResponse *)response
                           data:(NSData *)data
                          error:(NSError **)error;
```

通过请求返回的`data`创建特定的对象。

#### AFHTTPResponseSerializer

与RequestSerializer不同的是，`AFHTTPResponseSerializer`仅包含了最基本的逻辑，它负责对返回的数据进行校验（包括Content-Type和StatusCode的校验）以及持久化相关的逻辑（NSCoding）。

剩下大部分的工作被交给了它的子类，从类图中可以看到，针对不同类型的数据，`AFHTTPResponseSerializer`上总共派生了6种不同的子类。

#### AFHTTPResponseSerializer的子类

- **AFImageRespsonseSerializer**

  通过返回的数据创建`UIImage`对象；

- **AFPropertyListResponseSerializer**

  将数据通过`NSPropertyListSerialization`进行解析，返回结果；

- **AFXMLDocumentResponseSerializer**

  用数据创建并返回一个`NSXMLDocument`对象；

- **AFXMLParserResponseSerializer**

  用数据创建并返回一个`NSXMLParser`对象；

- **AFJSONResponseSerializer**

  通过`NSJSONSerialization`返回格式化后的对象，通过`removesKeysWithNullValues`可以将结果为`NSNull`的键值对删除；

- **AFCompoundResponseSerializer**

  不代表某种具体的Serializer，在初始化时可以设置多个Serializer，依次应用只要有一个解析成功就返回成功的结果；

## AFSecurityPolicy和SSL Pinning

### HTTPS

首先来简单的回顾一下HTTPS的原理，HTTPS就是HTTP加上SSL（或TLS）。HTTPS是一种混合的加密方案，在开始传输之前客户端需要和服务器交换信息以确定加密的算法和密钥等参数，这个过程叫做**SSL握手**。

简单来看SSL握手需要经过以下几步

![SSL握手](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/AFNetworking/6fe51d62abd8aea75d8fda36b9cf4163.png)

1. 客户端生成一个**随机数1（Client Random）**，以**明文**的形式与自己支持的加密算法一起发送给服务器；
2. 服务器收到客户端的随机数1，选择一个加密算法，生成一个**随机数2（Server Random）**，加上自己的证书，以**明文**的形式发送给客户端 ；
3. 客户端验证证书的合法性；
4. 客户端再生成一个**随机数3（Pre-master key）**，用证书中的公钥加密，发送给服务器，此时客户端将这三个随机数组合起来就是后续通信所用的密钥；
5. 服务端拿到Pre-master的密文，用自己的私钥解密，同样得到后面的对话密钥；

至此客户端和服务器交换密钥的过程就完成了，密钥交换是通过**非对称加密**算法来完成，而之后传输的HTTP数据都通过这个密钥来进行**对称加密**以提高效率；

### 中间人攻击

中间人攻击正是市面上各种HTTPS抓包软件的实现原理，在SSL握手的前两步，客户端发送的随机数1以及服务端返回的随机数2和证书都是以**明文**的形式传输的。中间的代理可以**截获**服务器返回的证书，并向客户端返回一个假的证书，客户端用假的证书加密`Pre-master secret`，中间人得到`Pre-master`后再用**服务器的证书**加密发送给服务器，这样一来中间人就知道了客户端和服务器的通信密钥。

这里关键在于客户端对于证书的验证上，所以在抓HTTPS的包之前通常需要在设备上安装证书并手动添加信任。

### SSL Pinning

`SSL Pinning`是用来预防中间人攻击的一种方式，也就是在开发时就将服务器的证书一起打包到客户端中，这样在SSL握手的时候就可以对比服务端证书的一致性，从而识别出中间人攻击。

`SSL Pinning`能够很有效的防止中间人攻击，但这也不是绝对安全的方法，攻击者可以通过逆向应用中相关的认证方法来绕过`SSL Pinning`的证书校验来实现中间人攻击。

### AFSecurityPolicy

AF提供了一个类型`AFSecurityPolicy`，可以很方便的实现`SSL Pinning`，`AFSecurityPolicy`有三种验证模式：

```objective-c
typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
    AFSSLPinningModeNone,
    AFSSLPinningModePublicKey,
    AFSSLPinningModeCertificate,
};
```

- **AFSSLPinningModeNone**：

  不对证书做任何校验，不做任何配置时的默认值；

- **AFSSLPinningModeCertificate**：

  对证书的整体进行校验，比较麻烦，如果证书有变动或证书过期会导致校验失败；

- **AFSSLPinningModePublicKey**：

  只验证证书中的公钥，这种方式比较灵活一些，即使服务端的证书有所变动，只要公钥没有变动就可以通过验证，通常来说比较建议使用这种验证模式；

对于上层的业务方来说，只需要设置一下`AFURLSessionManager`中的`securityPolicy`属性，选择一个校验类型，然后将证书打包进App中。`AFSecurityPolicy`在初始化的时候会自动将Bundle中的所有`.cer`文件加载进来。

#### 校验

在实际请求中是通过`挑战-应答（Challenge-Response）`机制来进行校验的，在iOS中`挑战-应答`相关的代理方法被用来进行用户身份校验以及SSL握手。`NSURLSessionDelegate`和`NSURLSessionTaskDelegate`都有相关的方法：

- `NSURLSessionDelegate`中的`URLSession:didReceiveChallenge:completionHandler:`
- `NSURLSessionTaskDelegate`中的`URLSession:task:didReceiveChallenge:completionHandler:`

这两个协议分别处理`URLSession`层面的和`Task`层面的挑战应答，AF实现了这两个协议，调用`AFSecurityPolicy`的：

```objective-c
- (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                  forDomain:(NSString *)domain;
```

方法来校验证书的合法性。

## Q&A

- `AFImageResponseSerializer`中的`automaticallyInflatesResponseImage`有何作用？

  JPG/PNG是一种经过压缩的图片格式，直接通过`[UIImage imageWithData:]`初始化一个`UIImage`并不会对图片进行解码，而是显示的时候才会在主线程去解码，这样就可能会造成卡顿。

  ```objective-c
  if (self.automaticallyInflatesResponseImage) {
          return AFInflatedImageFromResponseWithDataAtScale((NSHTTPURLResponse *)response, data, self.imageScale);
      } else {
          return AFImageWithDataAtScale(data, self.imageScale);
      }
  ```

  开启了`automaticallyInflatesResponseImage`会在图片下载完成后在子线程中将图片解压，这样在显示的时候就可以直接渲染到屏幕上。

- 为什么`AFImageResponseSerializer`在调用`[UIImage imageWithData:]`的时候要加锁？

  AF中根据`NSData`创建`UIImage`的时候是通过一个自定义的方法来实现的：

  ```objective-c
  + (UIImage *)af_safeImageWithData:(NSData *)data {
      UIImage* image = nil;
      static dispatch_once_t onceToken;
      dispatch_once(&onceToken, ^{
          imageLock = [[NSLock alloc] init];
      });
      
      [imageLock lock];
      image = [UIImage imageWithData:data];
      [imageLock unlock];
      return image;
  }
  ```

  这里对系统的`[UIImage imageWithData:]`方法加了同步锁，因为这个方法是在子线程下载图片完成后调用的，`[UIImage imageWithData:]`不是一个线程安全的方法。

