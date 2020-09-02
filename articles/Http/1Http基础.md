# Http那些必须知道的基础知识点

**本文以[鸿洋](https://juejin.im/user/1697301652584670)提供的 [玩Android 开放API](https://wanandroid.com/blog/show/2) 举例**

### 1.互联网的分层

网络结构有两种主流的分层方式：**OSI七层模型**和**TCP/IP四层模型**。

OSI是指Open System Interconnect，意为开放式系统互联。
TCP/IP是指传输控制协议/网间协议，是目前世界上应用最广的协议。

图

### 2.Http 定义

Hypertext Transfer Protocol，超⽂本传输协议，超⽂本即扩展型⽂本，指的是 HTML 中可以有链向别的⽂本的链接（hyperlink）

HTTP 是基于 TCP/IP 协议的[**应用层协议**](http://www.ruanyifeng.com/blog/2012/05/internet_protocol_suite_part_i.html)。它不涉及数据包（packet）传输，主要规定了客户端和服务器之间的通信格式，默认使用80端口，是无状态的。

目前广泛使用的是Http/1.1版本

### 3.URL 格式

三部分：协议类型、服务器地址(和端⼝号)、路径(Path)

协议类型://服务器地址[:端⼝号]路径

例如：https://wanandroid.com/blog/show/2

### 4.Http报文

##### 4.1.请求报文

这是请求的wanandroid的 OPEN API 接口

```http
POST /user/login HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 32
Host: www.wanandroid.com
Connection: Keep-Alive
Accept-Encoding: gzip
User-Agent: okhttp/4.3.1

username=AboBack&password=123456
```

* 第一行为请求行，包含了请求方法，请求Path，Http的版本
* 第二行和最后一行之间为Header 部分，也叫请求头，下面会具体说一下每个请求头的意思，最后一个请求头，需要回车换行，来和请求体分隔。
* 最后一行为请求体，也就是 Body 部分，这个部分是跟 Header 的Content-Type相关的，下面会具体说

##### 4.2.响应报文

```http
HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Set-Cookie: JSESSIONID=A80385088117D7F72F057C09A3BB1562; Path=/; Secure; HttpOnly
Set-Cookie: loginUserName=jhbxyz; Expires=Fri, 02-Oct-2020 05:42:35 GMT; Path=/
Set-Cookie: token_pass=42d528ea2b5a1b10fdfcdfebf2fbe862; Expires=Fri, 02-Oct-2020 05:42:35 GMT; Path=/
Set-Cookie: loginUserName_wanandroid_com=jhbxyz; Domain=wanandroid.com; Expires=Fri, 02-Oct-2020 05:42:35 GMT; Path=/
Set-Cookie: token_pass_wanandroid_com=42d528ea2b5a1b10fcfcdfebf2fbe862; Domain=wanandroid.com; Expires=Fri, 02-Oct-2020 05:42:35 GMT; Path=/
Content-Type: application/json;charset=UTF-8
Date: Wed, 02 Sep 2020 05:42:34 GMT
Transfer-Encoding: chunked
Connection: Keep-alive

{"data":{"admin":false,"chapterTops":[],"coinCount":0,"collectIds":[],"email":"","icon":"","id":75,"nickname":"jhbxyz","password":"","publicName":"jhbxyz","token":"","type":0,"username":"jhbxyz"},"errorCode":0,"errorMsg":""}
```

* 第一行为状态行，包含了Http的版本，状态码，状态的简单描述
* 第二行和最后一行之间为Header 部分，也叫响应头，下面会具体说
* 最后一行为是 Body 部分。

##### 4.3.请求头和响应头的部分字段

* Host：⽬标主机。注意：不是在⽹络上⽤于寻址的，⽽是在⽬标服务器上⽤于定位⼦服务器的，可用来区分访问一个服务器上的不同服务。
* Content-Type：指定 Body 的类型
* Content-Length：指定 **Body** 的⻓度（字节）。
* Connection：**keep-alive**表示要求服务器不要关闭TCP连接，**close**表示明确要求关闭连接，默认值是keep-alive
* Accept-Encoding：客户端接受的压缩编码类型，如 gzip
* Content-Encoding：压缩类型。如 gzip

* Location：指定重定向的⽬标 URL
* User-Agent：⽤户代理，即是谁实际发送请求、接受响应的，例如⼿机浏览器、某款⼿机 App。

* Accept: 客户端能接受的数据类型。如 text/html
* Accept-Charset: 客户端接受的字符集。如 utf-8

##### 4.4.重点说一个Content-Type

Content-Type指定 **Body** 的类型。主要有四类

* application/x-www-form-urlencoded（普通表单）：用于客户端向服务器请求时，它对应的 body 类似 url 的格式`username=AboBack&password=123456`
* multipart/form-data：含有⼆进制⽂件时的提交⽅式
* application/json ：以客户端服务端之间 json 来传输数据
* text/html：返回响应的类型，Body 中返回 html ⽂本

### 5.请求方法和状态码

##### 5.1请求方法

**GET**

- ⽤于获取资源
- 对服务器数据不进⾏修改
- 不发送 Body

**POST**

- ⽤于**增加或修改**资源
- 发送给服务器的内容写在 Body ⾥⾯

**PUT**

- ⽤于**修改**资源
- 发送给服务器的内容写在 Body ⾥⾯

**DELETE**

- ⽤于删除资源
- 不发送 Body

**HEAD**

- 和 GET 使⽤⽅法完全相同
- 和 GET 唯⼀区别在于，返回的**响应中**没有 Body，用来获取信息（速度快，传输快，因为没有 Body），主要是 Header 中的信息，比如下载的时候

##### 5.1状态码

三位数字，⽤于对响应结果做出类型化描述（如「获取成功」「内容未找到」）。

* 1xx：临时性消息。如：100 （继续发送）、101（正在切换协议，比如Http1.1 和 Http2 的协议切换）
* 2xx：成功。最典型的是 200（OK）、201（创建成功）。206 Partial Content 成功执行一个部分请求。这个在用于断点续传时会涉及到。
* 3xx：重定向。如 301（永久移动，通常会在Location首部中包含新的URL用于重定向）、302（暂时移动，通常会在Location首部中包含新的URL用于重定向）、304（内容未改变）。
* 4xx：客户端错误。如 400（客户端请求错误）、401（认证失败）、403（被禁⽌）、404（找不到内容）。
* 5xx：服务器错误。如 500（服务器内部错误）。

### 6.TCP/IP 协议

* IP：IP 协议提供了**主机和主机**间的通信

* TCP：IP 协议提供了主机和主机间的通信。TCP 协议在 IP 协议提供的主机间通信功能的基础上，完成这**两个主机**上**进程对进程**的通信，**面向连接**（*三次握手*）有状态的

* 端口号 ：为了**标识**数据属于哪个**进程**，我们给需要进行 TCP 通信的进程分配一个唯一的数字来标识它

* Socket：Socket也称作套接字，是在应用层和传输层之间的一个抽象层，它把TCP/IP层复杂的操作抽象为几个简单的接口供应用层调用以实现进程在网络中通信； Socket是 TCP 层的封装，通过 socket，我们就能进行 TCP 通信

### 7.TCP的三次握手

为什么要进行三次握手？

因为经过三次握手就能确认自己和对方的收发信能力，三次握手完成后，连接便建立了。这时候我们才可以开始发送/接收数据

图

首先，介绍一下几个概念：

-  `ACK`：响应标识，1表示响应，连接建立成功之后，所有报文段ACK的值都为1
-  `SYN`：连接标识，1表示建立连接，连接请求和连接接受报文段SYN=1，其他情况都是0
-  `FIN`：关闭连接标识，1标识关闭连接，关闭请求和关闭接受报文段FIN=1，其他情况都是0，跟SYN类似
-  `seq number`：序号，一个随机数X，请求报文段中会有该字段，响应报文段没有
-  `ack number`：应答号，值为请求seq+1，即X+1，除了连接请求和连接接受响应报文段没有该字段，其他的报文段都有该字段

知道了上面几个概念后，看一下三次握手的具体流程：

1. 第一次握手：建立连接请求。客户端发送连接请求报文段，将SYN置为1，seq为随机数x。然后，客户端进入SYN_SEND状态，等待服务器确认。
2. 第二次握手：确认连接请求。服务器收到客户端的SYN报文段，需要对该请求进行确认，设置ack=x+1（即客户端seq+1）。同时自己也要发送SYN请求信息，即SYN置为1，seq=y。服务器将SYN和ACK信息放在一个报文段中，一并发送给客户端，服务器进入SYN_RECV状态。
3. 第三次握手：客户端收到SYN+ACK报文段，将ack设置为y+1，向服务器发送ACK报文段，这个报文段发送完毕，客户端和服务券进入ESTABLISHED状态，完成Tcp三次握手。

从图中可以看出，建立连接经历了**三次握手**，当数据传输完毕，需要断开连接，而断开连接经历了**四次挥手**：

1. 第一次挥手：主机1（可以是客户端或服务器），设置seq和ack向主机2发送一个FIN报文段，此时主机1进入FIN_WAIT_1状态，表示没有数据要发送给主机2了
2. 第二次挥手：主机2收到主机1的FIN报文段，向主机1回应一个ACK报文段，表示同意关闭请求，主机1进入FIN_WAIT_2状态。
3. 第三次挥手：主机2向主机1发送FIN报文段，请求关闭连接，主机2进入LAST_ACK状态。
4. 第四次挥手：主机1收到主机2的FIN报文段，想主机2回应ACK报文段，然后主机1进入TIME_WAIT状态；主机2收到主机1的ACK报文段后，关闭连接。此时主机1等待主机2一段时间后，没有收到回复，证明主机2已经正常关闭，主机1页关闭连接。



### 参考文章

[HTTP协议理解及服务端与客户端的设计实现](https://yanzhenjie.blog.csdn.net/article/details/93098495)

[HTTP 协议入门](http://www.ruanyifeng.com/blog/2016/08/http.html)

[HTTPS 为什么是安全的](https://hencoder.com/tag/https/)

[全面了解HTTP和HTTPS（开发人员必备）](https://www.jianshu.com/p/27862635c077)

[HTTP 必知必会的那些](https://mp.weixin.qq.com/s/Fazx13maQfPJItfkOqk9FQ)

[手把手教你写 Socket 长连接](https://mp.weixin.qq.com/s/035SiIDhmbHzbtI1fF4h1g)