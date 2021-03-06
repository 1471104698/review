# HTTP



## 1、HTTP 状态码 和 报文格式

```
1xx 表示提示信息
【100 Continue】用于 post 请求两次传输，浏览器第一次将 post 请求头发送给服务器，服务器根据 post 请求头部的信息（content-length、Authenticate 等字段）进行验证和判断是否能够进行处理，如果可以，那么响应 100 让浏览器发送请求体

2xx 表示请求成功
【200 OK】表示服务器请求处理成功，可能会存在数据，数据存储在 响应体中
【201 Created】表示需要创建的资源已经创建完毕，资源的 URL 在响应头的 Location 字段中
【202 Accepted】表示服务器接收到请求，但是还没有处理，可能后面处理，也可能不会处理
【204 Not Content】服务器请求处理成功，没有数据返回给浏览器
【206 xxx】用于断点续传，表示还没有数据发送完成

3xx 跟资源相关
【301 Moved Permanently】永久重定向，表示资源的位置永久发生改变，在响应头的 Location 字段存储了新的 URL，浏览器会 cache 该 URL
【302 Found / Moved Temporary】临时重定向，表示资源的位置暂时发生改变，在响应头的 Location 字段存储了新的 URL，浏览器不会 cache 该 URL，此次访问新的 URL，下次访问还是旧的 URL
【304 not modified】资源没有更新，浏览器的 cache 数据仍然可以使用（在输入 url 访问的全过程中会提到）

可以看出，10x 20x 30x 的常见状态码都不存在 x03 这个状态码

4xx 客户端请求错误
【401 Unauthorized】用户未认证，需要先认证
【403 Forbidden】用户已经认证，但是没有权限访问请求的资源
【404 Not Found】用户请求的资源没有找到（不存在）



5xx 服务器错误
【502 Bad Gateway】网关/代理收到上游服务的无效响应（网关起到负载均衡、服务降级、熔断的作用，nginx 起到反向代理的作用）
【504 Gateway Time-out】网关超时，网关/代理服务器没有在超时时间内没有从上游服务器获取到响应，比如并发量大，请求堆积，那么后面的请求都会由于超时返回 504

上游服务和下游服务的区别：
【所有信息从上游流向下游】，即下游服务是调用上游服务的存在，即调用方是下游服务，被调用方是上游服务
上游服务不需要知道下游服务的存在，下游服务需要知道上游服务的存在，然后可选的进行调用
如果正在查看请求，那么客户端是上游，服务端是下游
如果正在查看响应，那么服务端是上游，客户端是下游

HTTP 请求报文的格式（响应报文类似）：
1）请求行：请求的 url 和 http 版本号，请求的方式
2）请求头
3）空行：用来分割请求头 和 请求体的
4）请求体


HTTP 请求方式 
1）GET：用于请求静态资源（图片、html），请求的参数存储在 url 上，请求体中不会存储数据
2）POST：请求服务器创建资源，需要创建的资源存储在请求体中
3）PUT：用于修改指定位置的资源
4）DELETE：用于删除指定位置的资源
5）HEAD：该请求跟 GET 类似，区别在于响应体不存在任何数据，主要是用来资源检查，比如检查缓存是否过期了

GET 和 POST 的区别：
1）GET 是用来请求静态资源，请求参数存储在 URL 上，HTTP 协议没有限制 URL 的大小，但是浏览器会进行限制；POST 是用来创建资源，请求参数存储在 请求体中，HTTP 协议没有限制请求体的大小，但是服务器（tomcat）会进行限制
2）GET 是幂等的，无论多少次请求都不会改变服务器资源，POST 是非幂等的，多次请求就会创建多个资源
3）GET 请求一次发送（当然 HTTP 协议没有规定），POST 有时候能够分为两次发送（关联 100 状态码）

为什么 POST 要分为两次发送？
当 POST 的请求体参数过大时，有时候服务器无法进行处理，那么将整个 POST 发送过去，服务器又不能进行处理，那么显然这次请求是无意义的，但是请求体的数据又会占据网络带宽。所以为了避免这种情况的发生，浏览器会先发送请求头给服务器，服务器经过验证后如果自己能够处理，那么再响应 100 状态码让浏览器发送请求体。
这么做的好处是在一定程度上能够减少网络带宽的占用
缺点是需要分为两次发送，需要占用更多的时间
```



## 2、HTTP 各个版本的演变逻辑

```java
1、HTTP 0.9
它只有 GET 请求，同时只能简单的处理 html 字符串
（可以说这时候只有一个简单的请求行和请求体，请求行没有多余的 HTTP 版本 和 HTTP 请求方式，同时也没有设置状态码和请求头）

2、HTTP 1.0
1）添加了 POST、PUT 等请求方式
2）能够处理 图片、音频 等文件
3）增加了请求头/响应头、状态码等

HTTP 1.0 的缺点：
使用短连接，每一个请求都需要重新建立一个 TCP 连接，效率较低


3、HTTP 1.1
1）长连接：添加了长连接的连接方式，默认使用长连接，当一个 HTTP 请求和响应完成后，不会立马断开连接。 
2）管道：添加管道，一个 TCP 连接能够同时发送多个 HTTP 请求（管道是基于长连接的）
3）断点续传：跟 206 状态码配合使用
4）分块传输 Chunk

HTTP 1.1 的缺点：
随着基于长连接的管道，存在 HTTP 队头阻塞的问题（短连接没有这个问题，长连接才有）：能够同时发送多个 HTTP 请求，但是必须按照顺序进行响应
在一条 TCP 连接上，先后发送了 请求 1 和 请求 2，服务器即使先处理好请求 2，但是也不能直接响应请求 2，只能等请求 1 处理并响应完才能去响应请求 2，因为这个 TCP 连接没有任何措施去标识响应对应的是哪个请求的，如果先将 响应 2 发送出去，那么可能会被当成是 请求 1 的响应



4、HTTP 2.0
1）代替文本格式，使用新的二进制格式
2）多路复用
3）头部压缩
4）服务推送

二进制格式：
HTTP 2.0 的报文全部修改为二进制形式的，而不再使用 HTTP 1.x 的文本形式
使用二进制格式主要是应该为了多路复用

多路复用：
HTTP 2.0 使用多路复用的方式来解决 HTTP 1.1 的 HTTP 队头阻塞的问题：
	它将同个域名上的同个用户的所有请求共用一条 TCP 连接，它定义了 3 个新的概念：流（stream）、消息（message）、帧（frame），一个完整的 HTTP 请求/响应 的过程为 stream，一个 HTTP 请求 或者 响应为 message，每个消息会被分为多个 frame 进行传输，而在每个 frame 上会存在一个 stream id 标识该 frame 属于哪个 stream。 这么做的好处在于，它相比 HTTP 1.1 来说实现了一种机制，能够标识响应是属于哪个请求的，这样就不需要按照 请求的顺序来进行响应了，解决了 HTTP 队头阻塞的问题。
	多路复用要求同一个 stream 的 frame 必须按序传输，即 FIFO
这种多路复用存在另一个问题：TCP 队头阻塞：
	由于每个 HTTP 请求/响应 都是使用的一个 TCP 连接，意味着它们的数据都是共用同一个 TCP 发送缓冲区的，这样一旦存在某个 stream 的某个 frame 丢失的话，那么就会导致滑动窗口无法右移，导致所有 stream 的数据在可发送窗口为 0 并且在丢包重传完成之前都无法发送出去，出现 TCP 队头阻塞。当出现这种情况时，HTTP 2.0 的效率比 HTTP 1.1 还差，因为 HTTP 1.1 可以通过多开几个 TCP 连接来尽可能避免这种问题。
	
头部压缩：
在 HTTP 报文中，大部分的字段都是重复的，并且基本都不会发生改变，因此 HTTP 2.0 约定发送方和接收方各自维护一份相同的静态字典和动态字段。
静态字典用来存储 method:GET、set-cookie 这种基本不会发生改变的字段，在字典中它们都各自对应一个索引编号，在 HTTP 报文中直接使用这个编号来代替这个字段，接收方接收到后根据编号比对自己的静态字典来获取对应的字段值。这样可以节省很多的内存
动态字段跟静态字典差不多，主要是维护一些动态变化的数据，比如请求的 url、cookie 值之类的

服务推送:
在 HTTP 1.x 中，浏览器请求一个 html 页面，服务器只会返回 html 页面，浏览器解析后会发起多个请求来获取 html 页面中需要的 js、css、图片等，效率来说相对较低
在 HTTP 2.0 中，服务器会主动推送资源，当浏览器请求一个 html 页面时，服务器在响应 html 时认为浏览器还可能需要对应的 js、css、图片等资源，因此会在响应中顺便带上这些资源，减少浏览器的请求


HTTP 2.0 的缺点：
1）使用的多路复用存在 TCP 阻塞问题，一旦丢包重传那么会阻塞其他的 stream，效率比 HTTP 1.1 还低
2）HTTP 2.0 协议没有要求使用 SSL 协议，但是主流浏览器都强制要求 HTTP 2.0 使用 SSL 协议，那么就是需要 TCP 3 次握手 + SSL 四次握手，如果把 TCP 第三次握手和 SSL 第一次握手合并，那么也需要 6 次握手，那么单单为了连接就需要 3 RTT 的时间（物理距离越大，那么 RTT 越长）
    

5、HTTP 3.0

HTTP 3.0 的特性：
1）使用 UDP + DH 算法 来代替 TCP + SSL，实现 1 RTT 连接
2）使用 HTTP 2.0 的多路复用
3）应用层面实现 TCP 的滑动窗口机制、超时重传机制、流量控制机制、拥塞控制机制
4）使用 Packet number 实现 UDP 层面的序列号机制
5）使用 前向纠错 减少丢包重传的几率

HTTP 3.0 目前讲的是 QUIC 协议，它使用 UDP + DH 算法 来代替 TCP + SSL，实现 1 RTT 连接
使用 UDP 的目的是为了减少 TCP 握手、挥手带来的时间消耗以及 TCP 队头阻塞问题
同时为了保证可靠性传输，它在应用层面自己实现了类似 TCP 的可靠性算法：滑动窗口、超时重传、流量控制、拥塞控制（慢启动、拥塞避免、快重传、快恢复），这种在应用层面实现的不依赖于操作系统和内核，可以根据用户的网络情况制定不同的拥塞控制算法，更加的有效

虽然 HTTP 3.0 使用的是 UDP，但是它也实现了多路复用、滑动窗口和丢包重传，那么为什么不会出现 TCP/UDP 队头阻塞 这种问题？
我们知道 HTTP 2.0 的 TCP 队头阻塞是 滑动窗口 导致的问题
而 HTTP 3.0 虽然使用了滑动窗口，但是它本身也对这种机制做了一些改造，它定义了一个 Packet number，每个包都有一个 Packet number，这个 Packet number 可以当作是 UDP 层面的序列号机制
并且它规定了丢包重传的新包的 Packet number 要比丢失的旧包的 Packet number 要大，即丢包重传的包当作一个新的包来传输，比如 Packet N 丢失了，那么它重传的包应该是 Packet N + M，而不再是 Packet N。
这意味着即使发生丢包，滑动窗口也不会阻塞在丢包的位置，而会继续右移。

将丢失的包当作一个新的包不会导致同一个 stream 上的 Packet 的数据顺序发生改变吗?
在 HTTP 2.0 中，为什么要求同一个 stream 中需要按序传输？
这里我们需要先讲下 HTTP 和 TCP 之间的关系： TCP 保证的是将 HTTP 顺序交给它的数据进行编号，这个编号就叫做序列号，然后在 TCP 接收方根据这个序列号进行拼接，保证了顺序，因此滑动窗口当丢包重传时，需要按照原来的序列号进行重传，避免乱序，因此会才导致丢包重传时窗口阻塞。
这意味着 HTTP 按照顺序发送 1 2 4 3 的数据包给 TCP，TCP 本身是不知道这些数据包是乱序的，在它眼里，1 2 4 3 就是顺序，它的责任就是按照 1 2 4 3 的顺序发送，并且按照 1 2 4 3 的顺序进行拼接。
因此在 HTTP 层面必须保证数据是有序的，否则 TCP 会乱序发送，这也正是 HTTP 2.0 中 stream 的 frame 必须按序发送的原因，正是因为它没有跟 TCP 的序列号机制一样的机制，所以它才需要按序发送，所以它才会导致窗口阻塞
而在 HTTP 3.0，它给每个 Packet 标识了所在 stream 的 offset，这个 offset 就相当于 HTTP 层面的序列号机制，即使乱序发送，那么 HTTP 层面的接收方也可以通过这个 offset 正确拼接，因此在丢包重传时可以将包当作心的包发送，因为我们不需要在 TCP 层面保证顺序，而是在 HTTP 层面保证顺序。


前向纠错的作用：
比如一段数据被切成 10 个 Packet 包，对 10 个包进行异或运算，运算结果会作为 FEC 包连同 10 个包数据一起传输过去，如果在传输过程中除了 FEC 包外存在其他的一个包丢失了，那么可以根据 FEC 包 和 其他 9 个包来推算出丢失包的数据，从而减少丢包重传的几率
虽然说多发了一个包，但是这个代价比丢包重传（重传需要时间，并且后续的数据也会进行阻塞）要小得多。
异或原理：a^b = c，a^c = b，
因此 p1^p2^p3^p4^p5^p6^p7^p8^p9^p10 = FEC
那么如果丢失了一个包，可以通过 p1^p2^p3^p4^p5^p6^p7^p8^p9^FEC = p10 来计算出丢失的包的数据
但是仅仅只能用在丢失一个包的情况，如果丢失了 两个及两个以上的包，那么就只能重传了
```





## 3、HTTPS

```java
HTTPS 是在 应用层 和 传输层之间加的一层 SSL/TCL 协议，可以说是在表示层的协议

由于单纯的使用 AES 对称加密 和 RSA 非对称加密存在中间人攻击问题，所以需要出现了 CA 证书

CA 证书是 CA 机构颁发的，CA 证书的生成过程：
1）服务器生成一对非对称密钥，然后将 公钥、服务器域名、持有人 等信息发送给 CA 机构
2）CA 机构自己有一对非对称密钥，它使用一个散列算法对 这些信息 进行 hash 计算生成 信息摘要，然后使用自己的 CA 私钥对信息摘要进行 RSA 非对称加密
3）CA 机构将 信息的明文、散列算法、加密后的信息摘要 封装起来作为一个 CA 证书，颁发给服务器

客户端如何验证 CA 证书？
CA 公钥是内置在计算机上的，相当于每个计算机都有世界公认的 CA 机构的 CA 公钥，当客户端拿到服务器的 CA 证书后，它通过 CA 证书上的 CA 机构从自己的计算机中获取对应的 CA 公钥，然后解密 被加密后的信息摘要，然后将服务器信息明文使用 CA 证书上的散列算法进行计算，然后跟解密后的信息摘要进行比对，如果一致，那么表示服务器可信任。
    
CA 证书如何防止篡改？
CA 证书可以被拦截，但是不能被篡改：
1）因为中间人拦截了 CA 证书后，如果它修改了服务器信息明文，那么客户端验证的时候发现两个信息摘要不一致，那么不会信任；
2）如果它修改了服务器明文，同时使用自己的 CA 公钥解密信息摘要，然后替换掉信息摘要，但是它没有 CA 私钥，所以无法正确加密，这样客户端解密失败，同样无法信任

SSL 四次握手过程：
1）第一次握手：客户端携带一个随机数（第一个随机数）和自己支持的对称加密算法
2）第二次握手：服务器收到信息后，保存随机数，然后自己初始化一个随机数（第二个），将随机数、CA 证书 和 选定的加密算法发送给客户端
3）第三次握手：客户端收到 CA 证书，保存随机数，然后再初始化一个随机数（第三个），利用服务器选定的加密算法和三个随机数生成一个对称密钥，然后使用对称密钥加密一段握手信息，然后使用 CA 证书上的 服务器公钥加密握手信息 和 第三个随机数，发送给服务器
4）第四次握手：服务器收到后，使用 私钥进行解密，然后使用三个随机数 和 加密算法生成对称密钥，然后使用对称密钥解密 握手信息，解密成功表示跟客户端生成对称密钥一样，然后使用这个对称密钥加密一段握手信息，发送给客户端
客户端收到后解密成功，表示服务器跟自己的对称密钥一致，那么可以开始通信。


在 TCP 握手的第三次握手中，客户端可以同时发起 SSL 第一次握手，相当于 TCP + SSL 压缩为 6 次握手，减少了一次
```



## 4、HTTP 1.1 分块传输 Chunk

对于客户端请求静态文件，服务端只需要在响应头指定 `Content-Length` 即可告知客户端需要接收的数据量大小

但是对于动态生成的文件，比如文件边生成边传输，生成的文件到底多大服务端也并不知道，因此服务端并不能在发送前就事先告知客户端的大小，那么 Content-Length 就不起作用，因此出现了分块传输机制 Chunk，使用 `Transfer-Encoding: chunked` 来代替 `Content-Length`

文件传输分成一个个的 chunk 块，chunk 编码格式如下：

```
[chunk size][\r\n][chunk data][\r\n][chunk size][\r\n][chunk data][\r\n][chunk size = 0][\r\n][\r\n]
```

chunk size 表示后续数据块的大小，chunk data 表示数据，当数据传输完成时，以 0 作为结尾，即 chunk size = 0

这里 chunk size 以十六进制表示



chunk 编码响应例子：

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

25
This is the data in the first chunk

1C
and this is the second one

3
con

8
sequence

0


```

25 是十六进制，转变成十进制为 37，而 `This is the data in the first chunk` 占了 35byte，那么最后存在 \r\n 两个字节表示换行

3 是十六进制，转变成十进制为 3，`con` 占 3 字节，所以不需要换行

0 表示传输结束，后面存在两个 \r\n

转变成字符串存储的话就是：

```
"This is the data in the first chunk\r\n"
"and this is the second one\r\n"
"con"
"sequence"
```

解码数据为：

```
This is the data in the first chunk
and this is the second one
consequence
```





## 5、HTTP 缓存

### 1、cache-control

HTTP 1.0 使用 Expires 来控制缓存，HTTP 1.1 使用 cache-control 用来控制浏览器是否进行缓存以及对应的缓存策略

cache-control 存在以下几种取值：

- public：浏览器和代理服务器等都进行缓存
- private：浏览器进行缓存，代理服务器不进行缓存（比如 CDN 服务器就不能缓存），其他缓存策略在没有指定 public 或者 private 的时候默认使用 private

- no-cache：进行缓存，缓存是否使用需要经过 `对比缓存` 来判断
- no-store：不进行缓存
- max-age=600：指定缓存有效时间，以 s 为单位，这里是 600s 后过期



### 2、Expires 和 max-age

当同时存在 Expires 和 max-age 时，**max-age 优先于 max-age**

max-age 存储的是一个相对的时间，比如 `cache-control：max-age=600`，它的过期时间是通过响应头字段 Date 来进行计算的

Expires 存储的是一个绝对时间，比如 `expires：Thu, 08 Sep 2022 19:03:54 GMT`



### 3、缓存类型

缓存分为两种类型：强制缓存和协商缓存



####  1、强制缓存

- 如果浏览器有缓存
  - 缓存没有过期，那么直接使用该缓存
  - 缓存过期了，那么带上缓存标识去询问服务器资源是否更新
    - 如果没有更新那么收到 304 状态码，浏览器继续使用该缓存
    - 如果更新了那么收到 200 状态码以及新资源，浏览器更新缓存。

- 如果没有缓存，那么请求服务器获取资源，收到 200 状态码以及资源。

强制缓是 cache-control ：max-age=xxx（xxx > 0） 的情况



#### 2、协商缓存

协商缓存就是无论缓存是否过期都会去咨询服务器

- 如果浏览器有缓存（不存在过期的说法）发请求带上缓存标识询问服务器是否资源是否已经更新
  - 如果没有更新那么收到 304 状态码，浏览器继续使用该缓存
  - 如果更新了那么收到 200 状态码 以及新资源，浏览器更新缓存
- 如果没有缓存，那么直接请求服务器，收到 200 状态码以及资源

协商缓存是 cache-control ：no-cache 或者  cache-control ：max-age = 0 的情况

协商缓存无论是否存在缓存都需要访问服务器，感觉这个缓存没有起到什么作用，实际上并不是，no-cache 的设置是比如一些资源经常发生变动，浏览器缓存的数据可能随时都会变成旧数据，因此浏览器需要每次都去询问服务器，而如果资源没有发生变动，那么服务器不需要返回这个资源，减少网络带宽，这也是此时缓存的作用



### 4、使用缓存时选择 cache-control 取值

资源是否需要缓存

- 不需要，选择 no-store
- 需要，再判断资源是否会经常发生变动
  - 会，选择 no-cache
  - 不会，再判断是否允许中间代理服务器缓存
    - 允许：选择 public，再指定 max-age
    - 不允许：忽略（不管，cache-control 默认值就是 private）或者选择 private，再指定 max-age

![image.png](https://pic.leetcode-cn.com/1631370130-BEmYuX-image.png)



### 5、缓存标识

[图解 no-cache 等，顺带有 etag 等的用法](https://zhuanlan.zhihu.com/p/55623075)



缓存存在两种类型的标识，用来协助服务器判断资源是否发生修改

- last-modified 和 if-modified-since
- etag 和 if-none-math

last-modified：用来记录服务器返回给客户端的资源当时的最新一次修改的时间

etag ：作为资源的特定标识，根据资源的一些数值进行计算得到的



当客户端第一次请求某个资源的时候，服务器会将资源以及资源最新修改的时间 last-modified 以及 根据 etag 发送给客户端，客户端将资源缓存，并且将 last-modified 和 etag 保存下来，当客户端下次由于某种情况（缓存过期或者no-cache）需要询问服务器时，将保存的 last-modified 的值赋值给 if-modified-since，将 etag 的值赋值给 if-none-math 字段，服务器根据这两个字段值进行比对判断资源是否发生修改，如果没有那么返回 304，如果修改了那么返回 200 和 新资源



etag 的计算是保证了即使一个数据进行了修改，它的修改时间发生了改变，但是它如果修改后数据跟修改前一样，那么它的 etag 也是一样的，比如字符串资源就是字符串长度+字符串 hash 值的计算，这样的话相同的字符串最终得到的 etag 必定是一致的



为什么有了 last-modified 还需要 etag ？

```
因为 last-modified 可能存在该问题：last-modified 记录的是时间，精确到秒级的，如果资源在 1s 内发生了修改，在 no-cache 的情况下，服务器收到 if-modified-since 也无法察觉，因为比对的时间也是一致的，而 etag 则是资源只要发生修改变得不一致，那么就能够识别出来
```



为什么不直接使用 etag？

```
个人认为是 etag 每次都需要进行计算然后进行比对，比较耗费 CPU 资源，所以可以的话借助 last-modified 来减少这种情况
```



etag 和 last-modified 同时存在时的优先级？

```
同时存在的话优先对比 etag，如果 etag 不同，那么 last-modified 肯定不同，如果 etag 相同，即使发生了修改，修改时间不同，但是资源内容没有发生改变，那么无法返回新资源，返回 304
如果 etag 不存在，那么再判断 last-modified
```



### 6、缓存强制过期校验 must-revalidate   

[Cache-Control: must-revalidate](https://zhuanlan.zhihu.com/p/60357719)



cache-control 取值还存在另外一个值：must-revalidate 

must-revalidate 字面意思是强制校验

有以下 cache-control 取值：

```
Cache-Control: max-age=86400, must-revalidate   
```

写这条缓存的人想表达的意思是：在 86400（1天）内无论缓存是否过期都必须想服务器发起请求校验资源是否发生修改

但是实际上这条配置并不会跟它表达的意思一样，它在缓存失效前都是直接从本地读取，并不会进行校验

这是因为 **must-revalidate 生效有个前提：缓存必须已经过期了。**

这就有疑问了，根据一般的想法，缓存过期了浏览器不都会自动去询问服务器的么，这个配置还有什么作用？那是因为 HTTP 规范允许客户端在一些特殊情况下使用过期缓存而不向服务器询问，而这个配置就是用来解决这个问题的，强制要求客户端不使用过期缓存，缓存过期了必须向服务器进行询问资源是否更新。



实际上配置的人想用的是 no-cache 。



那 must-revalidate  一定能强制浏览器不使用过期吗？

```
并不是，因为浏览器的页面回退前进功能，它会默认使用缓存的数据，即使缓存过期了也不会进行校验，使用 no-cache 和 must-revalidate 都不能使浏览器在该情况下不使用过期缓存，除非使用 no-store，这种情况下浏览器本地都没有缓存
```



### 7、启发式缓存

想让浏览器进行缓存，有三种方式：

- HTTP 1.1 的 cache-control：max-age
- HTTP 1.0 的 expires
- last-modified

max-age 优先级高于 expires，那第三种呢？

```
HTTP/2 200
Date: Wed, 27 Mar 2019 22:00:00 GMT
Last-Modified: Wed, 27 Mar 2019 12:00:00 GMT
```

假设存在上面的完整的响应头信息，没有 cache-control，也没有 expires，但它实际上也可以使用缓存，在 chrome 浏览器中添加了这么一个功能：使用 `(Date - Last-Modified0) / 10` 就是缓存的过期时间，即当前时间到资源的最后一次修改时间的十分之一

如果想禁用这个启发式缓存，那么添加 cache-control：no-cache 或者 cache-control：must-revalidate
