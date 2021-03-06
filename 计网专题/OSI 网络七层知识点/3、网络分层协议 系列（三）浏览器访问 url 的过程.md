# 浏览器访问 url 的超详细过程



[ 在浏览器地址栏输入一个URL后回车，背后会进行哪些技术步骤？ - 车小胖的回答 - 知乎](https://www.zhihu.com/question/34873227/answer/518086565 )



大致流程：

- URL 解析
- DNS 查询
- TCP 连接
- SSL 连接
- 发送 HTTP 请求
- 接收响应
- 渲染页面



## 1、URL 解析

首先会判断输入的 URL 的是一个 合法的 URL 还是一个 搜索的关键词，并且根据输入的内容自动进行分词、字符编码等操作



需要判断请求的内容在浏览器中是否存在缓存，

- 如果有缓存，判断缓存是否已经过期
  - 如果缓存没有过期，则读取缓存，加载页面
  - 如果缓存过期了则继续向服务器请求，询问资源是否更新，如果没有更新，返回 304，那么继续使用原来的缓存，如果更新了，那么获取返回的资源，将资源存入缓存中，加载页面
- 如果没有缓存，那么向服务器请求，获取返回的资源，加入到缓存中，加载页面

 ![img](https://pic1.zhimg.com/80/v2-0489444034d569b37867e2e527a7d5d4_720w.jpg) 



如果输入的是 zhihu.com，那么浏览器怎么知道访问的是 https 还是 http 呢？

**浏览器有自己的方案，它会先默认访问 http**，即 `http://zhihu.com`，除非输入的是 `https://zhihu.com` 指定访问 https



## 2、DNS 查询



[DNS 查询过程](https://www.zhihu.com/question/372902597/answer/1068847429) 



由于 浏览器知道 TCP 只能在知道 目的 IP 地址 的情况下才能够进行传输，因此需要浏览器需要进行 DNS 查询，获取域名对应的 ip 地址，查询的域名是 `www.zhihu.com`



## 3、TCP 连接

通过 DNS 查询获取到 IP 地址后，开始 TCP 三次握手建立连接

同样的 ，`zhihu.com` 的主机不会跟当前主机在同一个网络段，所以 TCP 握手数据包的传输 需要经历 ARP 获取网关的 MAC 地址，然后到达 网关 再 路由转发 到达 目的主机 的过程

三次握手建立后，会封装一个 HTTP 请求报文，然后交给 TCP ，TCP 会将 HTTP 请求报文分割为多个报文段，并且对每个报文段进行标号，用于后续有序拼接，然后将每个报文段按序交给 IP 层

同样经过路由转发的过程到达 目的主机（以后都省略这个步骤，因为无论是什么数据只要需要传输都需要经过路由转发，但是只需要第一次进行 ARP 广播获取 跨网段用的 网关的 MAC 地址即可）



这时候，如果 zhihu 使用的是 https，那么 zhihu 后台服务器 会同时监听 http 协议 和 https 协议 的两个端口，由于浏览器默认是访问 http，因此这第一次发送的 HTTP 请求报文会发送到 zhihu 监听的 http 协议上，而此时 zhihu 会响应一个 302 状态码的报文，(**临时重定向，表示资源还在，不过需要使用另外一个 URL 访问**)，告知我们需要重新去访问 https.zhihu.com

因此，浏览器需要重新去访问 https.zhihu.com

```java
/*
注：为了防止像这样每次要先访问 http 再去访问 https，
因此 zhihu 服务器返回的 302 报文中有这么一个字段：Strict-Transport-Security: max-age=31536000
max-age 表示在设置的时间到期之前，访问 http.zhihu.com 时将转换为访问 https.zhihu.com
以此来避免频繁的两次访问
*/
```



## 4、SSL 连接

当得到需要访问 https.zhihu.com ，浏览器会 以 https 的方式去访问这个地址，由于域名不变，所以 ip 地址是不变

重新进行 TCP 三次握手，然后此次使用的是 https 协议，

所以进行 HTTP 通信之前需要进行加密，因此进行 SSL 四次握手：

- 客户端发送 自己支持的 SSL 版本号、自己支持的 加密算法列表、一个随机数（第一个随机数）
- 服务端收到后，发送自己的 CA 证书、选定的 加密算法、一个随机数（第二个随机数）
- 客户端收到后，根据 CA 证书上的机构在自己电脑上找到内置的 CA 公钥，然后配合 CA 证书上的 散列算法 进行 hash 计算得到签名，然后将这个签名 和 CA 证书上的签名进行比对，如果一致，表示有效，这样就放心存储 CA 证书上服务器的公钥
- 客户端验证完成后，生成一个随机数（第三个随机数），然后使用 包括前面两个随机数在内的 三个随机数 根据选定的 散列算法生成 密钥，然后随机生成一段字符串，使用密钥加密，作为握手信息，再把第三个随机数和握手信息使用 服务器的公钥进行加密，发送给服务器
- 服务器获取到加密信息，使用自己的私钥进行解密，然后获取 第三个随机数，使用三个随机数和选定的 加密算法加密生成密钥，再使用这个密钥对 握手信息进行解密，解密成功表示服务器知道自己生成的密钥没有错误，然后再自己再使用密钥加密一段信息发送给客户端
- 客户端收到后使用密钥解密，解密成功表示服务器和自己的密钥是一致的，那么 SSL 握手成功



## 5、发送 HTTP 请求、接收响应、渲染页面

TCP 和 SSL 连接成功后，封装 HTTP 发送报文，使用 密钥加密，然后交给 TCP，TCP 再交给 IP 发送给服务器

服务器收到后封装一个 HTTP 响应报文，将请求的 html 资源放到 响应体中，按照相同的规则返回给浏览器

浏览器接受到后进行渲染