### 浏览器 Event Loop

简单的分三步

1. task (取一个最老的 task)

2. microtask (清空当前 microtask 栈, 当前 tick 产生的 microtask 也插入到当前的 list 里面)

3. render (不一定每个 loop 都会执行, 主要看执行时间, requestAnimationFrame 在这个阶段执行)

   requestAnimationFrame -> style -> layout -> paint

定时器合并

定时器宏任务可能会跳过渲染, 执行回调的时间比较短, 而且没有微任务的话, 同时到期的定时器任务可能会一起执行

### NodeJS Event Loop

nodejs 事件循环是由 Libuv 处理

#### 事件队列

由 libux 处理的事件队列:

- timers (setTimeout, setInterval)
- I/O
- setImmediate
- close event

还有 2 个由 nodejs 处理的中间队列

- NextTicks (process.nextTick)
- 微任务队列 (Promise)

每次循环先清空一个 libux 事件队列, 然后回到 nodejs 清空 nexttick 队列和 promise 队列, 然后再回到 libux 下一个事件队列 (按照上面的顺序)

BlueBird 和 Q 等 promise 库 早于原生的 promise, 内部实现方式不同, BlueBird 使用 setImmediate 实现, Q 是用 nexttick 实现, 那些执行顺序和原生的也不同

node V11 之后的版本向浏览器行为看齐, nexttick 和 promise 队列放在每个宏任务之后执行, 而不是每个宏任务队列完成之后

### 常见的内存泄露

1. 意外的全局变量

   ```js
   function foo1() {
     bar = 'some text';
   }

   function foo() {
     this.value = '意外的全局变量';
   }
   foo(); // new 缺失, this 在非严格模式下是windows
   ```

2. 遗忘的计时器或回调
   `setInterval`
3. 闭包
   ```js
   let theThing = null;
   function replaceTheThing() {
     const oldTheThing = theThing;
     function unused() {
       if (oldTheThing) {
         console.log('hi');
       }
     }
     theThing = {
       longStr: new Array(1000000).join('*'),
       someMethod: function () {
         console.log('message');
       },
     };
   }
   setInterval(replaceThing, 1000);
   ```
4. 被移除的 DOM 引用

   ```js
   var elements = {
     button: document.getElementById('button'),
     image: document.getElementById('image'),
   };
   function doStuff() {
     elements.image.src = 'http://example.com/image_name.png';
   }
   function removeImage() {
     // The image is a direct child of the body element.
     document.body.removeChild(document.getElementById('image'));
     // At this point, we still have a reference to #button in the
     //global elements object. In other words, the button element is
     //still in memory and cannot be collected by the GC.
   }
   ```

### HTTPS

SSL/TLS 协议, TLS 是 SSL 后续协议名称.

需要经过 4 次握手 (前面还有 TCP 的 3 次握手, 一共 7 次)

1. 客户端发出请求 (报告自己支持的 SSL/TLS 协议版本, 加密套件列别, 算法列表等的)

2. 服务端回应(自己的证书, 协议版本等等)

   证书包含 服务器公钥, 服务器信息, 然后对这个信息使用 Hash 算法得到消息摘要然后用 CA 私钥加密这个消息摘要得到数字签名

3. 客户端验证证书

   同样的 Hash 算法得到消息摘要, 然后用浏览器预设的 CA 公钥解密数字签名得到摘要对比, 这样就确定这个公钥是服务器的. 然后用这个公钥对一个随机数加密 (用于生成之后通信的对称密钥)

4. 服务器回应 (用私钥解密随机数 获取对称密钥)

### HTTP/2

为什么不是 HTTP/1.2, HTTP/2 引入了一个二进制分帧层，它定义了如何封装 HTTP 消息并在客户端与服务器之间传输, HTTP 语义不受影响但是传输方式变了, 并不能和 HTTP/1.x 兼容.(编码方式变了, HTTP/1.x 协议以换行符作为纯文本的分隔符，而 HTTP/2 将所有传输的信息分割为更小的消息和帧，并采用二进制格式对它们编码)

1. 请求与响应复用

   在 HTTP/1.x 中, HTTP 虽然可以通过 keep-alive 复用 TCP 连接, 但是一个 TCP 连接上的 HTTP 请求只能是串行请求的, 客户端只能通过多个 TCP 连接发起并行请求提升性能, 但是浏览器限制了同一个域同时最多只能发起 6 个 TCP 请求, 这样出现了域分片的优化手段, 就是通过不同域名子域去请求资源, 但是额外 DNS 查询和 TCP 慢启动还是会损害一些性能.

   在 HTTP/2 中客户端向一个域名请求过程, 只会创建一条 TCP 连接, 通过多路复用 ,分解 HTTP 消息, 然后重新组装 HTTP 请求都通过一个连接并行的请求和响应. 基于此, 原来针对 HTTP/1.x 合并请求资源减少 HTTP 请求的优化没必要了, 反而会影响缓存.

2. 数据流优先级

   将 HTTP 消息分解为独立的帧后, HTTP/2 标准允许为每个数据流设置权重, 这样服务器可以将高优先级以最优方式传给客户端

3. 服务器推送

   服务器除了对最初的请求响应之外, 服务器还可以向客户端推送额外的资源, 无需客户端明确请求, 比如 html 请求, 服务器分析知道客户端接下来会请求里面的 js,css 等文件, 就可以一起返回给客户端, 这个功能帮助客户端将资源放进缓存以备将来之需, 当然客户端也可以通过发送 `RST_STREAM`帧拒绝.

4. 头部压缩

   在 HTTP/1.x 中, 头部数据是以纯文本形式,通常开销在 500-800 字节, 而且大多的 HTTP 头部数据都是相同的, 为了减少开销, HTTP/2 搞了个 HPACK 压缩头部元数据, 就是把报文中字段名和值变成索引值, 这样就有 了索引表, 常用的 HTTP 标头放在静态表, 静态表由规范定义, 例如`method: GET`对应成索引表中 index 为 2 的一个值, 然后头部不确定的如 user-agent 等维护在一个动态索引表, 动态的添加.

### React
