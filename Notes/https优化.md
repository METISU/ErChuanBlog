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

## Cache-Control

这个不用过多描述了，只要注意的是一些界面不能使用缓存，比如秒杀界面

对于一些比较大的数据量要检查是否要更新缓存时，可以使用 **if-Modified-Since**、**If-None-Match** 字段，如果返回 **304 Not Modified**，则代表不需要更新缓存，节省流量以及时间

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
2. ECDHE最后可以不用等待service回复，抢先发送消息
3. service key exchange以及client key exchange过程

### tls1.3
