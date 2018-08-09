现实世界中的网络是由无数的计算机和路由器组成的一张的大网，应用的数据包在发送到服务器之前都要经过层层的路由转发。而Traceroute是一种常规的网络分析工具，用来定位到目标主机之间的所有路由器

# 原理

在介绍Traceroute的原理之前，需要了解几个技术名词：

- **IP协议**

  IP协议是TCP/IP协议族中最核心的部分，它的作用是在两台主机之间传输数据，所有上层协议的数据（HTTP、TCP、UDP等）都会被封装在一个个的IP数据包中被发送到网络上。


- **ICMP**
  ICMP全称为**互联网控制报文协议**，它常用于传递错误信息，ICMP协议是IP层的一部分，它的报文也是通过IP数据包来传输的。

- **TTL**
  TTL（time-to-live）是IP数据包中的一个字段，它指定了数据包最多能经过几次路由器。从我们源主机发出去的数据包在到达目的主机的路上要经过许多个路由器的转发，在发送数据包的时候源主机会设置一个TTL的值，每经过一个路由器TTL就会被减去一，当TTL为0的时候该数据包会被直接丢弃（不再继续转发），并发送一个超时ICMP报文给源主机。

具体到traceroute的实现细节上，有两种不同的方案：

## 基于UDP实现

在基于UDP的实现中，客户端发送的数据包是通过UDP协议来传输的，使用了一个大于30000的端口号，服务器在收到这个数据包的时候会返回一个**端口不可达**的ICMP错误信息，客户端通过判断收到的错误信息是TTL超时还是端口不可达来判断数据包是否到达目标主机，具体的流程如图：

![基于UDP实现的traceroute](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/Traceroute/ae00f77eb31c1a177d1ed174627fb5f6.png)

1. 客户端发送一个TTL为1，端口号大于30000的UDP数据包，到达第一站路由器之后TTL被减去1，返回了一个超时的ICMP数据包，客户端得到第一跳路由器的地址。
2. 客户端发送一个TTL为2的数据包，在第二跳的路由器节点处超时，得到第二跳路由器的地址。
3. 客户端发送一个TTL为3的数据包，数据包成功到达目标主机，返回一个**端口不可达**错误，traceroute结束。

Linux和macOS系统自带了一个`traceroute`指令，可以结合Wireshark抓包来看看它的实现原理。首先对百度的域名进行traceroute：`traceroute www.baidu.com`，每一跳默认发送三个数据包，我们会看到下面这样的输出：

![traceroute](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/Traceroute/ca575007df7f201b4082bc4f7ae8178c.png)

对该域名的IP：`115.239.210.27`进行traceroute，此时Wireshark抓包的结果如下：

![抓包结果](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/Traceroute/8efb53eca9a33228f445535978977c26.png)

注意看红框处的内容，跟第一张图对比，可以看到`traceroute`程序首先通过UDP协议向目标地址115.239.210.27发送了一个**TTL为1**的数据包，然后在第一个路由器中TTL超时，返回一个错误类型为`Time-to-live exceeded`的ICMP数据包，此时我们通过该数据包的源地址可知第一站路由器的地址为`10.242.0.1`。之后只需要不停增加TTL的值就能得到每一跳的地址了。

然而一直跑下去会发现，traceroute并不能到达目的地，当TTL增加到一定大小之后就一直拿不到返回的数据包了：

![结果全是丢失](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/Traceroute/6ab2e617a82b310157ddffc52ab926c1.png)

其实这个时候数据包已经到达目标服务器了，但是因为安全问题大部分的应用服务器都不提供UDP服务（或者被防火墙挡掉），所以我们拿不到服务器的任何返回，程序就理所当然的认为还没有结束，一直尝试增加数据包的TTL。

目前在网上找到许多开源iOS traceroute实现大多都是基于UDP的方案，实际用起来并不能达到想要的效果，所以我们需要采用另一种方案来实现。

## 基于ICMP实现

上述方案失败的原因是由于服务器对于UDP数据包的处理，所以在这一种实现中我们不使用UDP协议，而是直接发送一个**ICMP回显请求（echo request）**数据包，服务器在收到回显请求的时候会向客户端发送一个**ICMP回显应答（echo reply）**数据包，在这之后的流程还是跟第一种方案一样。这样就避免了我们的traceroute数据包被服务器的防火墙策略墙掉。

采用这种方案的实现流程如下：

![基于ICMP实现的traceroute](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/Traceroute/e993ccc5bed5f2f90ced00d9bdf11b62.png)

1. 客户端发送一个TTL为1的**ICMP请求回显**数据包，在第一跳的时候超时并返回一个ICMP超时数据包，得到第一跳的地址。
2. 客户端发送一个TTL为2的ICMP请求回显数据包，得到第二跳的地址。
3. 客户端发送一个TTL为3的ICMP请求回显数据包，到达目标主机，目标主机返回一个**ICMP回显应答**，traceroute结束。

可以看出与第一种实现相比，区别主要在发送的数据包类型以及对于结束的判断上，大体的流程还是一致的。

值得一提的是在Windows系统中也有traceroute程序，它的名字叫做`tracert`，`tracert`就是用采用这种方法来实现的，感兴趣的话可以自行尝试一下，这里就不再演示了。

# 实现

这里我们主要讨论基于ICMP的实现，相关的Demo已经上传至github：[https://github.com/L-Zephyr/TracerouteDemo.git](https://github.com/L-Zephyr/TracerouteDemo.git)

采用这种方案时，ICMP数据包的创建、解析、校验都需要我们自己进行，ICMP是封装在IP数据包的数据段中传输的，所以关键在于如何创建和发送ICMP数据，以及接收到返回的数据时如何从IP数据包中将ICMP解析出来：

## 创建ICMP数据

ICMP数据包头部的格式如下：

![ICMP数据结构](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/Traceroute/8065c9f9fd3110dc90fb5f97a159bff5.png)

其中的`类型`字段用来表示消息的类型，在[Wiki](https://zh.wikipedia.org/wiki/%E4%BA%92%E8%81%94%E7%BD%91%E6%8E%A7%E5%88%B6%E6%B6%88%E6%81%AF%E5%8D%8F%E8%AE%AE)上可以看到所有类型代表的含义。报文中的标识符和序列号由发送端指定，如果这个ICMP报文是一个请求回显的报文（类型为8，代码为0），这两个字段会被原封不动的返回。

根据上图中各个字段的大小可以定义如下类型：

```objective-c
typedef struct ICMPPacket {
    uint8_t     type; // 类型
    uint8_t     code; // 类型代码
    uint16_t    checksum; // 校验码
    uint16_t    identifier; // ID
    uint16_t    sequenceNumber; // 序列号
    // data...
} ICMPPacket;
```

其中的`type`字段指定了这个ICMP数据包的类型，是需要重点关注的对象，为此定义一个报文类型的枚举：

```objective-c
// ICMPv4报文类型
typedef enum ICMPv4Type {
    kICMPv4TypeEchoReply = 0, // 回显应答
    kICMPv4TypeEchoRequest = 8, // 回显请求
    kICMPv4TypeTimeOut = 11, // 超时
}ICMPv4Type;
```

比较麻烦的是校验的计算，这一部分直接使用了苹果官方示例[SimplePing](https://developer.apple.com/library/content/samplecode/SimplePing/Introduction/Intro.html)中的代码，所涉及到的几个工具方法封装在类型`TracerouteCommon`中。

在发送数据的时系统会自动加上IP头部不需要自己处理，如此一来我们只需要创建一个`ICMPPacket`数据包并通过socket发送到目标服务器就可以了。

## 解析ICMP数据

接下来就是要接收服务器向我们返回的ICMP数据了，我们接收到的是带有IP头部的原始数据，所以必须先进行一些处理将ICMP从IP数据包中提取出来，IP数据包由两部分组成：数据包头部信息部分以及实际的数据部分。下图是IPv4数据包的结构：

![ipv4](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/Traceroute/c88cf39691309273e3021cc9c23967a0.png)

一眼看上去是不是感觉很混乱，其实这里面只有用红框圈出来的这这三个字段需要我们关心：`版本`表示该数据包是IPv4还是IPv6；之前说过ICMP协议是通过IP协议来传输的，如果该数据包传输的是ICMP协议则`协议`字段会被设置为`1`；由于IPv4数据包带有可选的选项字段，所以其头部的长度是可变的，此时需要根据`首部长度`字段来获取具体的数据。

根据上面的结构可以定义类型：

```objective-c
typedef struct IPv4Header {
    uint8_t versionAndHeaderLength; // 版本和首部长度
    uint8_t serviceType;
    uint16_t totalLength; 
    uint16_t identifier;
    uint16_t flagsAndFragmentOffset;
    uint8_t timeToLive;
    uint8_t protocol; // 协议类型，1表示ICMP
    uint16_t checksum;
    uint8_t sourceAddress[4];
    uint8_t destAddress[4];
    // options...
    // data...
} IPv4Header;
```

提取ICMP数据包的方法如下：

```objective-c
+ (ICMPPacket *)unpackICMPv4Packet:(char *)packet len:(int)len {
    if (len < (sizeof(IPv4Header) + sizeof(ICMPPacket))) {
        return NULL;
    }
    
    const struct IPv4Header *ipPtr = (const IPv4Header *)packet;
    if ((ipPtr->versionAndHeaderLength & 0xF0) != 0x40 || // IPv4
        ipPtr->protocol != 1) { // ICMP
        return NULL;
    }
    
    // 获取IP头部长度
    size_t ipHeaderLength = (ipPtr->versionAndHeaderLength & 0x0F) * sizeof(uint32_t); 
    if (len < ipHeaderLength + sizeof(ICMPPacket)) {
        return NULL;
    }
    
    // 返回数据部分的ICMP
    return (ICMPPacket *)((char *)packet + ipHeaderLength);
}
```

其中出现的如`ipPtr->versionAndHeaderLength & 0xF0`的判断是因为版本号和首部长度各自只占4个bit，在结构中直接定义了一个1字节的`uint8_t`类型来表示，所以只能通过位运算符`&`来获取各自的值。

## 整体流程

有了上面的两步，剩下的事情就很简单了，下面是整体流程的伪代码：

```objective-c
// 1. 创建一个套接字
int sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_ICMP);

// 2. 最多尝试30跳
int ttl = 1;
for (0...30) {
    // 3. 设置TTL，发送3个ICMP数据包，每一跳都将递增TTL
    setsockopt(sock, IPPROTO_IP, IP_TTL, &ttl, sizeof(ttl));
    ++ttl;
	for (0...3) {
		// 4. 发送并等待返回的数据包
        sendto(...);
        recvfrom(...);
        
        // 5. 解析数据包，记录数据，成功条件判断
        ICMPPacket *packet = unpack(...);
    }
}
```

`socket`的类型采用了`SOCK_DGRAM`，有些小伙伴可能会感到疑惑：用SOCK_DGRAM创建套接字不还是发送UDP数据么？

确实在许多系统的实现中要直接发送ICMP的话需要使用原始套接字（类型为`SOCK_RAW`），这在iOS系统中是不被允许使用的，但是查阅资料中了解到macOS支持一种使用参数`SOCK_DGRAM`和`IPPROTO_ICMP`来直接创建ICMP套接字方式，尝试之下果然iOS也支持这种用法。不过在使用中发现了一个问题：使用IPv4套接字的时候接收到的数据包是带有原始IP头部的，而使用IPv6套接字的时候收到的数据包却没有IP头部，这个问题让我比较疑惑，各位大佬如果有对这一块了解的话还望赐教。

# 总结

[Demo](https://github.com/L-Zephyr/TracerouteDemo.git)中的示例程序已经在模拟器和真机环境经过测试，可以看到，现在Traceroute已经能够正常使用了：

![Traceroute](https://raw.githubusercontent.com/L-Zephyr/static_resource/master/Resources/Traceroute/1aebeceb0527bab03dc9270289622ae3.png)

有些路由器会隐藏的自己的位置，不让ICMP Timeout的消息通过，结果就是在那一跳上始终会显示星号，此外服务器可以伪造traceroute路径的，不过一般应用服务器也没有理由这么做，所以Traceroute的结果还是能够为网络分析提供一些参考的。