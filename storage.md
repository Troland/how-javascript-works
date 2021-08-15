# 存储引擎及如何选择合适的存储 API

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-storage-engines-how-to-choose-the-proper-storage-api-da50879ef576)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第十六章。**

## 概述

每当设计网页程序的时候，为本地设备选择合适的存储机制尤为重要。一个好的存储引擎可以帮助开发人员有效地存储数据，减少传输带宽及提高程序的响应速度。正确的存储缓存策略是构建移动端离线网页体验的核心组成部分，越来越多的用户默认以为可以离线使用移动端网页程序。

本章，我们将讨论各种可用的存储 API 和服务，并提供一些在构建网页程序时如何正确地选择存储引擎的建议。

## 数据模型

数据存储模型决定了其内部是如何组织存储数据的。这会影响到整个网页程序的设计，并计算出为获取高性能网页程序及解决其所遇到的问题所需要的开销。没有所谓更好的技术和一刀切的解决方案，因为所有的问题都是工程学相关的问题。那么，让我们来瞧瞧可供选择的数据模型吧：

* 结构型：以预定义字段将数据存储于表中，因其为典型的基于 SQL 的数据库管理系统，所以可以很好地适应灵活和动态的数据查询。IndexedDB 是一个浏览器端结构型数据库的显著例子。
* 键/值型：键/值数据存储及关系型 NoSQL 数据库，允许开发者通过唯一键索引来存储和获取非结构型数据(即非预定数据类型的字段的数据)。键/值数据存储就像哈希表存储，意及其允许在固定时间内访问索引的模糊数据类型的数据。键/值数据型存储的很好的例子有浏览器端的 Cache API 和 服务器端 Apache Cassandra。
* 字节流型：这一简单的模型把数据存储为定长，模糊字符串的字节变量，让应用层来控制其内部数据组织。该模型尤其适合于文件存储系统和其它层次型组织的大型数据。字节流存储的典型例子包括文件系统和云存储服务。

## 持久性

网页程序的数据存储方法可以以数据的存储时长来进行分析：

* 会话持久性：仅当单个网页会话或者活动的浏览器选项卡时数据有效，关闭即失效。会话持久性数据存储一个例子即 [Session Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage)。
* 设备持久性：该类数据存储于特殊设备的跨会话和浏览器选项卡/窗口有效。设备持久性存储机制的一个例子即 [Cache API](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage)。
* 全局持久性：该类数据跨会话和设备存储。因此，它是兼容性最好的数据持久性方案。它不会存储于设备上，这意味着需要从服务端存储中获得数据。因为这里只讨论针对设备的数据存储，所以这里只是稍微提下服务端数据存储。

## 客户端数据持久性

现如今，有相当多的浏览器 API 可供选择用于存储数据。这里将详细讨论这些方法，然后对其进行比较以便让开发者轻松地选择正确的数据存储方案。

然而，首先在选择如何存储数据之前，开发者需要考虑几件事情 。当然了，第一件事即必须想清楚打算如何使用网页程序及之后的维护和性能优化。即使胸有成竹，可代选择的方案可能只有几个。以下为开发者需要考虑的问题：

* 浏览器支持-优先考虑标准化和组织良好的 API，因为这些 API 不会轻易变动且兼容性好。这些 API 同样有非常丰富的文档和活跃的开发者社区。
* 事务-有时候，事务对于相关的数据存储操作集原子化成功或失败至关重要。传统数据库使用事务模型来实现该功能，在事务模型中把相关数据更新划分为任意的单元。
* 同步/异步-少数存储 API 是同步的意即存储或者检索数据请求会阻塞当前活跃线程直到数据请求结束。使用同步数据存储 API 会阻塞主线程且会让程序界面假死。尽量使用异步存储 API。

## 对比

这里，让我们浏览一下网页开发者当前可用的 API 并使用上述的几个维度来进行比较。

<iframe src="https://airtable.com/embed/shrIBGJdUMiYCKvH9/tbl7Z3CqbW4dD7Dtn"></iframe>

## 文件系统 API

![0_9kpehy4mub8f-hsp](https://user-images.githubusercontent.com/1475173/50570382-f0b9c100-0dc3-11e9-9f15-b38abc429c4e.png)

有了 FileSystem API，网页程序就可以在用户本地文件系统的沙箱中进行新建，读取，操作和写文件。

该接口包含如下几个部分：

* 读取和操作文件：`File/Blob`，`FileList`，`FileReader`
* 新建和写文件：`Blob()`，`FileWriter`
* 访问目录和文件系统：`DirectoryReader`，`FileEntry/DirectoryEntry`，`LocalFileSystem`

文件系统 API 并不是标准的 API.因其兼容性不太好，所以切记不要在生产环境中使用。各种浏览器厂商的实现会有很大的不同且该 API 以后可能会变更。

文件和目录条目 API 接口文件系统用来表示一个文件系统。可从任意文件系统条目的 [filesystem](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemEntry/filesystem) 属性中获取这些对象。少数浏览器提供了额外的 API 来创建和操作文件系统。

该接口不会允许开发者访问用户的文件系统。相反，开发者会在浏览器沙箱内获得一个虚拟磁盘。若想要访问用户的文件系统，可以采取安装 Chrome 插件的方法。

## 获得文件系统

网页程序可调用 `window.requestFileSystem()` 来访问沙箱文件系统。：

```
// 注意: 该文件系统以 Google Chrome 12 为前缀The file system has been prefixed as of Google Chrome 12:
window.requestFileSystem = window.requestFileSystem || window.webkitRequestFileSystem;
window.requestFileSystem(type, size, successCallback, opt_errorCallback)
```

第一次调用 requestFileSystem() 方法的时候会新建一个本地存储。需要注意的是该文件系统是沙箱型的，意即网页程序不可以访问其它程序的文件。

在获得访问文件系统的权限后，开发者可以对文件和目录进行大部分标准文件系统操作。

和其它存储策略相比，FileSystem 大有不同，它旨在满足数据库不能很好地解决客户端存储的情况。一般来说，程序用来处理大型二进制 blobs 文件或者和浏览器上下文之外的程序分享数据。

以下为使用 FileSystem 的好范例：

* 断点续传工具-当选择文件或者文件目录来上传的时候，会把文件复制进本地沙箱后一次上传一个分片。
* 视频游戏，音乐或者其它含有大量媒体文件的程序
* 为了更好的性能提供离线访问或者本地缓存功能的音频/图片编辑器-数据 blobs 一般会非常庞大且需要可读写
* 离线视频播放器-其需要下载大型文件以备之后观看或者高效寻轨-流式传输
* 离线网页邮件客户端-下载附件且进行本地存储

以下为该 API 的当前浏览器支持情况：

![0_ndu4n8xqf6qeqmsy](https://user-images.githubusercontent.com/1475173/50570378-ef889400-0dc3-11e9-9959-c08458c5631d.png)

## Local storage

![0_asohzlowolitnuel](https://user-images.githubusercontent.com/1475173/50570381-f0212a80-0dc3-11e9-9cd1-c917f908af1c.png)

localStorage API 允许开发者访问 [文档](https://developer.mozilla.org/en-US/docs/Web/API/Document) 源的 [Storage](https://developer.mozilla.org/en-US/docs/Web/API/Storage) 对象。存储的数据在多个浏览器会话之间仍然有效。localStorage 和 [sessionStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window.sessionStorage) 类似，只不过存储在 localStorage 中的数据没有过期时间，而存储在 sessionStorage 中的数据会在页面会话结束时丢失-意即当关闭页面时即丢失。

无论是 localStorage 还是 sessionStorage 其数据只存储在特定的页面源中，页面源包含协议，主机名和端口。

以下为该 API 的当前浏览器支持情况：

![0_hxc_nupnycubhj-l](https://user-images.githubusercontent.com/1475173/50570379-f0212a80-0dc3-11e9-9c7c-9316a216e5b0.png)

## Session storage

![0_-imsnws_l1g0syla](https://user-images.githubusercontent.com/1475173/50570384-f0b9c100-0dc3-11e9-94f5-12c6dfc69719.png)

sessionStorage 属性允许开发者访问当前源的会话 [Storage](https://developer.mozilla.org/en-US/docs/Web/API/Storage) 对象。前面简述过，sessionStorage 和 [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage) 类似。唯一的区别即，存储在 localStorage 中的数据没有过期时间而 sessionStorage 中的数据会在页面会话结束时丢失。页面会话的时效为浏览器打开时且在页面重载和恢复时。在新的选项卡中打开新页面或者窗口会导致重新初始化一个新的会话，这与会话 cookies 的工作机制是不一样的。

请注意无论数据存储于 sessionStorage 还是 localStorage 都仅只在指定的页面源中有效。

以下为该 API 的当前浏览器支持情况：

![0_hxc_nupnycubhj-l](https://user-images.githubusercontent.com/1475173/50570379-f0212a80-0dc3-11e9-9c7c-9316a216e5b0.png)

## Cookies

![0_vkqiniyfu2o7d7bh](https://user-images.githubusercontent.com/1475173/50570385-f1525780-0dc3-11e9-9021-a89939840dea.png)

所谓 cookie(网页 cookie，浏览器 cookie) 指的是由用户的服务器发送到客户端的一小段数据。浏览器将其存储下来然后在下一次请求的时候捎带上发往同一服务器。一般而言，它被用来告知两个请求是否来自于同一个客户端-比如保持用户登录状态。它为 [无状态](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview#HTTP_is_stateless_but_not_sessionless) HTTP 协议记录有状态信息。

Cookies 有以下三个主要用途：

* 会话管理-登录，购物车，游戏积分或者其它需要在服务端记录的数据。
* 个性化-用户参数，皮肤和其它设置
* 监控-记录和分析用户行为

Cookies 曾经是通用的客户端存储方案。当它是客户端存储的唯一方案的时候，这是不二选择，现如今推荐选择使用现代存储 API 来存储客户端数据。每次发送请求都会捎带上 Cookies，所以会影响性能(特别是当在一个移动端请求数据的时候)。

有两种类型的 cookies：

* 会话 cookie-当用户关闭浏览器时失效。网页浏览器可以使用会话恢复技术来固化大多数会话 cookie，就好像未曾关闭浏览器一样。
* 永久性 cookie-和客户端关闭即过期相反，永久性 cookie 会在指定的过期时间过期或者在一个指定的时间段(Max-age)后过期。

请注意不要在 cookie 中存储凭据或者敏感信息，因其固有的不安全缺陷机制。

然而，不肖说，cookie 是兼容性最好的方案。

## Cache

![0_xz2u-ztabhwjosky](https://user-images.githubusercontent.com/1475173/50570386-f1525780-0dc3-11e9-9156-2440dedb14f3.png)

Cache 接口是缓存请求/响应对象的存储机制。请注意和 workers 一样可在窗口作用域内使用 Cache 接口。虽然 Cache 是在服务工作线程规范中定义的，但这并不表示一定要和服务工作线程一起使用。

一个源可以拥有多个命名的缓存对象。开发者只需要在脚本(比如在[服务工作线程中](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorker))中实现如何处理更新缓存即可。除非显式请求否则不会更新缓存中的对象，只能通过删除缓存对象，否则不会过期。使用 [CacheStorage.open()](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage/open) 来打开指定名称的缓存对象，然后调用任意的缓存方法来维护缓存。

开发者需要定时清除缓存条目。每个源在浏览器端都有限额的缓存数据。使用 [StorageEstimate](https://developer.mozilla.org/en-US/docs/Web/API/StorageEstimate) 来估算缓存配额使用率。浏览器尽力管理硬盘空间，但它有可能会删除指定源的缓存数据。浏览器可能会删除指定源的所有数据抑或不会。切记使用名称来对脚本进行版本管理且只操作可以安全操作的脚本版本。查看 [Deleting old caches](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorker_API/Using_Service_Workers#Deleting_old_caches) 以获取更多信息。

**CacheStorage** 接口表示 [Cache](https://developer.mozilla.org/en-US/docs/Web/API/Cache) 对象存储。

接口：

* 提供一个可以为 [ServiceWorker](https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorker)，其它类型工作线程或者 [window](https://developer.mozilla.org/en-US/docs/Web/API/Window) 作用域可访问的所有命名的缓存的主目录(虽然是在 [服务工作线程](https://w3c.github.io/ServiceWorker/) 中定义的缓存，但是并不意味着只能将其和工作线程配合使用)
* 维护一份字符串名称和 [Cache](https://developer.mozilla.org/en-US/docs/Web/API/Cache) 对象的映射

使用 [CacheStorage.open()](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage/open) 来获取 [Cache](https://developer.mozilla.org/en-US/docs/Web/API/Cache) 实例。

使用 [CacheStorage.match()](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage/match) 来检查指定的 [Request](https://developer.mozilla.org/en-US/docs/Web/API/Request) 是否是 CacheStorage 对象追踪的 [Cache](https://developer.mozilla.org/en-US/docs/Web/API/Cache) 对象的键。

使用通过全局 [caches](https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/caches) 属性来访问 CacheStorage。

## IndexedDB

![0_hp66xm7oe9u8ofk1](https://user-images.githubusercontent.com/1475173/50570387-f1525780-0dc3-11e9-810e-9769ffc85783.png)

IndexedDB 是一种客户端持久性数据存储方案。因其允许开发者创建拥有富查询能力的网页程序而不用关心网络情况，这些网页程序可以线上或者离线运行。

IndexedDB 适用于大量的数据存储(比如，外借图书馆 DVD 目录)和不需要保持网络连通的网页程序(比如，邮件客户端，待办事项及便笺)。

因大家都比较熟悉其它存储 API ，本文将对 IndexedDB 多唠会嗑。另外，如今随着网页程序越来越复杂 IndexedDB 也正变得越来越流行。

## IndexedDB 原理

IndexedDB 允许开发者使用键来存储和获取一个对象。所有对数据库的操作均发生于事务之中。和其它网页存储方案一样，IndexedDB 遵循[同源策略](http://www.w3.org/Security/wiki/Same_Origin_Policy)。因此，不能够跨域访问数据，只能访问同一个域名下的存储数据。

可以在包括 [网页服务线程](https://developer.mozilla.org/En/DOM/Using_web_workers) 的大多数上下文上使用该异步 API。IndexedDB 曾经也有 [synchronous](https://developer.mozilla.org/en/IndexedDB#Synchronous_API) 版本，应用于网页线程中，但是由于社区对此并不感冒所以被从规范中删除了。

IndexedDB 曾经也有一个被称为 WebSQL 数据库的竞品规范，但是已经被 W3C 所弃用。虽然 IndexedDB 和 WebSQL 均是存储方案，但是它们功能并不一样。WebSQL 数据库是一个关系型数据库访问系统，而 IndexedDB 只是一个索引表系统。

不要以其它类型数据库为蓝本，想当然地使用 IndexedDB。相反，需要仔细阅读文档。以下为开发者所需要了解的核心概念：

* **IndexedDB 数据库存储键-值对**-值可以是复杂的结构型对象而键可以是这些对象的属性。开发者可以使用对象的任意属性来创建索引进行快速搜索，比如枚举排序。键也可以是二进制型对象。
* **IndexedDB 是建立在事务型数据模型之上的**-IndexedDB 中的所有操作都发生于[事务](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Basic_Concepts_Behind_IndexedDB#gloss_transaction)上下文之中。因此，开发者不可以在事务之外执行命令或者打开游标。同样地，事务只能自动而不可以手动提交。
* **大多数 IndexedDB 都是异步的 **-API 不会通过返回值地形式来返回数据。相反，需要传入回调函数来处理返回值。意即，开发者不是同步把值存储进数据库或者直接从数据库中取回值。相反，发起 request 请求即表示一次数据库操作。当数据处理结束会通知开发者，开发者所监听的事件类型会通知数据操作是否成功。这和 [XMLHttpRequest](https://developer.mozilla.org/en/DOM/XMLHttpRequest) (或者其它这么多 JavaScript 相关的东西) 的工作原理大同小异。
* **IndexedDB 使用大量的请求**-请求是对象用来接收之前提到的成功或者失败事件。它们包含 onsuccess 和 onerror 属性，和 readyState，result，errorCode 等用来告知请求状态的属性一样。
* **IndexedDB 是面向对象的**-IndexedDB 并不是一个含有表示行列集合的表关系型数据库。这一巨大的差异影响开发者设计和构建网页程序。
* **IndexedDB 不使用结构型查询语言(SQL)**-它在索引上使用查询后会创建一个游标，可以使用该游标来遍历结果集。若不熟悉 NoSQL 系统，可以阅读 [维基百科关于 NoSQL 的文章](https://en.wikipedia.org/wiki/NoSQL)。
* **IndexedDB 也应用了同源策略**-一个源即包含域名，应用程序层协议及 URL 端口地址的文档，脚本即在源中执行。每个源都拥有其关联的数据库集。每个数据库在源中都有唯一的标识。

## **IndexedDB** 局限性

IndexedDB 被设计用来满足大多数的客户端存储情况的。然而，它并没有被设计用来处理如下情况：

* **国际化排序**-并不是所有的语言以同样的方式排列字符串，因此国际化排序是不支持的。虽然数据库并不能够以指定的国际化顺序来存储数据，开发者可以读取数据库中数据然后自行排列数据。
* **同步**- API 并不是用来和服务端数据进行同步的。开发者必须自己写代码来把客户端 indexedDB 数据库和服务端数据库进行同步。
* 全文检索-该 API 中没有和 SQL 中的 LIKE 类似的运算符。

另外，需要注意的是浏览器会在以下情况清除数据库：

* **用户发起清除操作的请求**-许多浏览器都允许用户清除指定网站的数据，包括 cookie, 书签，存储的密码以及 IndexedDB 数据。
* **浏览器在隐私模式下**-一些浏览器含有『隐私浏览』(Firefox) 和 『无痕浏览』(Chrome)模式。会话结束会清除数据库。
* **超出了磁盘容量或者磁盘限额。**
* **数据损坏。。。**

虽然现实情况和浏览器能力日新月异，但是浏览器产商都朝着尽一切可能保存数据的方向努力。

![0_kgdqye70_z58d7na](https://user-images.githubusercontent.com/1475173/50570765-b9064580-0dd2-11e9-9cf7-1d92173c9f01.png)

## 选择合适的存储 API

正如之前所说的那样，最好尽可能采用兼容性好的 API 且提供异步调用模型来最大限度地提升 UI 响应速度。这些标准自然而然会产生如下技术选择：

* 使用 [Cache API](https://developers.google.com/web/fundamentals/instant-and-offline/web-storage/cache-api) 来操作离线存储。该 API 在支持 [服务工作线程](https://jakearchibald.github.io/isserviceworkerready/) 功能的浏览器中可用。Cache API 非常适用于排列和一个已知 URL 相关联的资源。
* 使用 IndexedDB 来存储程序状态和用户生成的内容。和只支持 Cache API 的浏览器相比，这使得用户可以在更多的浏览器中离线使用程序。

## 参考

* <https://developers.google.com/web/fundamentals/instant-and-offline/web-storage/>
* <https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies>
* <https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API/Basic_Concepts_Behind_IndexedDB>