## 1.HTTP协议的工作流程
1. HTTP协议和HTTPS协议的区别：
    1. 两者都处于应用层，HTTP是基于TCP协议的。HTTPS协议则是基于TLS、SSL协议层之上的协议。
    ![image.png](https://upload-images.jianshu.io/upload_images/17728742-744a3fd8aae571d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. HTTP的默认端口是80，HTTPS的默认端口是443
3. HTTP的工作过程：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-d634d7f75a38930d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.HTTP的协议结构
##### 1.请求报文的组成
1. 请求报文由以下几个部分组成：
    1. 请求行
    2. 请求头部
    3. 请求空行
    4. 请求体
2. 常见的请求头字段如下：
    1. Host：指定被请求资源的网络主机和端口号
    2. Connection：它的值通常有两个，分别是keep-alive和close。keep-alive可以使得客户端到服务器端的连接持续有效，这样同一个客户端对服务器发起的后续请求，就不用重新建立一个新的客户端到服务器端的连接。
        1. Connection的值一般为keep-alive，这个功能是Http/1.1中引入的。
    3. Accept：指定浏览器端可以接收的MIME类型。一般这个字段的值为`*/*`,表示浏览器可以处理所有类型
    4. Cache-Control：指定请求和响应遵循的缓存机制。缓存指令是单向的，响应中出现的缓存指令在请求中未必会出现；缓存指令是独立的，在请求消息或者响应消息中设置Cache-Control并不会修改另一个消息处理过程中的缓存处理过程。其值一般有：
        1. no-cache：指所有内容不会被缓存
    5. Accept-Encoding：浏览器声明自己可以接收的编码方法。其值通常是`gzip, deflate, br`，表示浏览器支持压缩，压缩方法为gzip
    6. Accept-Language：浏览器声明自己接收的语言。例如其值为` zh-CN,zh;q=0.9`
    7. Accept-Charset：指定浏览器可以接受的字符集。如果在请求消息中没有设置这个域，默认时表示任何字符集都可以接受。
    8. User-Agent：通知HTTP服务器，客户端使用的操作系统和浏览器的名称和版本
    9. If-None-Match：这个字段的值和ETag一起工作，工作原理是在HTTP Response中添加ETag信息。当用户再次请求该资源时，将在HTTP Request中加入If-None-Match信息。如果服务器验证资源的ETag没有改变，表示资源没有更新，将返回一个304状态告诉客户端使用本地缓存文件。否则将使用200状态和新的资源和ETag。
    10. Referer：包含一个URL，用户从该URL代表的页面出发访问当前请求的页面
##### 2.响应报文的组成
1. 响应报文由以下几个部分组成：
    1. 状态行
    2. 响应头
    3. 响应空行
    4. 响应体
2. 常见的响应头字段如下：
    1. Server：指定Web服务器的信息
    2. Content-Type：Web服务器通知浏览器自己响应的对象的长度
## 3.HTTPS
1. HTTPS：全称hypertext transfer protocol over secure socket layer，它是以安全为目标的HTTP通道。
##### 1.加密算法
1. 加密算法分为对称加密和非对称加密。
    1. 对称加密：指的是加密的密钥和解密的密钥是一样的，特点是计算量小。==一句话：解铃还需系铃人==
    2. 非对称加密：指的是加密的密钥和解密的密钥是不一样的，也就是说密钥成对出现。公钥加密需要私钥解密，私钥加密需要公钥解密。特点时计算量大。==基于性能的考虑，一般使用非对称加密算法得出密钥，再用对称加密算法对消息内容进行加密，然后再进行传输。==
##### 2.TLS
1. TLS全称Transport Layer Security，它是SSL的后续版本。
![image.png](https://upload-images.jianshu.io/upload_images/17728742-e84a75150b0d86a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
2. TLS的工作流程：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-1daf2aa7343ced45.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3. http和https的区别：
![image.png](https://upload-images.jianshu.io/upload_images/17728742-db4853499a97f505.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.CGI
1. CGI全称Common Gateway Interface，是HTTP协议中最重要的技术之一，是一个web服务器提供信息服务的标准接口。
2. 客户端与CGI的通信
![image.png](https://upload-images.jianshu.io/upload_images/17728742-dadc546dd75c7dc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)