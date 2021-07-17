# 深入理解 WebSockets 和带有 SSE 机制的HTTP/2 以及正确的使用姿势

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-deep-dive-into-websockets-and-http-2-with-sse-how-to-pick-the-right-path-584e6b8e3bf7)，略有改动，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是  JavaScript 工作原理的第五章。**

现在，我们将会深入通信协议的世界，绘制并讨论它们的特点和内部构造。我们将会给出一份 WebSockets 和 HTTP/2 的快速比较 。在文末，我们将会分享如何正确地选择网络协议的一些见解。

## 简介

现在，复杂的网页程序拥有丰富的功能，动态交互能力被认为是理所当然的事情。而这并不令人感到惊讶－因为自互联网诞生以来，它已走过一段漫长的时间。

起初，互联网并不是用来支持如此动态和复杂的网页程序的。它本来设想是由大量的 HTML 页面组成的，每个页面链接到其它的页面，这样就形成了包含信息的网页的概念。一切都是极大地围绕着所谓的 HTTP 请求/响应模式来建立的。客户端加载一个网页，直到用户点击页面并导航到下一个网页。

大约在 2005 年，引入了 AJAX，然后，很多人开始探索客户端和服务端双向通信的可能性。不变的是，所有的 HTTP 链接均是由客户端控制的，意即必须由用户交互或者定期轮询以从服务器加载数据。

## 让 HTTP 支持双向通信

支持服务器主动向客户端推送数据的技术已经出现了好一段时间了。比如 "[Push](https://en.wikipedia.org/wiki/Push_technology)" 和 "[Comet](http://en.wikipedia.org/wiki/Comet_%28programming%29)" 技术。

长轮询是服务端主动向客户端发送数据的最常见用来创建这一错觉的 hack 之一。通过长轮询，客户端打开了一个到服务端的 HTTP 连接，服务端保持该连接直到返回响应数据。当服务端有新数据需要发送时，它会把新数据作为响应发送给客户端。

让我们看一下简单的长轮询代码片段：

```
(function poll(){
   setTimeout(function(){
      $.ajax({ 
        url: 'https://api.example.com/endpoint', 
        success: function(data) {
          // 处理 `data`
          // ...

          //递归调用下一个轮询
          poll();
        }, 
        dataType: 'json'
      });
  }, 10000);
})();
```

这是一个基础的自执行函数，第一次会自动运行。它每隔 10 秒钟异步请求服务器并且当每次发起对服务器的异步请求之后，会在回调函数里面再次调用 `ajax` 函数。

其它技术涉及到 [Flash](http://help.adobe.com/en_US/FlashPlatform/reference/actionscript/3/flash/net/Socket.html) 和 XHR 多方请求以及所谓的 [htmlfiles](http://cometdaily.com/2007/12/27/a-standards-based-approach-to-comet-communication-with-rest/)。

所有这些方案都有一个共同的问题：都带有 HTTP 开销，这样就会使得它们不适用于低延迟程序。试想一下浏览器中的第一人称射击游戏或者其它要求实时组件功能的在线游戏。

## WebSockets 的出现

[WebSocket](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) 规范定义了一个 API 用以在网页浏览器和服务器建立一个套接字连接。通俗地讲：在客户端和服务器保有一个永久连接，两边可以在任意时间开始发送数据。

![](https://user-images.githubusercontent.com/1475173/39884220-a86a093a-54bb-11e8-8cd4-fac1e27f4813.png)

客户端通过 WebSocket 握手的过程来创建 WebSocket 连接。在这一过程中，首先客户端向服务器发起一个常规的 HTTP 请求。请求中会包含一个 `Upgrade` 的请求头，通知服务器客户端想要建立一个 WebSocket 连接。

让我们看下如何在客户端创建 WebSocket 连接：

```
// 创建新的加密 网页套接字 连接
var socket = new WebSocket('ws://websocket.example.com');
```

>网页套接字地址使用了 `ws` 协议。`wss` 是一个等同于 `HTTPS` 的安全的网页套接字连接。

该协议开始启动打开到 websocket.example.com  的 WebSocket 连接的过程。

下面是初始化请求头的简化例子。

```
GET ws://websocket.example.com/ HTTP/1.1
Origin: http://example.com
Connection: Upgrade
Host: websocket.example.com
Upgrade: websocket
```

如果服务器支持网页套接字协议，它将会同意升级请求，然后通过在响应里面返回 `Upgrade` 头来进行通信。

让我们看下 Node.js 的实现：

```
// 我们将会使用 https://github.com/theturtle32/WebSocket-Node 来实现 WebSocket
var WebSocketServer = require('websocket').server;
var http = require('http');

var server = http.createServer(function(request, response) {
  // 处理 HTTP 请求
});
server.listen(1337, function() { });

// 创建服务器
wsServer = new WebSocketServer({
  httpServer: server
});

// WebSocket 服务器
wsServer.on('request', function(request) {
  var connection = request.accept(null, request.origin);

  // 这是最重要的回调，在这里处理所有用户返回的信息
  connection.on('message', function(message) {
      // 处理 WebSocket 信息
  });

  connection.on('close', function(connection) {
    // 关闭连接
  });
});
```

连接建立之后，服务器使用升级来作为回复：

```
HTTP/1.1 101 Switching Protocols
Date: Wed, 25 Oct 2017 10:07:34 GMT
Connection: Upgrade
Upgrade: WebSocket
```

一旦连接建立，会触发客户端网页套接字实例的 `open` 事件。

```
var socket = new WebSocket('ws://websocket.example.com');

// WebSocket 连接打开的时候，打印出 WebSocket 已连接的信息
socket.onopen = function(event) {
  console.log('WebSocket is connected.');
};
```

现在，握手结束了，最初的 HTTP 连接被替换为 WebSocket 连接，该连接底层使用同样的 TCP/IP 连接。现在两边都可以开始发送数据了。

通过 WebSocket，你可以随意发送数据而不用担心传统 HTTP 请求所带来的相关开销。数据是以消息的形式通过 WebSocket 进行传输的，每条信息是由一个或者多个帧组成，帧包含你所传输的数据(有效载荷)。为了保证当消息到达客户端的时候被正确地重新组装出来，每一帧都会前置关于有效载荷的 4-12 字节的数据。使用这种基于帧的信息系统可以帮助减少非有效载荷数据的传输，从而显著地减少延迟。

**注意：**这里需要注意的是只有当所有的消息帧都被接收到而且原始的信息有效载荷被重新组装的时候，客户端才会接收到新消息的通知。

## WebSocket 地址

前面我们简要地谈到 WebSockets 引进了一个新的地址协议。实际上，WebSocket 引进了两种新协议：`ws://` 和 `wss://`。

URL 地址含有协议指定的语法。WebSocket 地址特别之处在于，它不支持锚(`sample_anchor`)。

WebSocket 和 HTTP 风格的地址使用相同的地址规则。`ws` 是未加密且默认是 80 端口，而 `wss` 要求 TSL 加密且默认 443 端口。

## 帧协议

让我们深入了解下帧协议。这是 [RFC](https://tools.ietf.org/html/rfc6455#page-27) 提供的：

```
0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+
```

由于 WebSocket 版本是由 RFC 所规定的，所以每个包前面只有一个头部信息。然而，这个头部信息相当的复杂。这是其组成模块的说明：

* `fin`(1 位)：指示是否是组成信息的最后一帧。大多数时候，信息正好一帧所以该位通常有值。测试表明火狐的第二帧数据在 32K 之后。

* `rsv1`，`rsv2`，`rsv3`(每个一位)：必须是 0 除非使用协商扩展来定义非 0 值的含义。如果收到一个非 0 值且没有协商扩展来定义非零值的含义，接收端会中断连接。

* `opcode`(4 位)：帧表示的数据。目前可用的值：

  `0x00`：该帧接续前面一帧的有效载荷。

  `0x01`：该帧包含文本数据。

  `0x02`：该帧包含二进制数据。

  `0x08`：该帧中断连接。

  `0x09`：该帧是一个 ping。

  `0x0a`：该帧是一个pong。

  (正如你所看到的，有相当一部分值未被使用；它们是保留以备将来使用的)。

* `mask`(1 位)：指示该连接是否被遮罩。正其所表示的意义，每一条从客户端发往服务器的信息都必须被遮罩，然后如果信息未遮罩，根据规范会中断该连接。

* `payload_len`(7 位)：有效载荷的长度。WebSocket 帧有以下长度分类：

  0-125 表示有效载荷的长度。126 意味着接下来两个字节表示有效载荷长度，127 意味着接下来的 8 个字节表示有效载荷长度。所以有效载荷的长度大概有 7 位，16 位和 64 位这三个等级。

* `masking-key` (32 位)：所有从客户端发往服务器的帧都由帧内的一个 32 位值所遮罩。

* `payload`：一般情况下被遮罩的实际数据。其长度取决于 `payload_len` 的长度。

为什么 WebSocket 是基于帧而不是基于流的呢？我和你一样不明白，我也想多学点，如果你有任何想法，欢迎在下面的评论区添加评论和资源。另外，[HackerNews](https://news.ycombinator.com/item?id=3377406) 上面有关于这方面的讨论。

## 帧数据

正如之前提到的，数据可以被拆分为多个帧。第一帧所传输的数据里面含有一个操作码表示传输的数据类型。这是必须的，因为当规范起草的时候，JavaScript 并不能很好地支持二进制数据。`0x01` 表示 utf-8 编码的文本数据，`0x02` 表示二进制数据。大多数人在传输 JSON 数据的时候都会选择文本操作码。当你传输二进制数据的时候，它会以浏览器指定的 [Blob](https://developer.mozilla.org/en-US/docs/Web/API/Blob) 来表示。

通过 WebSocket 来传输数据的 API 是非常简单的：

```
var socket = new WebSocket('ws://websocket.example.com');
socket.onopen = function(event) {
  socket.send('Some message'); // 向服务器发送数据
};
```

当 WebSocket 正在接收数据的时候（客户端），会触发 `message` 事件。该事件会带有一个 `data` 属性，里面包含了消息的内容。

```
// 处理服务器返回的消息
socket.onmessage = function(event) {
  var message = event.data;
  console.log(message);
};
```

你可以很容易地利用 Chrome 开发者工具的网络选项卡来检查 WebSocket 连接中的每一帧的数据。

![](https://user-images.githubusercontent.com/1475173/39884275-e9287a6a-54bb-11e8-9386-b8ec0c4dc58c.png)

## 数据分片

有效载荷数据可以被分成多个独立的帧。接收端会缓冲这些帧直到 `fin` 位有值。所以你可以把字符串『Hello World』拆分为 11 个包，每个包由 6(头长度) + 1 字节组成。数据分片不能用来控制包。然而，规范想要你有能力去处理[交错](https://en.wikipedia.org/wiki/Interleaving_%28data%29)控制帧。这是为了预防 TCP 包无序到达客户端。

连接帧的大概逻辑如下：

* 接收第一帧
* 记住操作码
* 连接帧有效载荷直到 `fin` 位有值
* 断言每个包的操作码都为 0

数据分片的主要目的在于允许开始时传输不明大小的信息。通过数据分片，服务器可能需要设置一个合理的缓冲区大小，然后当缓冲区满，通过网络发送一个数据分片。数据分片的第二个用途即多路复用，逻辑通道上的大量数据占据整个输出通道是不合理的，所以利用多路复用技术把信息拆分成更小的数据分片以更好地共享输出通道。

## 心跳包

握手之后的任意时刻，客户端和服务器可以随意地 ping 对方。当接收到 ping 的时候，接收方必须尽快回复一个 pong。此即心跳包。你可以用它来确保客户端是否仍然保持连接。

ping 或者 pong 虽然只是一个普通帧，但却是一个控制帧。Ping 包含 `0x9` 操作码，而 Pong 包含 `0xA` 操作码。当你接收到 ping 的时候，返回一个和 ping 携带同样有效载荷数据的 pong(ping 和 pong 最大有效载荷长度都为 125)。你可能接收到一个 pong 而未曾发送一个 ping。忽略它如果有发生这样的情况。

心跳包非常有用。利用服务（比如负载均衡器）来中断空闲的连接。另外，接收端不可能知道服务端是否已经中断连接。只有在发送下一帧的时候，你才会意识到发生了错误。

## 错误处理

你可以通过监听 `error` 事件来处理错误。

像这样：

```
var socket = new WebSocket('ws://websocket.example.com');

// 处理错误
socket.onerror = function(error) {
  console.log('WebSocket Error: ' + error);
};
```

## 关闭连接

客户端或服务器可以发送一个包含 `0x8` 操作码的数据控制帧来关闭连接。当接收到控制帧的时候，另一个节点会返回一个关闭帧。之后第一个节点会关闭连接。关闭连接之后，之后接收的任何数据都会被遗弃。

这是初始化关闭客户端的 WebSocket 连接的代码：

```
// 如果连接打开着则关闭
if (socket.readyState === WebSocket.OPEN) {
    socket.close();
}
```

同样地，为了关闭连接后运行任意清理工作，你可以为 `close` 事件添加事件监听函数：

```
// 运行必要的清理工作
socket.onclose = function(event) {
  console.log('Disconnected from WebSocket.');
};
```

服务器不得不监听 `close` 事件以便在需要的时候处理：

```
connection.on('close', function(reasonCode, description) {
    // 关闭连接
});
```

## WebSockets 和 HTTP/2 对比

虽然 HTTP/2 提供了很多的功能，但是它并不能完全取代当前的 push/streaming 技术。

关于 HTTP/2 需要注意的最重要的事即它并不能完全取代 HTTP。动词，状态码以及大部分的头部信息都会保持和现在一样。HTTP/2 只是提升了线路上的数据传输效率。

现在，如果我们对比 WebSocket 和 HTTP/2，将会发现很多类似的地方：

![](https://user-images.githubusercontent.com/1475173/39884335-22403eb4-54bc-11e8-9ec2-3e89827c7b8c.png)

正如以上所显示的那样，HTTP/2 引进了 [Server Push](https://en.wikipedia.org/wiki/Push_technology?oldformat=true) 技术用来让服务器主动向客户端缓存发送数据。然而，它并不允许直接向客户端程序本身发送数据。服务端推送只能由浏览器处理而不能够在程序代码中进行处理，意即程序代码没有 API 可以用来获取这些事件的通知。

这时候服务端推送事件（SSE）就派上用场了。SSE 是这样的机制一旦客户端－服务器连接建立，它允许服务器异步推送数据给客户端。之后，每当服务器产生新数据分片的时候，就推送数据给客户端。这可以看成是单向的发布－订阅模型。它也提供了一个被称为 EventSource 的 标准 JavaScript 客户端 API，该 API 作为 [W3C](https://www.w3.org/TR/eventsource/) 组织发布的 HTML5 标准的一部分已经在大多数的现代浏览器中实现。注意不支持原生 [EventSource API](http://caniuse.com/#feat=eventsource) 的浏览器可以通过兼容插件实现。

由于 SSE 是基于 HTTP 的，所以它天然兼容于 HTTP/2 并且可以组合使用以利用各自的优势： HTTP/2 处理一个基于多路复用流的高效传输层而 SSE 为程序提供了 API 用来支持服务端推送。

为了完全理解流和多路复用技术，先让我们来了解一下 IETF 的定义：『流』即是在一个 HTTP/2 连接中，在客户端和服务端间进行交换传输的一个独立的双向帧序列。它的主要特点之一即单个的 *HTTP/2* 连接可以包含多个并发打开的流，在每一终端交错传输来自多个流的帧。

![](https://user-images.githubusercontent.com/1475173/39884369-3749d112-54bc-11e8-8dc0-e07733b74e56.png)

必须记住的是 SSE 是基于 HTTP 的。这意味着，通过使用 HTTP/2，不仅仅可以把多个 SSE 流交叉合并成单一的 TCP 连接，还可以把多个 SSE 流（服务端向客户端推送）和多个客户端请求（客户端到服务端）合并成单一的 TCP 连接。多亏了 HTTP/2 和 SSE，现在我们有了一个纯粹的 HTTP 双向连接，该连接带有一个简单的 API 允许程序代码注册监听服务端的数据推送。缺乏双向通信能力一直被认为是 SSE 对比 WebSocket 的主要缺点。多亏了 HTTP/2，这不再是缺点。这就让你有机会坚持使用基于 HTTP 的通信系统而非 WebSockets。

## WebSocket 和 HTTP/2 的使用场景

WebSockets 依然可以在 HTTP/2 + SSE 的统治下存在，主要是由于它曾经是被广泛使用的技术，在特殊情况下，和 HTTP/2 比较它有一个优点即它天生拥有更少的开销（比如，头部信息）的双向通信能力。

假设你想要构建一个大型的多人在线游戏，在各个连接终端会产生大量的信息。在这样的情况下，WebSockets 会表现得更加完美。

总之，当你需要在客户端和服务端建立一个真正的低延迟的，接近实时连接的时候使用 WebSockets。记住这可能要求你重新考虑如何构建服务器端程序，同时也需要你关注诸如事件队列的技术。

如果你的使用场景要求显示实时市场新闻，市场数据，聊天程序等等，HTTP/2 + SSE 将会为你提供一个高效的双向通信通道的同时可以得到 HTTP 的所有益处：

* 当考虑现有架构的兼容性的时候，WebSockets 经常会是一个痛点，因为升级 HTTP 连接到一个完全和 HTTP 不相关的协议。
* 可扩展性和安全：网络组件（防火墙，入侵检测，负载均衡器）的建立，维护和配置都是为 HTTP 而设计的，大型／重要的程序会更喜欢具有弹性，安全和可伸缩性的环境。

同样地，你不得不考虑浏览器兼容性。查看下 WebSocket 兼容情况：

![](https://user-images.githubusercontent.com/1475173/39884410-54d4a8b0-54bc-11e8-8369-9763917c8fa1.png)

兼容性还不错。

然而，HTTP/2 的情况就不太妙了：

![](https://user-images.githubusercontent.com/1475173/39884447-753df098-54bc-11e8-9964-ec78572da12e.png)

* 仅支持 TLS（还不算坏）
* 仅限于 Windows 10 的 IE 11 部分支持
* 仅支持 OSX 10.11+  Safari 浏览器
* 仅当你能够通过 ALPN 使用 HTTP/2 （服务器需要明确支持的东西）才会支持 HTTP/2

SSE 的支持情况要好些：

![](https://user-images.githubusercontent.com/1475173/39884473-8a06d404-54bc-11e8-8357-100e6be31db5.png)

仅 IE/Edge 不支持。（好吧，Opera Mini 即不支持 SSE 也不支持 WebSockets，因此我们把它完全排队在外）。有一些优雅的垫片来让 IE/Edge 支持 SSE。

## SessionStack 是如何选择的？

[SessionStack ](https://www.sessionstack.com/?utm_source=medium&utm_medium=blog&utm_content=Post-5-websockets-outro) 同时使用 WebSockets 和 HTTP，这取决于使用场景。

一旦整合 SessionStack 进网页程序，它会开始记录 DOM 变化，用户交互，JavaScript 异常，堆栈追踪，失败的网络请求以及调试信息，允许你用视频回放网页程序中的问题及发生在用户身上的一切事情。全部都是实时发生的并且要求对网页程序不会产生任何的性能影响。

这意味着你可以实时加入到用户会话，而用户仍然在浏览器中。这样的情况下，我们会选择使用 HTTP，因为这并不需要双向通信（服务端把数据传输到浏览器端）。当前情况下，使用 WebSocket 就是过度使用，难以维护和扩展。

然而，整合进网页程序的 SessionStack 库应用了 WebSocket（优先使用，否则回滚到 HTTP）。它会分批处理并且向我们的服务器发送数据，这是单向通信。在这种情况下，之所以选择 WebSocket 是因为计划中的某些产品功能可能需要进行双向通信。