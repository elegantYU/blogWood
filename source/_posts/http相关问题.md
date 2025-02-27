---
title: 关于http的一一二二
date: 2023-11-15 17:44:12
tags:
---
## 缓存机制
在web中，http请求一般都是浏览器发起的，所以我们这里所说的http的缓存策略，其实也就是浏览器端的缓存策略，因为http本身只是一种协议，真正实现缓存还是要靠浏览器（其实就是浏览器指定存储在硬盘下。）
首先，我们要知道一点：http的缓存策略，是由客户端和服务器端共同去控制的，客户端可以通过在请求头里添加Cache-Control等字段来决定是否走缓存，服务器端也可以在响应头中添加Cache-Control等字段来告诉客户端是否可以缓存数据。
不管是客户端还是服务器端都是通过http头中的不同字段来控制的。
![缓存流程](https://s2.loli.net/2023/11/15/M1xqdRwAlSy4EoD.png)

### 强缓存
#### cache-control (http1.1提出)
![cache-control](https://s2.loli.net/2023/11/15/IAUNJOQkyCH8ahr.png)
#### Expires （优先级低）
Expires表示服务器端告诉客户端当前资源的失效时间，截止到哪个时间点，是一个绝对时间，即过了这个时间点请求的话，就说明缓存已经失效啦，但是由于服务器端时间和客户端时间可能存在偏差，这也就是导致了最后缓存的时间误差，另一方面，该字段是http1.0提出来的，现在我们基本都是用cache-control:max-age:30来替代。
**Expires 是 HTTP/1.0 的首部，Cache-Control 是 HTTP/1.1 的首部，Expires 首部和 Cache-Control:max-age 首部所做的事情本质上是一样的，但由于 Cache-Control 首部使用的是相对时间而不是绝对日期，所以更倾向于使用比较新的Cache-Control首部。绝对日期依赖于计算机时钟的正确设置。**
{% note primary%}
强制再验证：Pragma: no-cache
与 Cache-Control: no-cache 效果一致，当响应头中包含该指令时，当客户端再次发起请求时，会强制要求使用缓存之前将请求提交到源服务器进行验证。
{% endnote %}
{% note primary %}
cache-control优先级大于expires，只看cache-control
因为现在都是http1.1，所以关于强缓存真正起到作用的就是 cache-control 字段。
{% endnote %}
{% note success%}
唠一下 etag的生成
```js
ETAG: <weak> opaque-tag 
```
weak表示强校验和弱校验。初始带有W/表示弱校验。
区别在于强校验是逐字节的对比，而弱校验是语义相等。
HTTP/1.1协议虽然提出了 ETag，但并没有规定ETag的内容是什么或者说要怎么实现，唯一规定的是ETag的内容必须放在""内。
虽然说他没有规定，但是一般而言：
ETag生成结论
1. 对于静态文件（如css、js、图片等），ETag的生成策略是：文件大小的16进制+修改时间
2. 对于字符串或Buffer，ETag的生成策略是：字符串/Buffer长度的16进制+对应的hash值
{% endnote %}
{% note warning%}
Q:那么 etag变化了，文件一定修改了吗？
A:不一定，根据etag的生成算法决定，如果是根据修改时间判断的话，比如编辑了文件但没有改变文件内容的话，则etag改变而文件没有修改。
{% endnote %}
### 协商缓存（服务端再验证）
#### Etag/If-none-match
If-None-Match 的值为服务端上一次返回的 ETag 的值。在 ETag 的值前面添加 W/ 前缀表示可以采用相对宽松的算法。
Etag是上一次加载资源时，服务器生成的唯一标识，如果文件修改，则etag值会变化
#### Last-modified/If-modified-since
If-Modified-Since 的值为服务端上一次返回的 last-modified 的值。如果在此期间内容被修改了，最后的修改日期就会有所不同，源服务器就会回送新的文档。否则，服务器会认为缓存的最后修改日期与服务器文档当前的最后修改日期相符，会返回一个 304 NotModified 响应。
Last-Modified是该资源文件最后一次更改时间,服务器会在response header里返回
{% note primary %}
如果两个字段都收到，则要同时满足这两个条件才可以用缓存。
一般都是同时启用。
因为分布式系统尽量关掉ETag，因为每台机器生成的ETag不一样。这个时候就需要last-modified
{% endnote %}
{% note success %}
为什么etag的优先级更高呢
在精确度上，Etag要优于Last-Modified，Last-Modified的时间单位是秒，如果某个文件在1秒内改变了多次，那么他们的Last-Modified其实并没有体现出来修改，但是Etag每次都会改变确保了精度
在性能上，Etag要逊于Last-Modified，毕竟Last-Modified只需要记录时间，而Etag需要服务器通过算法来计算出一个hash值。
**在优先级上，服务器校验优先考虑Etag。**
所以，两者互补。
{% endnote %}

## 用户行为对浏览器缓存的控制
#### 地址栏访问
链接跳转或者是书签页打开，是正常的用户行为，会走浏览器缓存
#### F5刷新
浏览器会将请求头设置为max-age:0，跳过强制缓存，但还是会走协商缓存，浏览器会对本地文件过期，但会带上其他两个头。根据if-none-match、if-modified-since判断。也就是说会检查新鲜度。
#### ctrl+F5强制刷新
跳过强缓存和协商缓存，直接从服务器拉最新的资源。浏览器不仅会对本地文件过期，也不会带上其他两个头。相当于之前从来没有请求过，也就是200

### 如何不走缓存
#### cache-control
- no-cache：无论资源是否过期，都去协商缓存
- no-store:禁止一切缓存
- must-revalidate
#### expires
设置在当前时间之前
#### 前端如何设置呢
- 引入js，css文件url加上Math.random()值
- 页面上设置不让缓存
```js
<meta http-equiv="pragma" content="no-cache"> 
<meta http-equiv="Cache-Control" content="no-cache, must-revalidate"> 
<meta http-equiv="expires" content="Wed, 26 Feb 1997 00:00:00 GMT">
```
### 其他
强缓存会有两种形式存在， memory cache  和  disk cache 
- memory cache 资源存在内存中，如js，图片，字体等
- disk cache 非js文件，如css。

## http1.1和http2的改变
### http1.1的区别
1. 缓存处理：引入了cache-control、etag等缓存
2. 范围请求：在请求头添加了range头域，允许只请求资源的某个部分，返回206，支持断点传续
3. 新增24个错误码：206 409（请求资源与当前资源冲突） 410（服务器上的资源被永久删除）
4. Host头：HTTP1.0中认为每台服务器都绑定一个唯一的IP地址，因此， 请求消息中的URL并没有传递主机名（hostname） 。但随着虚拟主机技术的发展，在一台物理服务器上可以存在多个虚拟主机（Multi-homed Web Servers），并且它们共享一个IP地址。 HTTP1.1的请求消息和响应消息都应支持Host头域，且请求消息中如果没有Host头域会报告一个错误（400 Bad Request） 。有了Host字段，就可以将请求发往同一台服务器上的不同网站，为虚拟主机的兴起打下了基础。
5. 持久连接：connection：keep-alive就是tcp默认不关闭，可以被多个请求复用
6. 管道机制： 在一个tcp连接中，客户端可以同时发起多个请求。


#### http1.1

**优点**

- 1.0 每个 tcp 只能发送一个请求，1.1 引入长连接，tcp 链接默认不关闭，可以被多个复用
- 管道化，同个 tcp 连接里，客户端可以同时发送多个 http 请求，不用等待一个个响应 (6-8 限制)
- 支持断点续传

**缺点**

- 队头阻塞
  
  由于 http 遵守“请求-响应”的模式，页面中发起多个请求，**同一个 TCP 链接中**，每个请求必须等到前一个请求相应之后才能发送。如果一个 TCP 通道中某个 http 请求没有及时返回，后面的相应会被阻塞

- 不安全性

  无法验证通信双方的身份，也不能判断报文是否被窜改  

- 明文传输
  
  数据肉眼可见，能够方便地研究分析，但也容易被窃听

### http2的改动

1. 二进制分帧：HTTP/1.1的头信息是文本（ASCII编码），数据体可以是文本，也可以是二进制；HTTP/2 头信息和数据体都是二进制，统称为{% label primary @“帧” %}：头信息帧和数据帧；

2. 多路复用： 也就是双工通信。可以发起多重的请求-响应消息。就是说在一个连接里，客户端和服务端都可以发起多个请求和响应，不用按顺序。也就是解决了队头堵塞的这个问题。HTTP/2 把 HTTP 协议通信的基本单位缩小为一个一个的帧，这些帧对应着逻辑流中的消息。并行地在同一个 TCP 连接上双向交换消息。

3. 数据流：因为 HTTP/2 的数据包是不按顺序发送的，同一个连接里面连续的数据包，可能属于不同的回应。因此，必须要对数据包做标记，指出它属于哪个回应。HTTP/2 将每个请求或回应的所有数据包，称为一个数据流（stream）。每个数据流都有一个独一无二的编号。数据包发送的时候，都必须标记数据流ID，用来区分它属于哪个数据流。另外还规定，客户端发出的数据流，ID一律为奇数，服务器发出的，ID为偶数。数据流发送到一半的时候，客户端和服务器都可以发送信号（RST_STREAM帧），取消这个数据流。HTTP/1.1取消数据流的唯一方法，就是关闭TCP连接。这就是说，HTTP/2 可以取消某一次请求，同时保证TCP连接还打开着，可以被其他请求使用。客户端还可以指定数据流的优先级。优先级越高，服务器就会越早回应。

**请求优先级**

4. 首部压缩：HTTP 协议不带有状态，每次请求都必须附上所有信息。所以，请求的很多字段都是重复的，，一模一样的内容，每次请求都必须附带，这会浪费很多带宽，也影响速度。HTTP/2 对这一点做了优化，引入了头信息压缩机制（header compression）。一方面，头信息压缩后再发送（SPDY 使用的是通用的DEFLATE 算法，而 HTTP/2 则使用了专门为首部压缩而设计的 HPACK 算法）。；另一方面，客户端和服务器同时维护一张头信息表，所有字段都会存入这个表，生成一个索引号，以后就不发送同样字段了，只发送索引号，这样就提高速度了。

5. serve push 服务端推送： 允许服务端主动向客户端发起请求。常见场景是客户端请求一个网页，这个网页里面包含很多静态资源。正常情况下，客户端必须收到网页后，解析HTML源码，发现有静态资源，再发出静态资源请求。其实，服务器可以预期到客户端请求网页后，很可能会再请求静态资源，所以就主动把这些静态资源随着网页一起发给客户端了。


#### http2.0性能瓶颈
启用http2.0后会给性能带来很大的提升，但同时也会带来新的性能瓶颈。因为现在所有的压在底层一个TCP连接之上，TCP很可能就是下一个性能瓶颈，比如TCP分组的队首阻塞问题，单个Tcp的packet丢失导致整个连接阻塞，无法逃避，此时所有消息都会受到影响。未来，服务器端针对ht2.0下的TCP配置优化至关重要。
### https
ssl/tls：公钥加密法
客户端向服务端索要并验证公钥，然后用公钥加密传输，生成对话秘钥（非对称加密），对方采用对话秘钥进行通信（对称加密）

## PWA & Service worker
Progressive Web Apps  一个渐进式的web应用，主要是为了提升用户的体验，不是某个单一的技术，其核心技术包括 Web App Manifest，Service Worker，Web Push 等。
#### service worker 和 workder的区别

1. worker 是一种web worker技术，他允许独立于主线程运行的一个脚本，可以放一些{% label primary @计算密集型任务 %}在worker里，防止阻塞主线程渲染。
2. service worker 是一种全新的web worker技术，他可以用来管理网络请求，缓存数据，包括离线优化等等。还可以发送推送通知等等。是浏览器在后台独立于网页运行的脚本.
{% note primary%}
service worker拥有更高的权限，可以管理网络，缓存，通知等等，他们都是通过postmessage通信的。
{% endnote %}

## get和post的区别
<https://github.com/febobo/web-interview/issues/145>


