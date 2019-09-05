## 6.1 HTTP2

#### 6.1.1 新特性

- 新的二进制格式：HTTP 1.x的解析是基于文本，HTTP 2.0的解析是基于二进制，更加适用于服务器间信息传输了。
- 多路复用（MultiPlexing）：也就是连接共享，每个请求都有一个ID，这样在一个连接上可以发送多个请求，并且它们在传输过程中是混杂在一起的，接收方可以根据请求的ID将请求再归属到不同的服务端请求里面。
- 请求优先级（request prioritization）：HTTP/2 里的每个 stream 都可以设置依赖 (Dependency) 和权重，可以按依赖树分配优先级，解决了关键请求被阻塞的问题
- header压缩：通信上方各自缓存一份Header fields表，只传输压缩之后的相应编码，避免了重复的header传输。
- 服务端推送（server push）：浏览器发送一个请求，服务器主动向浏览器推送与这个请求相关的资源，这样浏览器就不用发起后续请求。
  - 客户端可以缓存推送的资源
  - 客户端可以拒收推送过来的资源
  - 推送资源可以由不同页面共享
  - 服务器可以按照优先级推送资源

#### 6.1.2  和HTTP1.X区别

- HTTP 1.0中一次请求响应就建立一次连接，用完关闭，每个请求都要建立一个连接。

- HTTP 1.1 Pipeling解决方式，也是我们常说的Keep-Alive模式，建立一次连接之后，若干个请求排队串行化单线程处理，后面的请求等待前面的请求返回了获得执行机会，一旦有请求超时等待，后续的请求只能被阻塞，毫无办法，也就是人们常说的线头阻塞（Head-of-Line Blocking）。

- HTTP 2.0多路复用，多个请求同时在一个连接上并行执行，若某个请求任务耗时严重，不会影响到其它请求的正常执行。



#### 6.1.3  **HTTP/2 & HTTPS**

>  HTTPS是建构在SSL/TLS之上的HTTP协议，一种保证信息传输安全的协议，HTTP/2的定义本身是和HTTPS无关的。但是，在开放的互联网上HTTP/2将只用于https://网址，而 http://网址将继续使用HTTP/1.x，目的是在开放互联网上增加使用加密技术，以提供强有力的保护去遏制主动攻击。所以，在进行HTTP/2推广的同时也在强制推广HTTPS。

>  目前大部分HTTP/2的实现都是要强制使用HTTPS的，很少能见到有直接在TCP层之上实现的HTTP/2。

> 因此，可能会有人说建立在HTTPS之上会额外消耗很多性能，这个答案是肯定的。HTTP使用TCP三次握手建立连接，客户端和服务器需要交换3个包，HTTPS除了TCP的三个包，还要加上SSL/TLS握手需要的9个包，所以一共是12个包，增加了建立连接的时间。但是HTTP/2的多路复用技术，建立一次连接可以一直发送请求，因此HTTPS带来的性能影响可以忽略不计。



#### 6.1.4  Spring BOOT2使用

> HTTP/2的实现都是强制配置HTTPS的，Spring Boot的实现也不例外，因此我们先将HTTP配置成HTTPS，配置HTTPS有需要证书。

#####  环境要求

- Undertow 2.0.16.Final、JDK 8
- Tomcat8.5以上、JDK8 (需要安装ARP、openssl、tomcat-native等软件)
- Tomcat9、JDK11(推荐)

##### 属性配置

```properties
server.http2.enabled=true
server.ssl.key-store=classpath:rabbitkeystore.jks
server.ssl.key-store-password=rabbit
server.ssl.key-password=rabbit
```