# WebRTC 及点对点网络通信机制

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-webrtc-and-the-mechanics-of-peer-to-peer-connectivity-87cc56c1d0ab)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是  JavaScript 工作原理第十八章。**

## 概述

何为 WebRTC ？首先，字面上已经给出了关于这一技术的大量信息，RTC 即为实时通信技术。

WebRTC 填补了网页开发平台中的一个重要空白。在以往，只有诸如桌面聊天程序这样的 P2P 技术才能够实现实时通讯而网页不行。但是 WebRTC 的出现改变了这一状况。

WebRTC 主要允许网页程序创建点对点通信，我们将会在随后的章节中进行介绍。我们将讨论如下主题，以便向开发者全面介绍 WebRTC 的内部构造：

* 点对点通信
* 防火墙和 NAT 穿透
* 信号，会话及协议
* WebRTC 接口

## 点对点通信

每个用户的网页浏览器必须按照如下步骤以实现通过网页浏览器进行的点对点通信：

* 同意开始进行通信
* 知道如何定位另一个点
* 绕过安全和防火墙限制
* 实时传输所有多媒体通信信息

众所周知，基于浏览器的点对点通信的最大挑战之一即如何定位和建立与另一个网页浏览器进行通信的网络套接字以进行双向数据传输。我们将会克服建立与此种网络连接相关的困难。

每当网页程序需要数据或者静态资源，会直接从相应的服务器获取，仅此而已。但是，如果若想要通过直接连接用户的浏览器来建立点对点的视频聊天就不可能，因为其它浏览器并不是一个已知的网页服务器，所以用户不知道需要建立视频聊天的 IP 地址。所以，需要更多的技术以建立 p2p 连接。

## 防火墙和 NAT 穿透

一般电脑是不会被分配一个静态公共 IP 地址的。原因是电脑是位于防火墙和网络访问转换设备(NAT) 后面的。

一个 NAT 设备会把防火墙内的私有 IP 地址转换为一个公共 IP 地址。对于安全和有限的可用公共 IP 地址来说，NAT 设备是必须的。这也是为什么开发者的网页程序不能够把当前设备看成拥有一个静态公共 IP 地址的原因。

让我们来了解下 NAT 设备的工作原理。当开发者处于一个企业网中然后加入了 WIFI，那么电脑将会被分配一个只存在于 NAT 后面的 IP 地址。假设是 172.0.23.4。然而，对于外部而言，用户的 IP 地址会是类似 164.53.27.98 这样的。那么，外部会把所有请求看作来自 164.53.27.98 而 NAT 设备会保证来自于目标用户电脑的请求的响应数据返回到相应的内部 172.0.23.4  IP 地址的电脑。这得归功于映射表。注意到除了 IP 地址，网络通信还需要通信端口。

随着 NAT 设备参与其中，浏览器需要知道进行通信的目标浏览器对应的机器 IP 地址。

这个就需要用到 NAT 会话穿透程序(STUN)和 NAT 穿透中继转发服务器。为使用 WebRTC 技术，开发者需要请求 STUN 服务器以获得其公共 IP 地址。这就好像你的电脑请求远程服务器，询问远程服务器发起查询的客户端 IP 地址。远程服务器会返回对应的客户端 IP 地址。

假设这一过程进展顺利，那么开发者将会获得一个公共 IP 地址和端口，这样就可以告知其它点如何直接和你进行通信。同理，这些点也可以请求 STUN 或 TURN 服务器以获得公共 IP 地址然后告知其通信地址。

## 信号，会话和协议

前述网络信息检索过程只是更大的信号话题的一部分，在 WebRTC 中，它是基于 JavaScript 会话构建协议(JSEP)标准的。信号涉及网络检索和 NAT 穿透，会话创建及管理，通信安全，媒体功能元数据和调制及错误处理。

为了让通信顺利进行，节点必须确定元数据本地媒体环境(比如分辨率和编码能力等)和收集可用的程序主机网络地址。WebRTC 接口里面没有集成反复传输这一重要信息的信号机制。

WebRTC 标准并没有规定信号且没有在接口中实现是为了能够更加灵活地使用其它技术和协议。信号和处理信号的服务器是由 WebRTC 程序开发者控制的。

假设开发者基于浏览器的 WebRTC 程序使用之前所说的 STUN 服务器获取其公共 IP 地址，那么，下一步即和其它点进行协商和建立网络会话连接。

使用任意一个专门应用于多媒体通信的信号/通信协议初始化会话协商和通信连接。该协议负责管理会话和中断的规则。

会话初始协议(SIP) 是协议之一。多亏了 WebRTC 信号的灵活性，SIP 并不是唯一可供使用的信号协议。所选的通信协议必须和被称为会话描述协议(SDP)的应用层协议兼容，SDP 被应用于 WebRTC。所有的多媒体指定元数据都是通过 SDP 协议进行数据传输的。

任意试图和其它点进行通信的点(比如 WebRTC 程序)都会生成交互式连接建立协议(ICE)候选集。候选集表示一个可供使用的 IP 地址，端口及传输协议的集合。注意，一台电脑可以拥有多个网络接口(有线和无线等)，因此可以拥有多个 IP 地址，每个接口分配一个 IP 地址。

以下为 MDN 上描绘这一通信交换的图示：

![](https://user-images.githubusercontent.com/1475173/53612111-fe48c700-3c0b-11e9-95d0-53efb4a1480b.png)

## 建立连接

每个节点首先获取之前所说的公共 IP 地址。之后动态创建「信道」信号数据来检索其它节点并且支持点对点协商及创建会话。

这些「信道」不能够被外部检索和访问到且只能通过唯一标识符来访问。

需要注意的是由于 WebRTC 的灵活性且事实上信号创建程序并没有在标准中指定，使用的技术不同，「信道」的概念和使用会有些许异同。事实上，一些协议并不要求「通道」机制来进行通信。



本篇文章将会假设存在「信道」。



一旦两个或者更多的点连接到相同的「信道」上，节点就可以进行通信和协商会话信息。这一过程和[发布/订阅模式](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)有些许类似。大体上，初始点使用诸如会话初始协议(SIP)和 SDP 的信号协议发出一个「offer」的包。发起者等待连接到指定「通道」的接收者的「answer」应答。

一旦接收到应答，会开始选择和协商由各个节点生成的最优交互连接建立协调候选(ICE)集。一旦选定了最优 ICE 候选集，特别是确认了所有节点通信所要求的元数据，网络路由(IP 地址和端口)及媒体信息。这样就会完全建立及激活节点间的网络套接字会话。紧接着，每个节点创建本地数据流和数据通道端点，然后，最后使用任意双向通信技术来传输多媒体数据。

如果确认最优 ICE 候选的过程失败了，这样的情况经常发生于使用的防火墙和 NAT 技术，后备使用 TURN 服务器作为中继转发服务器。这一过程主要是使用一台服务器作为中间媒介，然后在节点间转发传输数据。请注意这不是真正的点对点通信，因为真正的点对点通信是节点之间直接进行双向数据传输。

每当使用 TURN 作为后备通信的时候，每个节点将不必知道如何连接并传输数据给对方节点。相反，节点只需要知道在会话通信期间实时发送和接收多媒体数据的公共 TURN 服务器。

需要重点理解的是这仅仅只是一个失败保护和最后手段。TURN 服务器需要相当健壮，拥有昂贵带宽和强大的处理能力及处理潜在的大量数据。因此，使用 TURN 服务器会明显增加额外的开销和复杂度。

## WebRTC 接口

WebRTC 中包含三种主要接口：

* **媒体捕捉和流－**允许开发者访问诸如麦克风和网络摄像机的输入设备。该接口允许开发者获取麦克风或者网络摄像机媒体流。
* **RTCPeerConnection－**开发者实时传输获取的视频和音频流到另一个 WebRTC  端点。开发者使用这些接口连接本地机器和远程节点。该接口提供创建到远程节点的连接，维护和监视连接及关闭不再需要的连接的方法。
* **RTCDataChannel－**该接口允许开发者传输任意数据。每个数据通道都和 RTCPeerConnection 有关。

我们将分别介绍这三类接口。

## 媒体捕捉和流

**媒体捕捉和流**接口经常被称为媒体流接口或者流接口，该接口支持音频或者视频数据流数据，处理音视频流的方法，与数据类型相关的约束，异步获取数据时的成功和错误回调及 API 调用过程中触发的事件。

[MediaDevices](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices) 的 **getUserMedia()** 方法提示用户授权允许使用媒体输入设备，创建一个包含指定媒体类型轨道的[媒体流](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream)。该媒体流，可包括诸如视频轨道(由诸如摄像机，视频录制设备，屏幕共享服务等硬件或者虚拟视频源所创建)，音频轨道(与视频类似，由诸如麦克风，A/D 转换器等的物理或者虚拟音频源所创建)且有可能是其它类型轨道。

该方法返回一个 [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) 并解析为 [MediaStream](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream) 对象。当用户拒绝授权或者没有可用的匹配媒体资源，promise 会分别返回 PermissionDeniedError 或者 NotFoundError。

可以通过 `navigator` 对象访问 MediaDevice 单例：

```
navigator.mediaDevices.getUserMedia(constraints)
.then(function(stream) {
 /* 使用流 */
})
.catch(function(err) {
 /* 处理错误 */
});
```

注意这里需要传入 `constraints` 对象以指定返回的媒体流类型。开发者可以进行各种配置，包括使用的摄像头(前置或后置)，帧频率，分辨率等等。

从版本 25 起，基于 Chromium 的浏览器已经允许通过 `getUserMedia()` 获取的音频数据赋值给音频或者视频元素(但需要注意的是默认媒体元素将会是静音的)。

可以把 `getUserMedia`作为[网页音频接口的输入节点](http://updates.html5rocks.com/2012/09/Live-Web-Audio-Input-Enabled)：

```
function gotStream(stream) {
    window.AudioContext = window.AudioContext || window.webkitAudioContext;
    var audioContext = new AudioContext();
    // 从流创建音频节点
    var mediaStreamSource = audioContext.createMediaStreamSource(stream);
    // 把它和目标节点进行连接让自己倾听或由其它节点处理
    mediaStreamSource.connect(audioContext.destination);
}

navigator.getUserMedia({audio:true}, gotStream);
```

## 隐私限制

由于该接口可能导致明显的隐私问题，规范在通知用户和权限管理方面对 `getUserMedia()` 方法有非常明确的规定。在打开诸如用户网页摄像头或者麦克风的媒体输入设备的时候`getUserMedia()` 必须总是获取用户的授权。

浏览器可能提供每个域名授权一次的权限功能，但必须至少第一次询问授权，然后若用户选择则必须特别授权进行中的权限。

通知中的规则同样重要。浏览器必须显示一个窗口显示使用中的摄像头或者麦克风，超过可能存在其它硬件指示器。即使当时设备没有进行录制，浏览器必须显示一个提示窗口提示已授权使用哪个设备作为输入设备。

## RTCPeerConnection

**RTCPeerConnection** 表示一个本地电脑和远程节点之间的 WebRTC 连接。它提供了连接远程节点，维护和监视连接及关闭不再活跃的连接的方法。

如下为一张 WebRTC 图表展示 了 RTCPeerConnection 的角色：

![](https://user-images.githubusercontent.com/1475173/53612109-fe48c700-3c0b-11e9-8066-92a0208cfc48.png)

从 JavaScript 方面看 ，图中需要理解的主要方面即 `RTCPeerConnection` 把复杂的底层内部结构的复杂度抽象为一个接口给开发者。WebRTC 所使用的编码和协议为即使在不稳定的网络环境下仍然能够创建一个尽可能实时的通信而做了大量的工作：

* 包丢失隐蔽
* 回音消除
* 带宽自适应
* 视频抖动缓冲
* 自动增益控制
* 降噪和减噪
* 图像「清洁」

## RTCDataChannel

不仅仅是音视频，WebRTC 还支持实时传输其它类型的数据。

`RTCDataChannel` 接口允许点对点交换任意数据。

该接口有许多用途，包括：

* 游戏
* 实时文本聊天
* 文件传输
* 去中心化网络

该接口有几项功能，充分利用 `RTCPeerConnection` 并创建强大和灵活的点对点通信：

* 使用`RTCPeerConnection` 创建会话
* 包含优先级的多个并发通道
* 可靠和不可靠消息传递语义
* 内置安全(DTLS)和消息堵塞控制

语法和已有的 WebSocket 类似，包含有 `send()` 方法和 `message` 事件：

```
var peerConnection = new webkitRTCPeerConnection(servers,
    {optional: [{RtpDataChannels: true}]}
);

peerConnection.ondatachannel = function(event) {
    receiveChannel = event.channel;
    receiveChannel.onmessage = function(event){
        document.querySelector("#receiver").innerHTML = event.data;
    };
};

sendChannel = peerConnection.createDataChannel("sendDataChannel", {reliable: false});

document.querySelector("button#send").onclick = function (){
    var data = document.querySelector("textarea#send").value;
    sendChannel.send(data);
};
```

由于通信是直接在浏览器之间进行的，所以 ，即使是使用中继转发服务器(TURN)，RTCDataChannel 会比 WebSocket 更快。

## WebRTC 实际应用

在实际应用中，WebRTC 需要服务器，但这很简单，因此会发生如下步骤：

* 用户各自检索节点然后交换诸如名字等详情。
* WebRTC 客户端程序(点)交换网络信息。
* 点交换诸如视频格式和分辨率的媒体信息。
* WebRTC 客户端程序穿透 [NAT 网关](http://en.wikipedia.org/wiki/NAT_traversal) 和防火墙。

换句话说，WebRTC 需要四种类型的服务端功能：

* 用户检索和通信
* 发信号
* NAT/防火墙穿透
* 中继转发服务器以防点对点通信失败

[ICE](http://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment) 使用 [STUN](http://en.wikipedia.org/wiki/STUN) 协议及其扩展 [TURN](http://en.wikipedia.org/wiki/Traversal_Using_Relay_NAT) 协议来创建 RTCPeerConnection 连接来处理 NAT 穿透和其它网络变化。

如前所述，ICE 是用来连接诸如两个视频聊天客户的节点协议。一开始，ICE 会试图使用最低的可能的网络延迟即使用 UDP 来直接连接节点。在这一过程中，STUN 服务器只有一个任务:让位于 NAT 之后的节点能够找到其公共地址和端口。开发者可以查看一下可用的 [STUN 服务器](https://gist.github.com/zziuni/3741933)(Google 也有一堆) 名单。

![](https://user-images.githubusercontent.com/1475173/53612107-fdb03080-3c0b-11e9-823a-e6d793127cee.png)

## 检索候选连接

若 UDP 失败，ICE 尝试 TCP，先 HTTP 后 HTTPS。如果直接连接失败-特殊情况下，由于企业 NAT 穿透和防火墙-ICE 使用中介(中继) TURN 服务器。换句话说，ICE 首先通过 UDP 使用 STUN 服务器来直接连接节点，若失败则后备使用 TURN 中继转发服务器。「检索连接候选者」指的是检索网络接口和端口的过程。

![](https://user-images.githubusercontent.com/1475173/53612108-fe48c700-3c0b-11e9-96d0-91317fbf2d68.png)

## 安全性

实时通信程序或者插件可能造成几种安全问题。例如：

* 未加密媒体或者数据有可能会在浏览器之间或者浏览器和服务器之间被窃取。
* 程序有可能在未经用户授权同意的情况下记录和分发音视频。
* 可疑软件或者病毒有可能会随着表面上无害的插件或者程序一起安装。

WebRTC 有几种方法用来解决如上问题：

* WebRTC 实现使用诸如 [DTLS](http://en.wikipedia.org/wiki/Datagram_Transport_Layer_Security) 和 [SRTP](http://en.wikipedia.org/wiki/Secure_Real-time_Transport_Protocol) 的安全协议。
* 包括信号机制在内的所有 WebRTC 组件都是强制加密的。
* WebRTC 不是一个插件：其组件运行于浏览器沙箱之中且不是在一个单独的进程之中，不需要单独安装组件且随着浏览器升级而更新。
* 摄像头和麦克风必须显式授权且当摄像头或者麦克风运行时，必须在用户窗口中清楚的显示。

对于需要实现一些浏览器之间实时通信流功能的产品而言，WebRTC 是一个令人难以置信和强大的技术。

参考资料：

- <https://www.html5rocks.com/en/tutorials/webrtc/basics/>
- <https://www.innoarchitech.com/what-is-webrtc-and-how-does-it-work/>