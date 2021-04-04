# HTTP详解

> 最近阅读了《透视HTTP协议》，同时系统性梳理了一遍自己对于HTTP协议的问题栈，希望能从头到尾把http的知识梳理清楚，本文会从http/1.1到http/2，如有不正确的地方，希望可以指出 :wink:

## HTTP/1.1
***
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

![系统架构设计 (2)](https://user-images.githubusercontent.com/22512175/113499721-bfa03c00-954a-11eb-9c5c-6705aa933ee3.png)

报文大致由3个部分组成
1. 请求行：描述请求或响应的基本信息
2. 头部字段集合：一般为key-value集合详细说明报文
3. 消息体：请求传输的数据结构

> 注：header与body之前需要有一个空行分割，同时，可以没有body，但必须有header

![WeChatWorkScreenshot_9c3d231a-b657-4273-8234-2f4084536a08 copy](https://user-images.githubusercontent.com/22512175/113504147-c9389c80-9568-11eb-8e3c-0f39cea2ad94.jpg)

##### 请求行

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
