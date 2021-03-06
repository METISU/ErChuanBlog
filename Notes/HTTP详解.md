# HTTP详解

> 最近阅读了《透视HTTP协议》，同时系统性梳理了一遍自己对于HTTP协议的问题栈，希望能从头到尾把http的知识梳理清楚，本文会从http/1.1到http/2，如有不正确的地方，希望可以指出 :wink:

## HTTP/1.1
### 概念解析
#### 概述
HTTP是一个运行于TCP/IP协议栈上的超文本运输协议，处于应用层。

#### 特点
* 可靠传输

因为HTTP基于TCP/IP协议栈，利用TCP协议收发数据，所以这个特点显而易见，因为TCP就是一种可靠、有序的传输方式。
* 灵活可扩展

HTTP协议只制定了部分规范，比如header与body之前用一行空行分割、必须要有header、header必要要有请求方法等，但是对于报文中的各个部分没有严格的规定，各个开发者可以与自己的云端进行商讨，指定属于自己的通信规则。
* 请求-应答通信模式

即需要请求方先发起请求，服务端收到请求后返回数据，或直接返回客户端想要的数据并返回200OK，或者提示重定向3XX，或者提示客户端请问问题4XX（比如很常见的404 Not Found，但这个错误码目前存在大量滥用现象），或者服务器内部错误5XX

* 无状态

形象地说就是HTTP协议没有记忆能力，如果下一个请求依赖前面请求的结果，就会非常致命，比如用户登录成功之后进行一系列操作，用户传输账号密码之后获取响应信息，但是下次请求还需要重新传输账号密码，否则服务端不知道请求的是谁。

### URI(URL)
#### 概述
* URI：为**统一资源标识符**（Uniform Resource Identifier）
* URL：为**统一资源定位符**（Uniform Resource Locator）
URI是URL的超集，HTTP里面的网址实质上是URL。

#### 格式
URL的基本格式是

![系统架构设计](https://user-images.githubusercontent.com/22512175/113471501-4a236580-948f-11eb-8c08-6587a0855a67.png)

* scheme：第一个部分为scheme，表示协议名，最常见的有http、https，另外还有比如ftp等

scheme之后必须跟随 ***特定字符://*** ，标志scheme结束

* host:port：之后是host+port组合，表示自愿所在的主机名。其中主机名必须要有，否则会找不到服务器，但是port可以省略，浏览器会根据scheme使用默认的端口，比如HTTP的80端口、HTTPS的443端口。

* path：协议://主机名+端口之后，跟随的是path，即需要查找的资源的路径，如果在根目录可以省略。

* query：协议名+主机+路径已经可以找到所有资源了，但是如果我们对于资源有一些要求，那这还不够。
比如我们希望查询到的资源是按照ASCII排序、或者我们需要某种特定格式的资源，这样我们就得带上query查询参数。query从“?”开始，之后以***key=value***形式传递，比如Google的搜索https://www.google.com/search?q=http ，用于搜索包含HTTP的资源。

以上为url的基本格式，也是目前大多数场景下应用的，但是url的完整形态还缺少user、password以及fragment

![系统架构设计 (1)](https://user-images.githubusercontent.com/22512175/113471566-d2096f80-948f-11eb-8451-f04163ee0cad.png)

* 其余：第一个多出的部分是用户名以及密码，但是因为把敏感信息用明文的形式暴露出来，这种方式基本被舍弃了。后续还增加了***fragment***，是定位内部资源的一个锚点，浏览器获取资源后可以跳转到想要的位置。
比如
[https://github.com/METISU/erChuanBlog/blob/main/Notes/HTTP详解.md#格式](https://github.com/METISU/erChuanBlog/blob/main/Notes/HTTP%E8%AF%A6%E8%A7%A3.md#%E6%A0%BC%E5%BC%8F)
这个url点击后浏览器会直接定位到**格式**这一栏。

#### 编码
我们通常把url复制出来之后会看到里面有一堆乱码，比如上面的例子，**详解**变为了**乱码：%E8%AF%A6%E8%A7%A3.md#%E6%A0%BC%E5%BC%8F**，这是因为url里面只能使用ASCII码，对于非ASCII码文字或者特殊字符采用转换成16进制值然后在前面加%。但是在Chrome等浏览器里面为了方便用户，地址栏是看不到转以后的乱码的。

### 报文
#### 报文结构

![系统架构设计 (4)](https://user-images.githubusercontent.com/22512175/113509970-e67e6280-958a-11eb-9250-34f81ea42521.png)

报文大致由3个部分组成
1. 起始行：描述请求或响应的基本信息
2. 头部字段集合：一般为key-value集合详细说明报文
3. 消息体：请求传输的数据结构

> 注：header与body之前需要有一个空行分割，同时，可以没有body，但必须有header

![WeChatWorkScreenshot_9c3d231a-b657-4273-8234-2f4084536a08 copy](https://user-images.githubusercontent.com/22512175/113504147-c9389c80-9568-11eb-8e3c-0f39cea2ad94.jpg)

##### 起始行
###### 请求行

![系统架构设计 (3)](https://user-images.githubusercontent.com/22512175/113499919-7cdf6380-954c-11eb-8468-3c92b6f61107.png)

请求行有三个部分组成
* 请求方法：如GAE、POST等
* 请求目标：要请求的目标资源
* 版本号：标记使用的HTTP版本

如上面Wireshark的抓包
```
GET /20-2 HTTP/1.0
```
表示GET请求，获取/20-2下的资源，用HTTP1.1

###### 状态行
响应报文的起始行叫做状态行，同样也由三部分构成
![系统架构设计 (6)](https://user-images.githubusercontent.com/22512175/113510117-c4391480-958b-11eb-8ee8-43eb3b951ca0.png)
* 版本号：表明使用的HTTP版本号
* 状态码：表示处理的结果，比较常见的就是我们最希望看到的200，表示成功
* 原因：用来解释状态码，比如常见的Not Found

<img width="917" alt="WeChatWorkScreenshot_122d5033-86fb-4339-b546-3214b070d675" src="https://user-images.githubusercontent.com/22512175/113510235-78d33600-958c-11eb-9fe5-5f47e7c0d06b.png">
这边的状态行就是

```
HTTP/1.1 304 Not Modified
```
意识是说我是用的版本是HTTP/1.1，状态码是304，缓存还可使用。

#### 头部字段集合
头部字段集合采用key: value的形式，形式非常自由，可以任意添加，但要注意以下几点：
1. 字段名不区分大小写
2. 字段名不能使用空格以及下划线"_"
3. 字段名后面必须接":"，但是value前可以存在空格

#### 消息体
消息体的格式较多，一般较为常用的有以下几种
* json格式传输，如{"input1":"xxx","input2":"ooo","input3":"aaa"}
* key=value形式，多个字段使用&连接,如input1=xxx&input2=ooo&input3=aaa
* 分块传输，响应头需要添加字段Transfer-Encoding: chunked，格式为当前块的长度+换行`\r\n`+当前块内容+`\r\n`，终止块和常规块一样，但长度取为0

> HTTP/2不支持分块传输，因为其提供了更有效的流传输机制。

```
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

7\r\n
Mozilla\r\n
9\r\n
Developer\r\n
7\r\n
Network\r\n
0\r\n
\r\n
```
* 范围请求（Partial Content）

返回状态码为206 Partial Content
如果包含多端数据则`Content-Type`设置为`multipart/byteranges`，并且每个片段是设置一个范围，使用`Content-Range`和`Content-Type`进行描述
```
HTTP/1.1 206 Partial Content
Date: Wed, 15 Nov 2015 06:25:24 GMT
Last-Modified: Wed, 15 Nov 2015 04:58:08 GMT
Content-Length: 1741
Content-Type: multipart/byteranges; boundary=String_separator

--String_separator
Content-Type: application/pdf
Content-Range: bytes 234-639/8000

...the first range...
--String_separator
Content-Type: application/pdf
Content-Range: bytes 4590-7999/8000

...the second range
--String_separator--

```

### Cookie
前面提到过HTTP是无状态的，那么遇到一些复杂的场景，比如用户鉴权之类的，就需要Cookie登场了。
#### Cookie使用
需要服务端发送`Set-Cookie`字段，客户端便会把响应的字段保存本地，下次再次请求相同服务，可以通过`Cookie`字段发送
##### Set-Cookie
###### Attributes
* <cookie-name>=<cookie-value>

Cookie以键值对开头
* Expires=<date> (optional)

Cookie的最长生存时间
* Max-Age=<number> (optional)

Cookie过期之前的秒数。零或负数将立即使cookie失效。如果同时设置`Expires`和`Max-Age`，则`Max-Age`具有优先级。
* Domain=<domain-value> (Optional)
  - 如果省略，则默认当前域名，**且不保函子域**（参考MDN文档）  
  - 与早期规范相反，`.example.com`域名中的前倒点被忽略
  - 多个`host/domain`不符合规范，但是如果该`domain`被包含，则其子域也被包含

表示Cookie可以发送到的主机。
* Path=<path-value> (Optional)

路径必须包含在请求的URL中，否则浏览器不会发送Cookie头。以`\`作为分隔符，同时会包含子目录，如`Path=/docs`，则`/docs`、`/docs/Web/`以及`/docs/Web/HTTP`等都会被匹配。
* Secure (Optional)

Cookie仅会发送给使用HTTPS的服务，用于防护中间人攻击。
* HttpOnly (Optional)

禁止`JavaScript`使用如`Document.cookie`属性访问Cookie。
* SameSite=<samesite-value> (Optional)

控制跨域请求是否发送Cookie。

### 缓存
#### Cache-Control
用于指定浏览器请求以及服务器响应中指定的缓存策略。
##### 属性
* public

响应可以被任意对象存储（比如CDN）
* private

响应仅能被浏览器存储
* no-cache

客户端使用前必须现象服务端验证缓存是否过期。
* no-store

不允许缓存
* max-age=<seconds>

缓存保险时间。（注：这个时间不是客户端收到响应的时间，是服务器发出报文的时间。）
* s-maxage=<seconds>

覆盖`max-age`或者`Expires header`，仅用于缓存代理。
* max-stale=<seconds>

表示客户端接受有效期超出指定秒数的响应（常用与代理缓存的响应。）
* min-fresh=<seconds>

表示客户端希望响应至少在指定的秒数内仍旧有效。
* stale-while-revalidate=<seconds>

表示客户端可以接受效期超出指定秒数的响应，同时在后台查询是否有新的响应。
* stale-if-error=<seconds>

表示如果查询新的响应失败客户端可以接受超出指定秒数的响应。
* must-revalidate

表示缓存一旦过期，则需要去服务端验证才可使用。
* proxy-revalidate

类似于`must-revalidate`，但用于缓存代理。
* immutable

表示响应主体不会随时间变化。因此，客户端不应该发送`if-none-match`或者`If-Modified-Since`来检查是否更新。
* no-transform

不允许中间缓存编辑响应。
* only-if-cached

表示只接受中间缓存，如果没有缓存，则返回504。

#### If-Modified-Since
只有在服务器在传输的时间之后修改过资源，才会返回响应体以及200 OK，否则返回304并且没有响应体。（仅能用于GET或者HEAD请求。）

#### Last-Modified
包含原始服务器的资源上次修改的日期和时间。用于验证器接收与存储的资源是否相同。

#### If-None-Match
如果服务器没有`ETag`与发送的相匹配，则返回报文与200 OK，否则返回304 Not Modified。（注：`ETag`有强弱之分，`ETag`前面加`W/`表示使用弱比较算法，即如果两个文件内容相同则表示相同，不同字节层面的相同，比如仅有文件生成日期不同也表示两个文件相同。）

#### ETag
表示自愿特定版本的标识符，如果内容未发生更改，则web服务器不需要发送完整的响应。如果指定的URL资源发生改动，则需要生成新的`ETag`。

### 性能优化
#### 资源内联（inlining）
资源内联指的是将一个资源作为另一个资源的一部分，使二者作为一个整体的资源来请求，比如`HTML`文件里嵌入`base64`的图片，这样做的优点是可以减少HTTP请求数、提升页面加载速度

#### 域名分片（Sharding）
一般浏览器会限制同一个域名HTTP并发连接数，所以可以多个域名指向同一个服务，这样可以多开几个连接。

#### 精灵图（Spriting）
客户端上面可能需要很多小图片，那么当需要用到的时候，需要向服务端请求很多次，容易造成大量时间的消耗。所以可以把多张小图片合成一张大图片，下载下来需要展示哪张图片就用对应坐标以及长款去截取。

***

## HTTPS
前文可以知道，HTTP是明文传输，这种状态等于在不安全的网络环境里面进行裸奔，如果被有心之人窃取到一些重要信息那就比较麻烦，因此需要引入**HTTPS (HyperText Transfer Protocol Secure)**。
### SSL/TLS

HTTPS的关键在于S，前文我们可以知道HTTP运行于TCP协议上，那HTTPS则是运行于SSL/TLS的HTTP协议。
![系统架构设计 (1)](https://user-images.githubusercontent.com/22512175/113558539-69a6c380-9632-11eb-9efd-e1e53965f415.png)
TSL(Transport Layer Security)（以前称为SSL(Secure Sockets Layer)）是应用程序在网上用来安全通讯的协议，通过使用加密协议在网络上提高安全性来保证通讯的私密性。

### HTTPS握手
#### 对称加密
实现安全对话最常见的办法是加密，只有掌握秘钥的人才能对数据进行解密，拿到原文。堆成加密就是加密和解密都用的是同一个秘钥的加密手段。目前常用的有AES以及ChaCha20.

![系统架构设计 (2)](https://user-images.githubusercontent.com/22512175/113566944-3c611200-9640-11eb-94bc-9223f184b2ad.png)

但是对称加密有一个致命的问题，即**秘钥交换**问题，如果黑客截断了秘钥，那么即便密文，在黑客眼里就是明文了。

#### 非对称加密
上述提到了对称加密秘钥传输的问题，有一个解决办法就是非对称加密。

对称加密只有一个秘钥，加密解密用的同一个。非对称加密不同，拥有两个秘钥，分为`公钥`以及`私钥`。通过公钥加密的数据只能私钥解密，反之，通过`私钥`加密的数据只能`公钥`解密。顾名思义公钥是公开的，任何人都能拿到对应的公钥，而私钥是保密的，只有持有者知道。
![系统架构设计 (3)](https://user-images.githubusercontent.com/22512175/113567283-ee004300-9640-11eb-9c15-12f2429d95ee.png)

非对称加密比较常用的有`RSA`，基于数学难题：将两个大素数的乘积因式分解的难题；`ECC`：基于椭圆曲线离散对称的难题，都拥有严谨的数据背景。

但是非对称加密即便解决了安全问题，但又引入了一个新的问题，**效率问题**。非对称加密相对于对称加密在效率上差了许多，用户体验会不好。

#### 混合加密
从上文可知，对称加密效率较高，但不安全，秘钥交换容易被截获；非对称加密安全性比较高，私钥独自保管，但效率上比较低。所以，可以结合一下两种加密方式，使用非对称加密加密秘钥，服务端拿到秘钥之后，进行私钥解密，获取秘钥，这样黑客即便截获了秘钥，也无法解密。

![系统架构设计 (4)](https://user-images.githubusercontent.com/22512175/113569751-93b5b100-9645-11eb-8d78-7dff7aa90cc3.png)

但是这样还有个问题，遇到中间人攻击，伪造身份，假装是服务端，给你他的公钥，客户端拿到公钥无法确认是谁的公钥，这样客户端就会和黑客建立连接，进行通讯，因为服务器的公钥是开放的，甚至于黑客解密后可以用服务端的公钥进行加密，然后传给服务端，和服务端通信。

#### 数字签名
为了解决上述问题，需要使用数字签名概念。

首先使用摘要算法（即散列函数）对内容进行加密，因为哈希函数的一个特性是无法逆向解密，并且修改一个字段就会隐私“雪崩效应”，我们在原文（包含服务端身份信息）后附带摘要，可以保证数据完成性并且不被篡改。这样一套流程就行成了。

![系统架构设计 (5)](https://user-images.githubusercontent.com/22512175/113585470-d89a1180-965e-11eb-9976-290d35d09474.png)
> 服务端生成秘钥

![系统架构设计 (6)](https://user-images.githubusercontent.com/22512175/113585893-6970ed00-965f-11eb-9c29-e31a9b176793.png)
> 客户端验证

自此，完整的流程完成。但是又遇到一个问题，**如果中间人伪造公钥，客户端无法确认这个是否是服务端的公钥，那就还是白搭**，这就陷入了一个鸡生蛋、蛋生鸡的问题，为了打破这个循环，我们必须得引入一个第三方公证机构，由其确保你拿到的公钥是你服务端的公钥。这个第三方机构就是**CA**，CA有自己的信任链，你必须信任**Root CA**，否则走不下去。

### 握手
到此可以讲解HTTPS的握手阶段了。

![系统架构设计 (7)](https://user-images.githubusercontent.com/22512175/113590596-709af980-9665-11eb-84fe-01c78157cc90.png)

1. Client Hello
  - 附带支持的TSL协议版本
  - 附带客户端支持的密码套件
  - 附带客户端随机数A（用于生成对称密码使用）
  - 附带支持的压缩方法
2. Server Hello
  - 确认使用的协议版本
  - 确认使用的密码套件、
  - 服务端随机数B（用于生成对称密码使用）
  - 服务端证书
3. 客户端逐级验证证书是否有效
4. 客户端回应
  - 附带第三个随机数`pre-master key`，并用服务端的公钥加密，发送给服务端。
  - 编码方式改变通知，之后都用三个随机数（随机数A、随机数B、pre-master key）生成的秘钥进行加密通讯。
  - 通知握手结束通知，附带前面内容的hash值，供服务端校验。
5.服务端最后回应
  - 编码方式改变通知，通知之后使用生成的秘钥加密。
  - 通知握手结束，发送前面内容hash值，供客户端校验。
6. 开始对称加密通信。
> 为啥需要三个随机数：SSL协议不信任每个主机都能产生完全随机的随机数，因此引入三个随机数，三个随机数生成的秘钥随机度就非常高了。如果使用`ECDHE`秘钥交换算法，保密度更高，并且支持**前向安全**。

### TLS1.3
从上文可知，`HTTPS`握手会消耗大量的时间，我们可以对其进行优化
TLS1.3主要升级项：
* 添加了0-RTT模式。
* 静态RSA和Diffie-Helman密码套件已被删除。
* ServerHello之后的所有握手消息都已加密。
* 新增了一些新的密码套件（Curve 25519 和 448）。
* 删除了MD5、SHA1

#### 性能提升
##### 1-RTT
由于`TLS1.3`大幅简化了密码套件，客户端可以再第一个`Client Hello`里面带上以及支持的密码套件以及公钥、对应的签名算法，这样只需一个RTT，就完成了秘钥交换。

![系统架构设计 (8)](https://user-images.githubusercontent.com/22512175/113799471-b819a680-9787-11eb-81df-51c002625e37.png)

1. 客户端Hello
  - 客户端随机数
  - supported_versions附带TLS1.3
  - signature_algorithms附带签名算法
  - supported_groups附带`DHE`或者`ECDHE`秘钥交换算法
  - key_share附带秘钥参数

2. 服务端Hello
  - 服务端随机数
  - supported_versions确认使用TLS版本
  - Encrypted Extensions附带服务端扩展（详见[这里](https://tools.ietf.org/html/rfc8446#page-60)）
  - Server Certificate附带服务端证书
  - Server Certificate Verify附带前面数据使用服务端私钥做的签名，加强身份认证

3. 客户端收到数据，机密通讯开始

> 注：TLS1.3基本采用椭圆曲线特性加密，第二部服务端收到客户端**椭圆曲线的公钥**，然后用自己的**椭圆曲线的公钥**进行计算，可以算出第三个随机数`Pre-Master`，就可以生成主密钥，客户端同理，收到服务端回信也可生成主密钥。

> 注2：椭圆曲线计算`Pre-Master`大致计算流程[在这](https://blog.csdn.net/mrpre/article/details/78025940)，这个流程可以保证流程即便黑客破解客户端随机数以及服务端随机数也无法计算出`Pre-Master`

#### 0-RTT
实现通信后，服务端发送`Session Ticket`给客户端，由客户端进行存储，下次通信时，客户端通过`pre_shared_key`发送给服务端，并附带`early_data`数据，服务端进行解密，验证有效就可以恢复会话，直接加密通讯

> 注：前向安全：TLS1.3删除了不支持前向安全的密码套件，比如RSA；指的是如果破解服务端的秘钥，那么之前截获的所有数据都可以破解。

***

## HTTP/2
HTTPS解决了HTTP安全性的问题，但是性能问题没有解决，反而因为秘钥交换等原因，延迟更大，因此HTTP/2推了出来。
### 头部压缩
HTTP/1.1可以压缩body，但是头部却不能压缩，而我们有很多的请求其实类似`GET`类型，没有body或者body很轻，HTTP/2针对这个现象采用`HPACK`算法进行了优化。
基本原理：
* 维护一份相同的静态表（Static Table），包含常见的头部名称，以及特别常见的头部名称与值的组合；
  - 1.用于完全匹配的字段，比如`:method: GET`可以只发送`Key`，及一个数字即可；2.部分匹配可以发送头字段数字：value，如32表示Cookie，则可用`32: xxxxxx`，完整静态表在[这里](https://httpwg.org/specs/rfc7541.html#static.table.definition)；
* 维护一份相同的动态表（Dynamic Table），可以动态地添加内容；
  - 静态表中不存在的字段，可以更新到动态表中，下次只用发送一个编码就行。
* 支持基于静态哈夫曼码表的哈夫曼编码（Huffman Coding）
  - 发送数据可以使用哈夫曼编码，需要客户端以及服务端内嵌[哈夫曼编码表](https://httpwg.org/specs/rfc7541.html#huffman.code)

> 注：HTTP/1.1中的状态行信息也被拆分放入头部中，使用编号传递，但前面必须加冒号，如`2	:method	GET`

### 多路复用
#### 帧
HTTP/1.1采用的是纯文本形式，HTTP/2采用二进制格式，将原先的数据打散为数个**二进制帧**，使用`Headers`帧存放头数据，`Data`帧存放传输内容。

![未命名文件](https://user-images.githubusercontent.com/22512175/113655587-b4751980-96cc-11eb-86e4-c04040bae877.png)

#### 数据流、消息
新的二进制分帧模式改变了客户端和服务端之间的交流方式，这个过程中首先了解两个概念
* 数据流(Stream):已建立的连接的**双向**字节流，可以承载一条或多条消息。
* 帧(Frame):HTTP/2 通信的最小单位，每个帧都包含帧头，至少也会标识出当前帧所属的数据流。

在HTTP/1.1中，如果想要多个请求，必须开启多条TCP连接（数量会被浏览器限制），并且每个连接只能保证一个响应，容易引起队头阻塞，造成效率低下。

HTTP/2 中新的二进制分帧层突破了这些限制，实现了完整的请求和响应复用: 客户端和服务器可以将 HTTP 消息分解为互不依赖的帧，然后交错发送，最后再在另一端把它们重新组装起来。在HTTP/2的连接上，帧是无序分发的，但如果它们的流id相同，就表示属于同一个流，在同一个流下，帧是有严格的先后顺序的。

![未命名文件 (1)](https://user-images.githubusercontent.com/22512175/113724361-d8fce000-9724-11eb-9a84-6992f51754da.png)

`HTTP/2`中的新二进制分帧层解决了`HTTP/1.1`中存在的队头阻塞问题，也消除了并行处理和发送请求及响应时对多个连接的依赖。

> 注：流id是有上限的，达到上限之后，只能关闭这条链接重新开一条TCP链接。
