# 基础

## SSH

对于iOS10以下的系统，越狱之后直接使用Cydia自带的`OpenSSH`即可，但是在iOS10及以上的系统中由于底层的修改导致`OpenSSH`无法使用，需要使用`DropBear`作为`OpenSSH`的替代品。使用`Yalu`越狱的设备已经自带了`DropBear`，使用其他工具越狱的设备需要自行安装，详细方法见[iOS10系统SSH](https://zhuanlan.zhihu.com/p/34003393)。

### 通过USB连接

建议通过USB来进行连接以获得最稳定的速度，首先需要使用`iproxy`来进行端口映射，`iproxy`可以通过安装工具包`usbmuxd`来获得：

```shell
brew install usbmuxd
```

之后可以将本地的`2222`号端口映射成SSH使用的`22`号端口：

```shell
iproxy 2222 22
```

在SSH的时候指定使用`2222`号端口即可：

```shell
ssh -p 2222 root@localhost
```

### Dropbear SSH连接不上？

使用中碰到了这样的报错：`Connection closed by 127.0.0.1`。解决方法是打开设备上的`/Library/Daemon/dropbear.plist`文件，将最后一个参数`22`修改成`127.0.0.1:22`，然后`kill`掉dropbear的进程并重启`/usr/local/bin/dropbear`，然后就能正常连接了。

### 文件传输

在手机和Mac之间互传文件可以使用`scp`（手机和电脑都要安装），大部分情况下并没有预装，可以从网上下载一份并拷贝到手机上的`/usr/local/bin`中。不过我逆向时使用的手机是iPhone5，并没有找到适合32位设备的scp，所以这里使用`rsync`来代替。

手机可以直接使用Cydia来安装`rsync`，Mac上直接使用brew来安装即可：`brew install rsync`

- **从Mac拷贝文件到手机**

  ```shell
  rsync -rvz -e 'ssh -p 2222' --progress 本地文件 root@localhost:手机的目标位置
  ```

- **从手机拷贝文件到Mac**

  ```shell
  rsync -rvz -e 'ssh -p 2222' --progress root@localhost:手机上的目标文件 Mac上的目标位置
  ```

这里指定了`ssh`端口为2222，因为我在前面已经使用`iproxy`将22号端口映射到2222了，如果没有使用`iproxy`则不需要加`-e 'ssh -p 2222'`这个参数。上面这两条命令都是在Mac端执行的。

## App签名

### 非对称加密

非对称加密需要两个密钥：公钥和私钥。公钥和私钥是一对，使用**公钥**加密的内容用**私钥**才能解开，使用私钥加密的内容用**公钥**才能解开。

这种算法常用来进行机密信息的交换：甲方生成一对密钥并将其中一个作为**公钥**对外公开，乙方使用这个公开的密钥加密数据，甲方拿到密文后使用**私钥**来解密得到明文。

### 数字签名

首先生成一对公私钥，公钥发布出去，私钥自己保存着。发送数据的摘要（通常用md5），然后使用自己的私钥对其进行加密，得到的密文就是这份数据的**签名**。其他人拿到数据和签名后，用公钥进行解密，然后在原始数据上再计算一次摘要，比较这两个摘要的值，相等则说明数据没有被篡改。

### App签名原理

App签名的过程涉及到两对公私钥，一对是苹果自己的，另一对是自己生成的：

1. 首先在Mac上生成一对公私钥，也就是在钥匙串中选择"从证书颁发机构请求证书"，得到的`CertificateSigningRequest`文件就是公钥，私钥保存在电脑中，这里成为`公钥L`和`私钥L`；

2. 苹果自身有一对公私钥，私钥在苹果的后台，公钥在每一台iOS设备上，这里称为`公钥A`和`私钥A`；

3. 将`公钥L`传到苹果的后台（也就是`CertificateSigningRequest`文件），然后苹果会用自己的`私钥A`去签名这个`公钥L`，然后得到一份包含了`公钥L`及其签名的数据，这个数据被称为`证书`。此时就可以将证书从苹果的后台下载回来了，之后这个证书会跟第一步生成的私钥关联起来；

4. 然后需要在苹果的后台申请AppID、配置设备ID和各种权限（Entitlements文件），再加上第三步得到的证书在一起用`私钥A`签名，这些数据和签名在一起最后得到的一个`Provisioning Profile`文件，下载到Mac上；

5. 在开发的时候会用自己的`私钥L`对App进行签名，这个签名和上一步的`Provisioning Profile`一起打包进App，也就是`embedded.mobileprovision`文件，Mach-O可执行文件会直接将签名写在里面，其余的资源文件会保存在`_CodeSignature`目录下；

6. 安装到设备上之后，iOS系统会先用自己的`公钥A`去验证`Provisioning Profile`的签名是否正确；证书中包含了我们自己的`公钥L`，首先iOS会先用`私钥A`来验证证书的签名，最后用证书中的`公钥L`来验证App自身的签名，同样其他各种权限也在这里验证；

7. 从上面的流程中可以看到，App是用自己生成的`私钥L`来签名的，这个`私钥L`只保存在生成的它的那台Mac上，所以如果想要让其他Mac也能打包，需要将本地`私钥L`导出，这就是`p12`文件；

可以看到App的签名和验证都是通过Mac本地生成的公私钥来完成，而苹果自身公私钥的作用就是保证本地公钥的正确性。

### 重签名

对于已经越狱的设备来说比较简单，可以使用`ldid`这样的工具直接对可执行文件进行签名，然后拷贝回设备覆盖原有的位置即可。对于非越狱设备来说则需要对ipa重签名之后再通过Xcode或iTools安装到设备上。

# 静态分析

## 砸壳

上传到`App Store`中的应用会被进行加密，这个过程也称为**加壳**，在运行的时候才会动态的解密，进过加壳应用的可执行文件是无法直接通过后面的`class-dump`和`Hopper`之类的方法进行分析的。所以逆向分析的第一步就是对App进行解密，这个过程称为**砸壳**（或脱壳）。

### 获取应用的BundleId

#### 1. 通过plist文件获取

首先用SSH连接上手机，运行目标应用，然后使用命令：`ps -e`查看当前运行进程的可执行文件的路径，通过`grep`筛选出App Store目录下的应用：

```shell
ps -e | grep Application
```

> App Store中下载的应用位于`/var/containers/Bundle/Application`目录下，`grep`是一个文本搜索工具

得到目标应用的安装路径：`/var/containers/Bundle/Application/2580EDA5-526F-43C8-8689-67D355A010EB/Target.app/Target`。应用对应的plist文件也在这个目录中，通过grep搜索得到Bundle Id：

```shell
cat /var/containers/Bundle/Application/2580EDA5-526F-43C8-8689-67D355A010EB/Target.app/Info.plist | grep CFBundleIdentifier -A 1
```

> `cat`命令用来打印文件内容，`grep -A 1`参数的作用是除了目标位置的那一行以外多打印一行

#### 2. 通过Clutch获取

`Clutch`是一个砸壳用的工具，安装到设备上之后直接在设备的命令行中执行

```shell
Clutch -i
```

就可以看到所有第三方应用的BundleId了，非常简单

### 获取应用的沙盒位置

从AppStore下载的应用运行在沙盒中，应用运行中只能访问沙盒中的文件，在得到Bundle ID后可以通过私有API来获取应用的`Document`目录，只要在Xcode中创建一个iOS应用并运行下面的代码：

```objective-c
NSURL *dataUrl = [[NSClassFromString(@"LSApplicationProxy") performSelector:@selector(applicationProxyForIdentifier:) withObject:@"上一步中得到的BundleId"] performSelector:@selector(dataContainerURL)]
```

就能得到应用的沙盒位置了:

```shell
/private/var/mobile/Containers/Data/Application/808C8F65-EFB3-4D5C-A1BA-ABE54E378749
```

### 解密

首先一个iOS的可执行文件是否被加密可以通过以下命令来判断：

```shell
otool -l 可执行文件 | grep -A 4 LC_ENCRYPTION_INFO
```

会看到这样输出：

![WX20180806-194821](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/be3b164b6381535a3826982147f07fd8.png)

已加密的文件`cryptid`为1，未加密的文件为0。


#### dumpdecrypted

首先获取[dumpdecrypted.dylib](https://github.com/stefanesser/dumpdecrypted)动态库文件，通过`scp`或`rsync`之类的工具将`dumpdecrypted.dylib`复制到目标应用沙盒中的`Document`目录下，然后通过`DYLD_INSERT_LIBRARIES`标志来注入我们的动态库：

``` shell
$ cd 目标应用沙盒中的Documents目录
$ DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib 目标应用的可执行文件
```

然后会在当前文件夹中生成一个后缀为`.decrypted`解密后的文件，将该文件拷贝到Mac中。

只有该设备的架构会被解密，所以还需要通过`lipo`抽取指定的架构，比如说32位的armv7：

```shell
lipo Target.decrypted -thin armv7 -output Target.armv7
```

最后的`Target.armv7`就是所需要的解密后的文件。

这种方式比较繁琐，可以使用这个[修改过的dumpdecrypted](https://github.com/AloneMonkey/dumpdecrypted)来简化一些流程。

#### Clutch

`Clutch`是另一种砸壳的方法，使用起来更加方便，设备安装了`Clutch`后直接通过命令：

```shell
Clutch -b 目标应用的BundleID
```

直接得到解密后的文件，会保存在指定的位置，将其拷贝到Mac上即可，这种砸壳方法最为简单，不过有时候会失败，这个时候再改用上一种方法来处理即可。

## class-dump

`class-dump`是一个可以从可执行文件中获取类、方法和属性信息的工具。将应用的解密后的可执行文件提取出来之后，导出指定架构的头文件到指定目录：

```shell
class-dump --arch armv7 TargetApp -A -H -o ../Headers
```

> `—arch`指定目标架构；`-H`生成头文件；`-o`指定目标文件夹，`-A`可以显示方法在文件中的实现地址，根据该地址可以直接在Hopper中找到对应实现位置。

之后会生成一堆头文件，包含了应用中所有的类型，以及类型中的方法和属性。

对于Swift的类型，为了避免命名冲突，编译时会在类名上添加其它的信息，`class-dump`之后看到的名称是其在运行时的名称，比如说工程中一个名为`DiscountTipView.swift`的类型在运行时的名称变成了`_TtC14NECatoonReader15DiscountTipView`，可以通过`xcrun`来将其格式化：

```shell
xcrun swift-demangle -compact _TtC14NECatoonReader15DiscountTipView
```

得到格式化后的名称为`NECatoonReader.DiscountTipView`，swift通过这样的机制避免不同库之间的命名冲突。

## Cycript

`cycript`是一个可以在运行时动态修改内存信息的工具，支持OC++和JavaScript的组合语法。可以在常规开发中使用，也可以在越狱设备上动态注入应用进程。

在设备上通过以下命令注入目标应用的进程：

```shell
cycript -p 进程的名称
```

然后会打开一个交互式界面，可以在这里执行代码：

```shell
cy# var alert = [[UIAlertView alloc] initWithTitle:@"hello" message:nil delegate:nil cancelButtonTitle:"cancel" otherButtonTitles:nil, nil]
cy# alert.show()
```

应用的界面上会弹出一个对话框。

使用`Cycript`注入程序，可以在运行中调用应用的方法，还可以查看应用的一些基本信息，如沙盒目录、BundleId、布局等等。在执行脚本的时候要确保应用在前台运行，最后通过`?exit`命令来退出。

### 一些指令

- `UIApp`：查看当前的`UIApplication`实例；

- `choose()`：查找界面上某个类型的所有实例，如：`choose(UIButton)`查找所有UIButton的实例：

  ```shell
  cy# choose(UIButton)
  [#"<UIButton: 0x1732ea80; frame = (0 0; 70 40); opaque = NO; autoresize = RM+BM; layer = <CALayer: 0x1732e510>>"...]
  ```

  之后可以直接通过它的地址修改Button的属性：

  ```shell
  0x1732ea80.hidden = true
  ```

- ...

## Hopper

### 用Hopper分析静态库

一般静态库文件里面会包含模拟器和真机的架构，如下指令可以查看静态库包含的架构信息：

```shell
$ lipo -info libWeiboSDK.a
Architectures in the fat file: libWeiboSDK.a are: armv7 arm64 i386 x86_64
```

在分析的时候只需要取其中的一个架构即可，可以使用命令将指定架构的部分提取出来：

```shell
lipo libWeiboSDK.a -thin arm64 -output libWeiboSDK_arm64
```

静态库就像相当于一个压缩包，里面包含了所有OBJECT文件（后缀为.o），用`ar`命令可以将所有`.o`解压出来：

```shell
ar -x ./libWeiboSDK_arm64
```

之后还可以通过`grep`命令来在所有`.o`文件中查找特定的字符以辅助分析：

```shell
grep "something" -rn ./
```

此外，为了方便分析还可以将所有的Object文件链接上一个Object文件，首先要确定Object文件是否包含bitcode：

```shell
otool -l XXX.o | grep bitcode
```

如果不包含bitcode则不会有输出。

然后通过`ld`命令链接成一个Object文件：

```shell
ld -r -arch arm64 -syslibroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk -bitcode_bundle ./*.o -o ../output
```

**如果不包含bitcode则不用添加`-bitcode_bundle`参数**，然后就能在上级目录看到一个名为`output`的文件了。上面这一长串的是iPhoneSDK的地址，可以在Xcode的包内容中找到。

### ARM汇编相关

#### 寄存器

iPhone设备都是ARM架构，在调用函数时，ARM汇编使用`r0`~`r7`这几个寄存器来传递参数（多余的参数通过栈来传递），函数的返回值通常保存在`r0`寄存器中。（注意只有ARM汇编使用这样的规则，在模拟器上运行的程序是使用的是`x86`汇编）

#### 指令

- **ldr**

  取指定地址上的值并将其保存到目标寄存器；

  像`ldr x0, x20, x8`或`ldr x0, [x20, x8]`表示将寄存器`x20`处的地址加上`x8`处的偏移量后赋值给`x0`，这种形式常见于取对象中的某个属性，属性可以根据偏移量来判断；

- **mov**

  赋值指令，如`mov x0 x22`将寄存器x22上的值赋给寄存器x0；

- **bl**

  一种跳转指令，跳转到指定的地址上，通常用来执行函数调用。在OC中所有的方法调用都是`objc_msgSend`，并且带有两个参数：调用的对象和调用的方法。

  在通过`bl`指令执行`objc_msgSend`时这两个参数分别保存在`x0`和`x1`这两个寄存器中：

  ```assembly
  000000010002ba1c         ldr        x0, #0x100042f20
  000000010002ba24         ldr        x1, #0x100042d40     
  ...
  000000010002ba28         bl         imp___stubs__objc_msgSend
  ```

  在ARC中还会看到这样一条指令：

  ```assembly
  000000010002ba30         bl         imp___stubs__objc_retainAutoreleasedReturnValue
  000000010002ba34         mov        x19, x0
  ```

  将方法的返回值保存在寄存器`x0`中。

- **dq**

  定义数据的指令，定义一个8字节，即64位长度的数据

- **str**

  从源寄存器中将一个32位的数据传送到目标地址中。如`str x0 [x1, #0x8]`将寄存器`x0`的值赋给`x1 + 0x8`处的地址。

- ...

# 动态调试

静态分析只能看到内部的函数执行，而动态调试可以实时获取程序运行时的参数、内存、流程等信息，在逆向分析的时候用的最多的还是动态分析的方式。

## 使用LLDB调试

Xcode在调试手机App的时候，会将一个`debugserver`文件复制到手机中，以便在手机上启动一个服务，允许Xcode进行连接和调试，该文件默认只允许调试自己开发的应用，在越狱之后可以用LLDB调试第三方应用。

### 权限签名

首先需要为`debugserver`文件赋予`task_for_pid`权限，`debugserver`文件的位置在设备上的：

```shell
/Developer/usr/bin/debugserver
```

现将其拷贝到电脑上，然后为其签名权限，先创建一个`entitlement.plist`文件，写入如下内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.springboard.debugapplications</key>
	<true/>
	<key>run-unsigned-code</key>
	<true/>
	<key>get-task-allow</key>
	<true/>
	<key>task_for_pid-allow</key>
	<true/>
</dict>
</plist>
```

然后使用`codesign`进行签名：

```shell
codesign -s - --entitlements entitlement.plist -f debugserver
```

`Developer`文件夹是只读的，所以要将签好名的`debugserver`文件复制到手机的`/usr/bin/`位置中，然后我们就可以使用这个`debugserver`来调试各种应用了。

### 连接目标应用

`debugserver`可以通过**进程名**或**进程ID**连接指定应用进行调试，首先`ps aux | grep /var/containers`找到目标应用的进程：

![WX20180721-142921](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/8d50ef556fd4f4582f9b40c843edfcc5.png)

第一个红框是该进程的ID，第二个红框是进程的名称，通过ID或进程名进行调试，在设备上执行：

```shell
debugserver *:8989 -a osee2unifiedRelease
```

或：

```shell
debugserver *:8989 -a 1238
```

> 其中`*:8989`的*表示任意主机，8989是我们自定义的端口号

如果出现`"error: failed to attach to process named: "" unable to start the exception thread"`之类的错误信息则表示签名的流程有问题，重新签一下名。

如果成功则会显示如下消息：

![WX20180721-144739](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/203cbab8e15933bb855c6b2bb1f2314a.png)

进入等待连接的状态，同时目标应用会卡住，然后在Mac上进行操作，首先通过`lldb`命令进入lldb的交互式命令界面，然后通过`process connect`命令远程连接上手机的调试服务：

![WX20180721-145706](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/247cfd663cc4b2546019ce098918f042.png)

看到上面的界面则表示连接成功了，其中`process connect connect://192.168.2.5:8989`中的IP地址为手机的IP地址，这里使用远程连接，也可以用`iproxy`做端口转发。

### 根据方法地址来设置断点

与正向开发的调试不同的是，我们看不到程序的源码，所以默认情况下我们无法直接对某个方法下断点（后面会介绍符号还原），需要先通过`Hopper`或`class-dump`找到目标方法的地址，然后通过地址来下断点：

```shell
(lldb) br set -a 0x11111
```

成功断点之后可以通过LLDB的指令来检查参数，之前提到在ARM汇编中，参数会通过`x0`~`x7`寄存器传递，可以通过打印寄存器的方式来查看参数的值：

```shell
(lldb) po [$r0 class] // 打印调用对象的类型
(lldb) x/s $r1 // 打印方法名(OC方法第二个参数是方法名)
(lldb) po $r2 // 第一个参数
```

剩下的参数可以通过打印当前调用的函数栈来获取：

```shell
(lldb) x -force -f A $sp $fp
```

### LLDB常用命令

- **c**：让程序继续执行，在交互式界面中`control + c`快捷键可以暂停应用；

- **image**：

  首先`image`命令是`target modules`命令的简称，它可以用来查看模块的信息：

  - **image list** ：

    查看程序中加载的所有模块，比如说:`image list -o -f`

    ![WX20180721-151203](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/fb5156b32947f1378831697b34cd6887.png)

    第一列是模块的**序号**，第二列是模块在内存中的**基地址**，第三列是模块的全路径和真正加载开始的地址

- **br**：

  与断点相关的操作。

  - **br set**：设置一个断点，参数`-a`可以在指定地址上设置断点：`br set -a 0x1234`；
  - **br list**：查看所有的断点；
  - ​

- **po**：

  `print object`的缩写，可以执行一个表达式并打印它的结果；该命令可以直接打印寄存器中的值：`po $x0`，打印`x0`寄存器的值，该寄存器在OC函数表示调用的类，`x1`寄存器表示调用的方法；

- **x**：

  命令`memory read`的缩写，内存读取指令。结合一些操作可以很方便的读取内存中的值，比如：

  ```shell
  (lldb) x/s $x1
  ```

  读取寄存器`x1`的值**并以字符串的形式**打印出来，；

- **register read**：读取所有寄存器的值；

- **si**：跳到当前指令的内部，就是Step Into

- **apropos**：一个帮助命令，用来搜索LLDB文档中相关命令的信息，如：`apropos watch`；


## 使用Xcode调试

平时的开发中我们都习惯使用Xcode，逆向开发同样也可以使用Xcode来调试，不仅在断点上更加直观，还有`View Hierarchy`以及`Memory Graph`这两个调试神器。

上面我们用重签名过的`debugserver`连接LLDB命令行进行调试，但是用Xcode调试的时候还是会使用默认位置的`debugserver`，想要用Xcode直接调试第三方应用的话，分两种情况：

### 越狱手机

对于越狱手机来说比较简单：

1. 将目标应用的可执行文件（不是ipa文件）拷贝到电脑上，文件路径可以通过`ps aux`找到；

2. `ldid`是一个专门用来签名可执行文件的工具（用brew安装），利用`ldid`将应用程序的`code sign`导出来：`ldid -e TargetApp >> TargetApp.xml`；

3. 修改导出的签名文件，加上`get-task-allow`权限：

   ```xml
   <key>get-task-allow</key>
   <true/>
   ```

4. 再用`ldid`对可执行文件重新签名：`ldid -STargetApp.xml ./TargetApp`，注意`-S`与参数之间没有空格；

5. 将重签名之后的可执行文件拷贝回去覆盖原有的文件；

6. 然后运行应用，在Xcode中找到并连接上目标应用的进程：

   ![WX20180722-173410](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/eec2bbc716825809c43d81308a73af7a.png)

### 非越狱手机



### 符号还原

我们可以在`Hopper`的反汇编信息中查看某个函数地址对应的符号，这些符号本身就是包含在二进制文件中的，但是符号表里面没有保存其中的对应关系，所以在Xcode中找不到。有一个名为[restore-symbol](https://github.com/tobefuturer/restore-symbol)的工具可以分析这些信息并将其写入符号表中，这样在Xcode里面就能看到了：

```shell
restore-symbol TargetApp -o TargetApp_symbol
```

得到可执行文件`TargetApp_symbol`，这是包含了符号信息的可执行文件，直接将其拷贝回手机是不能用的，还要进行重签名，可以使用`ldid`或者`codesign`（推荐使用ldid）**重签名**之后拷贝回手机覆盖原有的可执行文件。

结合Hopper和class-dump的分析可以知道目标类型以及方法的名字，之后再通过Xcode调试目标应用时就可以**直接对指定方法**下断点了：

![WX20180722-183051](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/23866045976b133d056506080619a118.png)

这样可以更加方便的进行分析和调试。

### Chisel

`Chisel`是Facebook开源的一系列LLDB调试指令的集合，在Xcode中可以暂停并查看视图结构，可以看到目标视图的地址，结合Chisel可以进行一些非常方便的分析：

- **presponder**：打印某个对象的响应链：`presponder 0x16dd64c0`；

- **pactions**：可以查看按钮的点击事件：

  ![WX20180722-194903](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/cfd1f1eb830014957a05e8ccd3c6486f.png)

  `loginAction:`就是点击按钮的相应方法；

- **pblock**：输入某个参数的地址，如果是一个Block的话，会输出Block的类型和实现地址；

- **pmethods**：输入某个实例对象的地址，打印该对象中的所有方法；

- **pinternals**：输入某个实例对象的地址，打印该对象中所有属性的值；

# 二次开发

上面介绍的方法可以定位某个功能点对应的类型和方法，下一步就是拦截和修改方法的内部实现，这样才能添加自定义的功能。

## Theos

`Theos`是一个越狱开发的工具包。首先需要在Mac上安装三个工具：

- `dpkg`：用于管理deb包；
- `ldid`：用于签名；
- `fakeroot`：用于模拟root权限；


然后开始安装：

1. 安装到`/opt/theos`文件夹：`sudo git clone --recursive https://github.com/theos/theos.git /opt/theos`；
2. 设置权限：`sudo chown -R $(id -u):$(id -g) /opt/theos`；
3. 设置环境变量：`export THEOS=/opt/theos`和`export PATH=$THEOS/bin:$PATH`；

然后就可以通过`nic.pl`命令通过模板来快速创建一个项目了。

## Tweak

`Tweak`是Theos提供的一种Hook第三方应用的一种方式，可以通过`Theos`的模板来快速创建：

1. **创建一个Tweak模板：**

   ![WX20180723-213427](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/9a6084e212cd383a6d9e0acead951d7d.png)

   选择`iphone/tweak`模板，有两个关键的选项：第一个红框是**需要注入的应用的Bundle ID**，第二个红框为安装成功后要杀掉指定的进程（为了让tweak生效）；

2. **然后会自动创建几个文件：**

   ![WX20180723-214048](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/453b17ccc17aba3e1d5299db6ce9b78c.png)

   其中`Tweak.xm`是我们编写代码的地方，它用的是一种名为`Logos`的DSL来编写：

   ```objective-c
   %hook BBPhoneUserLoginViewController // %hook 指定要hook的类型

   - (void)loginAction:(id)button {
   	%log;
   	%orig;
   }

   %end
   ```

   详细的语法可以查看文档：[http://iphonedevwiki.net/index.php/Logos](http://iphonedevwiki.net/index.php/Logos)。

3. **编译**

   使用`make`命令来安装，然后通过`make package`将其打包成deb，之后会在package文件中看到生成的deb文件。

4. **安装**

   `make install`可以直接将其安装到手机上，在这之前需要指定手机的IP和端口号，可以在`Makefile`文件中指定：

   ```makefile
   export THEOS_DEVICE_IP=localhost
   export THEOS_DEVICE_PORT=2222

   include $(THEOS)/makefiles/common.mk
   ```

   这里我们通过`iproxy`转发，所以是localhost。

5. **查看日志**

   设备的日志可以通过`Console.app`应用查看，选择设备并通过进程名来筛选。

# 逆向实践

以腾讯视频为例，实现视频去广告的功能，这里记录一下分析的过程和思路。

## 准备

1. `Clutch`或`dumpdecrypted`砸壳，得到解密后的可执行文件

2. dump所有类型的头文件

   ```sh
   class-dump --arch armv7 TargetApp -A -H -o ../dumped
   ```

3. 符号还原

   ```sh
   restore-symbol TargetApp -o TargetApp_symbol
   ```

4. 设置`get-task-allow`并重签名

   ```sh
   ldid -e TargetApp >> TargetApp.xml
   ldid -STargetApp.xml ./TargetApp
   ```

5. 拷贝回手机

6. 运行目标应用，使用Xcode连接开始调试。如果Xcode无法连接，或是连接LLDB的时候出现`segment fault 11`（通常是ptrace反调试）错误，说明应用程序做了反调试，应对方法可以参考[这篇文章](http://bbs.iosre.com/t/topic/8179)

## 分析

### 找到目标视图

要实现视频去广告的功能首先要找到视频播放器的类型，在Xcode中连接上目标应用，随便打开一个视频，然后查看其视图结构：

![视图结构](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/6ecda9b95a93a3a7540cca272ba53f80.png)

发现广告是由一个单独的播放器控制的，视图的类型是`QNBPlayerVideoAdsView`，找到它的地址然后通过`Chisel`查看响应链：
![响应链](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/19ce0debebb785c6d17ef610a8b12708.png)

现在知道广告视图的逻辑是由另外一个独立的VC：`QNBPlayerVideoAdsViewController `来处理的，从之前Dump出的头文件中找到这个类型可以看到类型里面的所有方法，然后可以先从一些方法的命名上猜测它的作用，直接通过符号设置断点进行调试，逐步验证。

比如说对`showAdsView`方法设置一个断点，并在断点出发的时候直接`thread return`跳过这个方法：

![断点](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/185ef8b0466c49e8f936406bd9ef3f7e.png)

然后发现并不起作用，还需要进一步分析。

### 查看方法调用流程

知道某个类型中方法的调用流程对于分析来说有很大的帮助，这里有很多种实现方法：`Theos`的`logify.pl`、hook `objc_msgSend`方法打印函数名或是其他调用追踪工具等。不过我主要使用Xcode来进行调试，希望流程能够更加简单一些，所以这里我提供一种通过LLDB脚本来实现的方法：

首先LLDB支持通过Python脚本来提供自定义的命令（Chisel正是基于这一点来实现的），只需要在`~/.lldbinit`文件中加上自己的脚本路径就可以了：

```shell
command script import 自定义的python脚本路径
```

然后通过脚本实现一个`traceclass`命令，具体实现在[这里](https://github.com/L-Zephyr/static_resource/blob/master/lldb_commands/lldb_commands.py)。

基本原理就是通过`br set -r`命令使用正则表达式对指定类型中的所有方法下断点，然后通过`br command add  1 -F lldb_commands.traceclass_action`让断点触发的时候执行我们自定义的一个Python函数：

```python
def traceclass_action(frame, location, dict):
  print(frame.GetFunctionName())
  frame.GetThread().GetProcess().Continue()
```

打印断点的方法并继续执行。

现在只需要在LLDB命令行中执行`traceclass QNBPlayerVideoAdsViewController`，控制台中就会打印出这个VC中所有方法的执行流程了：

![调用流程](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/c2f4d7ca35f17502ce8ff07b480aacdc.png)

### 定位逻辑位置

尝试hook了`showAdsView`和`adsStartPlay`无效后开始转换思路，首先来看一下广告视图的创建流程，给`loadView`方法添加一个断点，随后通过`bt`命令打印调用栈：

![调用栈](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/052a3aee308031e70da4fe4a31ac6360.png)

分析调用栈可以看到播放器的VC都是通过一个工厂类型来创建，并保存到父VC中的。查看`Memory Graph`可以看到：

![WX20180807-143853](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/99b76ace8dfbb7a036176f1fb086b374.png)

广告视图的实例被`QNBPlayerUIRootController`对象中的一个数组引用着，在上面的响应链中也可以看到这个VC，查看Header可以看到一个名为`childEventControllers `的属性，打印这个属性可以看到：

![子VC](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/a59fba61072729437321be4161524601.png)

这个根视图中保存了大量的子ViewController，播放页面的业务逻辑被分离到各个不同的子VC里面，继承自`QNBBasePlayerViewController`的子VC都会通过方法`-(void)excuteEvent:(id)arg1 forEventNode:(id)arg2`接受并处理事件，这里的设计可能是一个响应链的模式。

继续往下找发现每次处理广告事件的时候调用栈里都会有这个方法`-[QNBQQPlayerPlugin mediaPlayer:preAdStateChanged:withAdDuration:] `，设个断点查看它第一个参数的类型，然后追踪一下这个对象的调用流：

![trace](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/iOS逆向开发/15e7f749d14f0ed7a3d3971a6fa15dc9.png)

发现每次打开视频的时候都会执行一次`-[TVKMediaPlayerManager prepareAdInfo]`这个方法，编写`tweak`来hook跳过这个方法：

```objective-c
%hook TVKMediaPlayerManager

- (void)prepareAdInfo {
	return;
}

%end
```

再次进入后发现烦人的视频广告终于消失了~

