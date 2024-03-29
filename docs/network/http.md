# HTTP 与 HTTPS

## HTTP

HTTP 是超文本传输协议，HTTP 是一个在计算机世界里专门在两点之间传输文字、图片、音频、视频等超文本数据的约定和规范。

### HTTP 的特点

`特点：无连接、无状态、灵活`

`无连接：`每一次请求都要连接一次，请求结束就会断掉，不会保持连接（HTTP1.1后可以保持连接长连）；

`无状态：`每一次请求都是独立的，请求结束不会记录连接的任何信息，减少了网络开销，这是优点也是缺点

`灵活：`通过http协议中头部的Content-Type标记，可以传输任意数据类型的数据对象(文本、图片、视频等等)，非常灵活

`缺点：无状态、不安全、明文传输、队头阻塞`

`无状态：`请求不会记录任何连接信息，就无法区分多个请求发起者身份是不是同一个客户端的；

`不安全：`明文传输可能被窃听，缺少身份认证也可能遭遇伪装，还有缺少报文完整性验证可能遭到篡改

`明文传输：`报文(header部分)使用的是明文，直接将信息暴露给了外界，WIFI陷阱就是复用明文传输的特点，诱导你连上热点，然后疯狂抓取你的流量，从而拿到你的敏感信息

`队头阻塞：`开启长连接(下面有讲)时，只建立一个TCP连接，同一时刻只能处理一个请求，那么当请求耗时过长时，其他请求就只能阻塞状态(如何解决下面有讲)

### HTTP 报文组成
http报文：由请求报文和响应报文组成

请求报文：由`请求行、请求头、空行、请求体`四部分组成

响应报文：由`状态行、响应头、空行、响应体`四部分组成

* `请求行：`包括请求的方法，路径和协议版本

* `请求头/响应头：`包含了请求的一些附加的信息，一般是以键值的形式成对存在

* `空行：`协议中规定请求头和请求主体间必须用一个空行隔开，用来区分首部与实体，因为请求头都是key:value的格式，当解析遇到空行时，服务端就知道下一个不再是请求头部分，就该当作请求体来解析了；

* `请求体：`对于post请求，所需要的参数都不会放在url中，这时候就需要一个载体了，这个载体就是请求主体

* `状态行：`包含http协议及版本、数字状态码、状态码英文名称

* `响应体：`服务端返回的数据

### HTTP 的请求方法

HTTP1.0定义了三种请求方法： `GET`, `POST` 和 `HEAD`方法

HTTP1.1新增了五种请求方法：`OPTIONS`, `PUT`, `DELETE`, `TRACE` 和 `CONNECT`

http/1.1规定了以下请求方法(注意，都是大写):

`GET：` 请求获取Request-URI所标识的资源

`POST：` 一般用于修改Request-URI的资源

`HEAD：` 请求获取由Request-URI所标识的资源的响应消息报头

`PUT：` 请求服务器存储一个资源、

`DELETE：` 请求服务器删除对应所标识的资源

`TRACE：` 请求服务器回送收到的请求信息，主要用于测试或诊断

`CONNECT：` 建立连接隧道，用于代理服务器

`OPTIONS：` CORS跨域请求的预检请求；

### GET 和 POST的区别

从Restful语义来看：GET用来无副作用的请求资源，理应幂等，POST用来新资源；

从参数角度来看：GET请求一般放在URL中，POST请求放在请求体中，看起来post比get安全，但是在抓包的情况下都是一样的（所以面试的时候别再说POST比GET安全了）。

从编码角度看：GET只能进行 url 编码，参数的数据类型只接受ASCII字符，而POST支持更多的编码类型且不对数据类型限制。

从回退角度来看：GET在浏览器回退时是无害的，而POST会再次发起请求

从记录角度来看：GET请求参数会被完整保留在浏览器的历史记录里，而POST中的参数不会被保留

从发送角度来看：GET请求会一次性发送请求报文，POST请求通常分为两个TCP数据包，首先发 header 部分，如果服务器响应 100， 然后发 body 部分。

### http1.0 与 http1.1

`HTTP 1.0`

http1.0只支持POST/GET/HEAD命令

不支持断点续传，也就是说，每次都会传送全部的页面和数据。

只使用 header 中的`Last-Modified`、`If-Modified-Since`（协商缓存） 和 `Expires（强缓存）` 作为缓存失效的标准。

`HTTP 1.1`

引入了`持久连接`，即TCP连接默认不关闭，可以被多个请求复用，不用声明`Connection: keep-alive`；

引入了`管道机制`，即在同一个TCP连接里，客户端不用等待请求响应就可以发送多个请求，但要求服务端要按发送顺序返回；

支持断点续传，通过使用请求头中的 Range 来实现（206状态码）。

HTTP 1.1 中新增加了 `E-tag`、`If-None-Match`、`Cache-Control` 等缓存控制标头来控制缓存失效。

使用了虚拟网络，在一台物理服务器上可以存在多个虚拟主机，并且它们共享一个IP地址。

新增方法：`PUT`、 `OPTIONS`、 `DELETE`等。

更多的缓存标识：`Cache-Control`、`ETag`、`If-None-Match`

## Https

`HTTPS` 并不是一个新的协议，它只是在 `HTTP` 和 `TCP` 的传输中建立了一个安全层，它其实就是 `HTTP + SSL/TLS` 协议组合而成，而安全性的保证正是 `SSL/TLS` 所做的工作。

## HTTPS 和 HTTP 区别

HTTP 是明文传输，HTTPS 协议是可进行加密传输、身份认证的网络协议，比 HTTP 协议安全。

HTTPS对搜索引擎更友好，利于 SEO；谷歌、百度优先索引 HTTPS 网页。

HTTPS 标准端口 443，HTTP 标准端口 80。

HTTPS 需要用到SSL证书，而 HTTP 不用。

### HTTPS 握手过程

* 客户端发起 HTTPS 请求，发送客户端生成的随机数和支持的加密算法列表；

* 服务端返回证书、服务端生成的随机数、选择的加密方法给客户端；

* 客户端对证书进行合法性验证，验证通过后再生成一个随机数

* 客户端通过证书中的公钥对随机数进行加密传输到服务端，服务端接收后通过私钥解密得到随机数

* 三次握手此时已经完成，之后客户端和服务端都会根据这三个随机数，生成一个随机对称密钥，之后的数据都通过随机对称密钥进行加密传输。

### Https 中间人攻击

![image-20210708113111640](../../img/https.png)

1.本地请求被劫持（如DNS劫持等），所有请求均发送到中间人的服务器

2.中间人服务器返回中间人自己的证书

3.客户端创建随机数，通过中间人证书的公钥对随机数加密后传送给中间人，然后凭随机数构造对称加密对传输内容进行加密传输

4.中间人因为拥有客户端的随机数，可以通过对称加密算法进行内容解密

5.中间人以客户端的请求内容再向正规网站发起请求

6.因为中间人与服务器的通信过程是合法的，正规网站通过建立的安全通道返回加密后的数据

7.中间人凭借与正规网站建立的对称加密算法对内容进行解密

8.中间人通过与客户端建立的对称加密算法对正规内容返回的数据进行加密传输

9.客户端通过与中间人建立的对称加密算法对返回结果数据进行解密