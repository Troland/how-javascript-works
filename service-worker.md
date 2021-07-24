# Service Workers，生命周期及其使用场景

> 原文请查阅[这里](https://blog.sessionstack.com/how-javascript-works-service-workers-their-life-cycle-and-use-cases-52b19ad98b58)，略有删减，本文采用[知识共享署名 4.0 国际许可协议](http://creativecommons.org/licenses/by/4.0/)共享，BY [Troland](https://github.com/Troland)。

**这是 JavaScript 工作原理的第八章。**

![](https://user-images.githubusercontent.com/1475173/40543994-3bf0fff8-6059-11e8-8179-9c5ad4ab38c9.jpeg)

或许你已经了解到[渐进式网络应用](https://developers.google.com/web/progressive-web-apps/)将只会越来越流行，因为它旨在创造拥有更加流畅的用户体验的网络应用和创建类原生 App 的应用体验而非浏览器端那样的外观和体验。

构建渐进式网络应用的主要需求之一即在各种网络和数据加载的条件下仍然可用－它可以在网络不稳定或者没有网络的情况下使用。

本文我们将会深入了解 Service Workers：它们是如何工作的以及应该注意的地方。最后，我们将会列出一些Service Workers 可供利用的，独有的优势并且分享我们在 [SessionStack](https://www.sessionstack.com/) 中的团队实践经验。

## 概述

若想理解 Service Workers 相关的一切，你首先应该阅读一下之前发布的有关 [Web Workers](https://github.com/Troland/how-javascript-works/blob/master/worker.md) 的文章。

大体上，Service Worker 是一种 Web Worker，更准确地说，它更像是一个 [Shared Worker](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker)。

* Service Worker 运行在其全局脚本上下文中
* 不和特定网页相关联
* 不能够访问 DOM

Service Worker 接口之所以让人感到兴奋的原因之一即它支持网络应用离线运行，这使得开发者能够完全控制网络应用的行为。

## 生命周期

Service Worker 的生命周期和网页完全不相关。它由以下几个步骤组成：

* 下载
* 安装
* 激活

## 下载

这发生于浏览器下载包含 Service Worker 代码的 `.js` 文件。

## 安装

为了在网络应用中使用 Service Worker，首先得在 JavaScript 代码中对其进行注册。当 Service Worker 注册的时候，它会让浏览器在后台开始安装 Service Worker 的步骤。

通过注册 Service Worker，浏览器知晓包含 Service Worker 相关代码的 JavaScript 文件。看下如下代码：

```
if ('serviceWorker' in navigator) {
  window.addEventListener('load', function() {
    navigator.serviceWorker.register('/sw.js').then(function(registration) {
      // 注册成功
      console.log('ServiceWorker registration successful');
    }, function(err) {
      // 注册失败
      console.log('ServiceWorker registration failed: ', err);
    });
  });
}
```

以上代码首先检查当前执行环境是否支持 Service Worker API。如果是，则注册  `/sw.js` Service Worker。

可以在每次页面加载的时候，任意调用 `register()`－浏览器会检测 `service worker` 是否已经注册从而进行适当地处理。

`register()` 方法里面需要特别注意的地方即 Service Worker 文件地址。当前示例是在服务器根目录下。意即 service worker 会作用于整个源地址上。换句话说，即 service worker 会接收到该域名下所有页面 的 `fetch` 事件。如果注册 service worker 的文件路径是 `/example/sw.js` ，那么 service worker 会接收到所有页面路径以 `/example/` 为开头的 URL 地址的 `fetch` 事件（比如 `/example/page1/` `/example/page2/`）。

在安装阶段，最好加载和缓存一些静态资源。一旦静态资源缓存成功，Service Worker 的安装也就完成了。倘若加载失败－Service Worker 将会重试。一旦安装成功，静态资源也就缓存成功了。

这也就回答了为什么要在 load 事件之后注册 Service Worker。这不是必须的，但是强烈推荐这么做。

为什么要这样做呢？假设用户第一次访问网络应用。现在还没有注册 service worker，而且浏览器无法事先知晓是否会最终安装它。如果进行安装，则浏览器将需要为增加的线程开辟额外的 CPU 和内存，否则浏览器将会渲染网页。

**参考下[这里](https://javascript.info/onload-ondomcontentloaded)，`load ` 事件会加载完所有的资源比如图片，样式之后触发。**

最终的结果即是如果在页面中安装 Service Worker，将有可能导致页面延迟加载和渲染－不能够让用户尽快地访问网页。

需要注意的是这只会发生在第一次访问页面的时候。后续的页面访问不会被 Service Worker 的安装所影响。一旦在首次访问页面的时候激活了 Service Worker ，它就可以处理后续的页面访问所触发的页面加载/缓存事件。这是正确的，Service Worker 需要准备好以处理有限的网络连接活动。

## 激活

安装完之后下一步即激活。该步骤是操作之前缓存资源的绝佳时机。

一旦激活，Service Worker 就可以开始控制在其作用域内的所有页面。一个有趣的事实即：注册了 Service Worker 的页面直到再次加载的时候才会被 Service Worker 控制。当 Service Worker 开始进行控制，它有以下几种状态：

* 处理来自页面的网络或者消息请求所触发的 fetch 及 message 事件
* 中止以节约内存

以下即其生命周期：

![](https://user-images.githubusercontent.com/1475173/40543992-3b818826-6059-11e8-991a-1b801f01e8b9.png)

## 处理 Service Worker 内部的安装过程

在页面运行注册 Service Worker 的过程中，让我们来看看 Service Worker 脚本中发生的事情，它通过为 Service Workder 实例添加事件句柄来处理 Service Worker  实例的 `install` 事件。

以下为处理 `install` 事件所需要执行的步骤：

* 打开缓存
* 缓存文件
* 确认是否所有的静态资源已缓存

以下为一个 Service Worker 内部可能的简单安装代码：

```
var CACHE_NAME = 'my-web-app-cache';
var urlsToCache = [
  '/',
  '/styles/main.css',
  '/scripts/app.js',
  '/scripts/lib.js'
];

self.addEventListener('install', function(event) {
  // event.waitUntil 使用 promise 来获得安装时长及安装是否失败
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) {
        console.log('Opened cache');
        return cache.addAll(urlsToCache);
      })
  );
});
```

如果文件都成功缓存，则 service worker 安装成功。如果任意文件下载失败，那么 service worker 将会安装失败。因此，请注意需要缓存的文件。

处理 `install` 事件完全是可选，当不进行处理的时候，跳过以上几个步骤即可。

## 缓存运行时请求

该部分才是干货。在这里可以看到如何拦截请求然后返回已创建的缓存（以及创建新的缓存）。

当 Service Worker 安装完成之后，用户会导航到另一个页面或者刷新当前页面，Service Worker 将会收到 fetch 事件。这里有一个演示了如何返回缓存的静态资源或执行一个新的请求并缓存返回结果的过程的示例：

```
self.addEventListener('fetch', function(event) {
  event.respondWith(
    // 该方法查询请求然后返回 Service Worker 创建的任何缓存数据。
    caches.match(event.request)
      .then(function(response) {
        // 若有缓存，则返回
        if (response) {
          return response;
        }

        // 复制请求。请求是一个流且只能被使用一次。因为之前已经通过缓存使用过一次了，所以为了在浏览器中使用 fetch，需要复制下该请求。
        var fetchRequest = event.request.clone();
        
        // 没有找到缓存。所以我们需要执行 fetch 以发起请求并返回请求数据。
        return fetch(fetchRequest).then(
          function(response) {
            // 检测返回数据是否有效
            if(!response || response.status !== 200 || response.type !== 'basic') {
              return response;
            }

            // 复制返回数据，因为它也是流。因为我们想要浏览器和缓存一样使用返回数据，所以必须复制它。这样就有两个流
            var responseToCache = response.clone();

            caches.open(CACHE_NAME)
              .then(function(cache) {
                // 把请求添加到缓存中以备之后的查询用
                cache.put(event.request, responseToCache);
              });

            return response;
          }
        );
      })
    );
});
```

大概的流程如下：

* `event.respondWith()` 会决定如何响应 `fetch` 事件。 `caches.match()` 查询请求及查找之前创建的缓存中是否有任意缓存结果并返回 promise。
* 如果有，则返回该缓存数据。
* 否则，执行 `fetch` 。
* 检查返回的状态码是否是 `200`。同时检查响应类型是否为 **basic**，即检查请求是否同域。当前场景不缓存第三方资源的请求。
* 把返回数据添加到缓存中。

因为请求和响应都是流而流数据只能被使用一次，所以必须进行复制。而且由于缓存和浏览器都需要使用它们，所以必须进行复制。

## 更新 Service Worker

当用户访问网络应用的时候，浏览器会在后台试图重新下载包含 Service Worker 代码的 `.js` 文件。

如果下载下来的文件和当前的 Service Worker 代码文件有一丁点儿不同，浏览器会认为文件发生了改变并且会创建一个新的 Service Worker。

创建新的 Service Worker 的过程将会启动，然后触发 `install` 事件。然而，这时候，旧的 Service Worker 仍然控制着网络应用的页面意即新的 Service Worker 将会处于 `waiting` 状态。

一旦关闭网络应用当前打开的页面，旧的 Service Worker 将会被浏览器杀死而新安装的 Service Worker 就可以上位了。这时候将会触发 `activate` 事件。

为什么所有这一切是必须的呢？这是为了避免在不同选项卡中同时运行不同版本的的网络应用所造成的问题－一些在网页中实际存在的问题且有可能会产生新的 bug（比如当在浏览器中本地存储数据的时候却拥有不同的数据库结构）。

## 从缓存中删除数据

`activate` 回调中最为常见的步骤即缓存管理。因为若想删除安装步骤中老旧的缓存，而这又会导致旧 Service Workers 无法获取该缓存中的文件数据，所以，这时候就需要进行缓存管理。

这里有一个示例演示如何把未在白名单中的缓存删除（该情况下，以 `page-1` 或者 `page-2` 来进行命名）：

```
self.addEventListener('activate', function(event) {

  var cacheWhitelist = ['page-1', 'page-2'];

  event.waitUntil(
    // 获得缓存中所有键
    caches.keys().then(function(cacheNames) {
      return Promise.all(
        // 遍历所有的缓存文件
        cacheNames.map(function(cacheName) {
          // 若缓存文件不在白名单中，删除之
          if (cacheWhitelist.indexOf(cacheName) === -1) {
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});
```

## HTTPS 要求

当处于开发阶段的时候，可以通过 localhost 来使用 Service Workers ，但当处于发布环境的时候，必须部署好 HTTPS（这也是使用 HTTPS 的最后一个原因了）。

可以利用 Service Worker劫持网络连接和伪造响应数据。如果不使用 HTTPS，网络应用会容易遭受[中间人](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) 攻击。

为了保证安全，必须在支持 HTTPS  的页面上注册 Service Workers，这样就可以保证浏览器接收到的 Service Worker 没有在传输过程中被篡改。

## 浏览器支持

Service Workers 拥有良好的浏览器兼容性。

![](https://user-images.githubusercontent.com/1475173/40270197-7267d41e-5bbb-11e8-9bd3-978cab346084.png)

你可以追踪所有浏览器的支持进程：

<https://jakearchibald.github.io/isserviceworkerready/>

## Service Workers 提供了更多的功能的可能

Service Worker 独有的功能：

* 推送通知－允许用户选择及时接收网络应用的推送更新
* 后台同步－允许延迟操作直到网络连接稳定之后。这样就可以保证用户切实发送数据。
* 定期同步（以后支持）－提供了管理进行定期后台数据同步的功能
* **Geofencing** （以后支持）－可以自定义参数，也即 **geofences** ，该参数包含着用户所感兴趣的地区。当设备穿过某片地理围栏的时候会收到通知，这就能够让你基于用户的地理位置来提供有用的用户体验。

这里提到的每个功能将会该系列之后的文章中进行详细阐述。

我们持续不断地工作以让 SessionStack 的交互体验尽可能流畅，优化页面加载时间和响应时间。

当在 [SessionStack](https://www.sessionstack.com/) 上重放用户会话或者实时流播放，SessionStack 界面会从服务器持续抓取数据从而为用户创造一个类缓冲的使用体验（类似视频缓冲那种）。再详细了解一些原理即一旦在网络应用中集成 SessionStack 库，它将会持续收集诸如 DOM 变化，用户交互，网络请求，未处理异常以及调试信息的数据。

当重放或者实时观看一个会话的时候，SessionStack 会返回所有数据以方便观察发生于用户浏览器的所有事件。（视觉上和技术上的）。所有的这一切都是即时发生的，因为我们不想让用户等待。

由于数据是由前端抓取的，这个时候就可以使用 Service Workers 来处理类似播放器重载和再次流传输所有数据的情形。处理缓慢的网络连接也是非常重要的。

**参见维基百科关于流的[定义](https://zh.wikipedia.org/wiki/%E5%8D%B3%E6%99%82%E4%B8%B2%E6%B5%81%E5%8D%94%E5%AE%9A)，可以更好地理解这里的流的概念。**