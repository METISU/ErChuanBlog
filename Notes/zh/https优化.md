# HTTPS优化

近期处理了一些网络库的问题，发现有些储备的知识已经遗忘，有思路但是具体实现还得翻一些博客，这篇文章主要给自己做个备忘录功能，整理一下以前探究过的HTTPS相关的优化，如有更好的方案，可以一起探讨🥺

## DNS缓存

发起一个hppt请求，第一步要做的就是DNS解析（应用层方面），这时候计算机最常用的优化手段之一：缓存就派上用场了。

思路不难理解，重要的是使用过程中不出问题，几个关键事项：

* 缓存的有效期
* 缓存是否失效
* 缓存失效或者没有缓存的降级方案

任何东西最注重的是用户体验，我们添加缓存的目的是为了提升用户体验（响应速度更快），所以如果缓存出现问题导致用户访问不了，那比网络慢的体验还差，所以降级方案必须先考虑周全

大体流程上：

![DNS缓存](https://user-images.githubusercontent.com/22512175/148319012-80643741-fbc9-433b-a503-069e657e8cf8.png)
  
主要的关键点在于保证降级方案的正常实行，比如缓存过期了、失效等，具体的问题需要具体分析

## 多路复用

HTTP有个大名鼎鼎的问题：队头阻塞，解决方法也很简单，既然我的通道堵了，那多开几条就行了--并发连接

但是一般HTTP协议、浏览器等会限制连接数量，这个时候可以使用**域名分片**，说白了就是你限制我一个域名最多连接6个通道，那我就多开几个域名指向类似的服务器，那就达到了6 * x的效果

## Cache-Control

这个不用过多描述了，只要注意的是一些界面不能使用缓存，比如秒杀界面

对于一些比较大的数据量要检查是否要更新缓存时，可以使用 **if-Modified-Since**、**If-None-Match** 字段，如果返回 **304 Not Modified**，则代表不需要更新缓存，节省流量以及时间

## HTTP/2

再谈HTTP/2之前，首先看看HTTP/1.x的劣势，清楚了哪里不足，才可以知道需要往哪个方向进行优化，不是吗

1. 「大头儿子」现象：http请求的header一般较为固定，但body有大有小，甚至大部分的body都偏小，并且header重复，这是一块极大的浪费
2. 队头阻塞

接下来我们看看HTTP/2是怎么解决这些问题的

### 请求头过大

思路其实不难想到，一般重复发送的内容都是必要的内容，做不到不发，那么换个角度思考，减小发送的大小，方案就有了，「压缩」

压缩的方案是使用「HPACK」算法，实质上是client与service共同维护的一张表，在发送header的时候就使用key:value的形式，value可以是一个数字，比如2代表GET，同时为了与别的没被压缩的header进行区分，会在前面添加':'，比如:method等

### 多路复用

HTTP2将报文拆解成二进制格式，并且将原来的Header+Body的消息分散为Frame的形式，然后将同一个报文的消息赋予同一个Stream id，这样就可以做到在一个连接上发送多个stream，同时不同的stream之间的frame却是乱序的（在同一个stream中，frame是有严格的先后顺序的），意味着不需要等待，也就是说多个stream可以并行发送，也就是多路复用。

之后根据相应的stream id组成对应的报文。这个有个很大的优点是可以将带宽跑满，因为服务端与客户端的流互不干扰，所以一个tcp连接的传输不会浪费

以上说了HTTP/2的大概优点，还有一个很重要的点，HTTP/2也完美的向下兼容，所以不必担心升级HTTP/2对老的用户又什么影响

## HTTP/3

其实HTTP/2的优化已经算做到了极致（至少在应用层上可以这么说），但是HTTP是一个应用层的协议，会受到传输层的限制，这就不得不提TCP的队头阻塞了，而这个又是在传输层，所以HTTP协议无论怎么优化，都无法解决这个问题。

那么为了解决这个问题，我们只有在传输层动手脚，但又不可能更改传输层的协议（传输层的协议如果需要更改代价过于高昂），那么就在现有的传输层协议里面选取一种，相信大部分都会首先想到与TCP齐名的UDP，但是UDP快是快了，然而不可靠啊，那咋办咧，Google就想到了在这基础上在应用层做了一层封装，于是 **QUIC**协议诞生了

说的直白点就是TCP协议比较重，而UDP其实只是对于ip层的简单封装，那取UDP之后对于想要的特定自己进行封装，打造一个类似“定制”的协议（严格地说也不能算定制，因为QUIC不仅仅可用于HTTP）。

首先肯定解决了队头阻塞问题，因为UDP压根没有这个概念，其次UDP是“无连接”的，也就是说在传输层方面就快了许多，然后在这基础上实现可靠传输，并引入类似于HTTP/2 的“steam”和“多路复用”，保证单个stram有序但不同的stram不会相互阻塞，然后QUIC使用ID的概念来标记连接，带来的好处是比如切换网络，由于ip以及port会发生改变，TCP由于是根据ip+port连接的，那就需要重连，但是QUIC就不需要了，因为id没变，还是这个连接

## SSL/TLS

TLS是什么就不过多述了，下面主要讲些TLS上面的优化

首先过一下传统的RSA算法握手过程：

1. client to service: 随机数、tls版本、密码套件等必要信息
2. service to client: 随机数、使用的密码套件、证书
3. client顺着证书链验证证书有效，client to service: 公钥加密**Pre-Master**（第三个随机数），发送给服务端，同时client已经有3个随机数，可以生成对称加密秘钥
4. service to client: 拥有3个随机数，确认秘钥，之后使用秘钥加密传输

可见一个完成的TLS连接需要耗费2 RTT，可以看看建立一个HTTPS连接各个流程所花费的时间

``` shell
curl -o /dev/null -s -w "time_namelookup: %{time_namelookup}\ntime_connect: %{time_connect}\ntime_appconnect: %{time_appconnect}\ntime_redirect: %{time_redirect}\ntime_pretransfer: %{time_pretransfer}\ntime_starttransfer: %{time_starttransfer}\ntime_total: %{time_total}\n" "https://github.com"

time_namelookup: 0.001451
time_connect: 0.172504
time_appconnect: 0.540843
time_redirect: 0.000000
time_pretransfer: 0.540928
time_starttransfer: 0.724706
time_total: 0.928256
```

可见tls耗时0.368339，占据整体时间的39.6%，这是一个很大的占比了，也就是说在tls优化部分也能带来很大的改善

首先加密算法方面，可以使用较新的ECDHE算法，流程

1. client->service: 随机数A、密码套件等
2. service->client: 随机数B、确认密码套件、证书，以及**Server Key Exchange**（选择ECDHE算法，确定椭圆曲线，生成椭圆曲线的私钥自己使用，公钥公开）
3. client->service: 生成椭圆曲线私钥、公钥，同样的将公钥发送给service，即**Client Key Exchange**，至此，自己的椭圆曲线私钥、公钥，云端的椭圆曲线公钥，通过ECDHE算法计算出**Pre-Master**（具体流程感兴趣的同学可以了解下，还是比较复杂的十分严谨的数学逻辑），至此，客户端已经有了随机数A、B、Pre-Master，生成**Master Secret**（对称加密秘钥），然后**提前发出加密的 HTTP 数据**
4. service->client: 同样的，拥有自己的椭圆曲线私钥、公钥，收到客户端椭圆曲线公钥，生成**Pre-Master**，然后根据随机数A、B、Pre-Master生成**Master Secret**

可见主要区别在于
1. **Pre-Master**不用传输，即可保证**前向保密**
2. ECDHE最后可以不用等待service回复，抢先发送消息（即TLS False Start）
3. service key exchange以及client key exchange过程

### TLS1.3

TLS1.3首先完全做到了向下兼容，也就是说即便云端不支持TLS1.3，那么客户端也不会出现问题，之后TLS1.3删除了许多不安全的算法，这样带来的最大的优势是不必像TLS1.2一样进行复杂的协商步骤（协商秘钥交换算法）

具体原理是client在client hello阶段使用 **supported_groups** 扩展添加支持的算法，用 **key_share** 扩展带上公钥，这样第一个消息发送成功之后，服务端就可以计算出**Pre-Master**，从而得到**Master Secret**

流程：

1. client->service: 随机数A、支持的TLS版本、supported_groups、key_share
2. service->client: service此时可以选择加密算法，然后生成自己的随机数B、私钥、公钥，此时已经有了自己的私钥、公钥、client的公钥，可以生成**Pre-Master**，加上随机数A、B，就可以生成**Master Secret**，然后同样通过**supported_versions**告知客户端使用的是TLS1.3，同时附带自己的公钥、随机数B、证书，然后发送加密的消息摘要
3. client->service: 此时客户端已经有了自己的私钥、公钥、service的公钥，同样生成**Pre-Master**，然后加上随机数A、B生成**Master Secret**，发送确认消息，开启对称加密

从上可以看到，ECDHE加密方式由于**Pre-Master**不需要加密传输所带来的便利

## 会话复用

从上可以看到，商量出一个**Master Secret**多么复杂，所花费的代价多么大，因此，是不是可以使用下缓存的思想，把**Master Secret**缓存下来，这就是会话复用的核心思想，可以使用**Session ID**以及**Session Ticket**
