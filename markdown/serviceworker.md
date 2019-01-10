# 利用ServiceWorker实现页面的快速加载和离线访问

> Service workers 本质上充当Web应用程序与浏览器之间的代理服务器，也可以在网络可用时作为浏览器和网络间的代理。它们旨在（除其他之外）使得能够创建有效的离线体验，拦截网络请求并基于网络是否可用以及更新的资源是否驻留在服务器上来采取适当的动作。他们还允许访问推送通知和后台同步API。(引用自：[链接](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API))

简单的来说，ServiceWorker（后文简称sw）运行在页面后台，使用了sw的页面可以利用sw来拦截页面发出的请求，同时配合CacheAPI可以将请求缓存到客户本地

因此我们可以：

- 将页面的文件存储在客户端，下次打开页面可以不向服务器发出资源请求，极大的加快页面加载速度
- 离线打开页面的同时可以在sw发出请求，更新本地的资源文件
- 实现离线访问页面

但是也存在着一些问题

- 由于打开页面不再向服务器发出页面请求，因此当服务器上的页面有新版本的时候客户端无法及时升级
- sw存在一定的兼容性问题

![sw-compatible](https://raw.githubusercontent.com/violinux666/blog/master/asset/sw-compatible.png)

IE全面扑街，pc上兼容性不太好，移动端安卓支持良好，ios要12+。但考虑到sw并不会影响的页面的正常运行，所以项目上还是能投入生产的。

## 基本例子

### index.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Sw demo</title>

</head>
<body>
    
</body>
<script src="index.js"></script>
<script>
    if(navigator.serviceWorker){
        navigator.serviceWorker.register('sw.js').then(function(reg){
            if(reg.installing){
                console.log('client-installing');
            }else if(reg.active){
                console.log('client-active')
            }
        })
    }
</script>
</html>
```

### index.js
```js
document.body.innerHTML="hello world!";
```

### sw.js
```js
var cacheStorageKey = 'one';

var cacheList = [
    "index.html",
    "index.js"
]
self.addEventListener('install', function (e) {
    console.log('sw-install);
    self.skipWaiting();
})
self.addEventListener('activate', function (e) {
    console.log('sw-activate);
    caches.open(cacheStorageKey).then(function (cache) {
        cache.addAll(cacheList)
    })
    var cacheDeletePromises = caches.keys().then(cacheNames => {
        return Promise.all(cacheNames.map(name => {
            if (name !== cacheStorageKey) {
                // if delete cache,we should post a message to client which can trigger a callback function
                console.log('caches.delete', caches.delete);
                var deletePromise = caches.delete(name);
                send_message_to_all_clients({ onUpdate: true })
                return deletePromise;
            } else {
                return Promise.resolve();
            }
        }));
    });
    e.waitUntil(
        Promise.all([cacheDeletePromises]
        ).then(() => {
            return self.clients.claim()
        })
    )
})
self.addEventListener('fetch', function (e) {
    e.respondWith(
        caches.match(e.request).then(function (response) {
            if (response != null) {
                console.log(`fetch:${e.request.url} from cache`);
                return response
            } else {
                console.log(`fetch:${e.request.url} from http`);
                return fetch(e.request.url)
            }
        })
    )
})
```

### 说明

这样就完成了一个简单的sw页面了，现在通过服务器访问页面html、js资源将直接从客户端本地读取，实现页面的快速打开和离线访问

- 客户端和sw都有不同的事件回调，这些事件将在不同的sw生命周期中被触发，后续会有详细介绍
- 首次打开页面的时候sw先进行install回调，执行self.skipWaiting()后将接着执行activate，activate内对缓存列表的文件进行缓存
- cacheStorageKey是缓存的识别码，当cacheStorageKey的值变化，sw的activate会将旧的缓存给删除掉，重新调用cache.addAll设置缓存
- sw的fetch事件会拦截页面发出的请求，将根据缓存情况作出不同的处理

### 生命周期与事件

sw应用的生命周期我简单抽象为三种
- *安装*：页面首次打开，加载对应的sw文件
- *活动*：加载过sw文件后，打开页面
- *更新*：当服务器的sw文件与客户端的不一致的时候打开页面

#### 客户端

| 名称 | installing | active |
| ------ | ------ | ------ |
| 安装 | 触发 | 不触发 |
| 活动 | 不触发 | 触发 |
| 更新 | 不触发 | 触发 |

#### sw

| 名称 | install | activate | fetch|
| ------ | ------ | ------ |------|
| 安装 | 触发 | 触发 |不触发|
| 活动 | 不触发 | 不触发 |触发|
| 更新 | 触发 | 触发 |不触发|

总结一下：
- 客户端除了在首次打开的时候触发installing，其他都是触发的active
- sw端活动状态只执行fetch，安装和更新状态只执行install和activate


## 页面与sw通信

通信方面我之前有翻译过文章，[链接地址](https://github.com/violinux666/blog/issues/4)，大家感兴趣可以看看。这里我直接展示把封装好的通信接口接口

有了通信接口，我们就可以优化很多事情，比方说在***cacheStorageKey发生变化的时候通知页面给予客户一定的响应***

### 客户端

```js
function send_message_to_sw(msg){
    return new Promise(function(resolve, reject){
        // Create a Message Channel
        var msg_chan = new MessageChannel();
        // Handler for recieving message reply from service worker
        msg_chan.port1.onmessage = function(event){
            if(event.data.error){
                reject(event.data.error);
            }else{
                resolve(event.data);
            }
        };
        // Send message to service worker along with port for reply
        navigator.serviceWorker.controller.postMessage(msg, [msg_chan.port2]);
    });
}
```

### sw
```js
function send_message_to_client(client, msg){
  return new Promise(function(resolve, reject){
      var msg_chan = new MessageChannel();
      msg_chan.port1.onmessage = function(event){
          if(event.data.error){
              reject(event.data.error);
          }else{
              resolve(event.data);
          }
      };
      client.postMessage(msg, [msg_chan.port2]);
  });
}
function send_message_to_all_clients(msg){
  clients.matchAll().then(clients => {
      clients.forEach(client => {
          send_message_to_client(client, msg).then(m => console.log("SW Received Message: "+m));
      })
  })
}
```

### 动态缓存资源文件

上述的做法需要事先写好cacheList，有一定的维护量，现在介绍一种不需要维护cacheList的做法：

```js
self.addEventListener('fetch', function (e) {
  e.respondWith(
    caches.match(e.request).then(res => {
      return res ||
        fetch(e.request)
          .then(res => {
            const resClone = res.clone();
            caches.open(cacheStorageKey).then(cache => {
              cache.put(e.request, resClone);
            })
            return res;
          })
    })
  )
});
```

这样做的话缓存资源的操作将从activate转移到fetch事件内，fetch事件先判断有没有缓存，没有缓存的话将发出对应的请求并进行缓存

这样的做法的缺点是无法在首次加载页面的时候就完成静态化，因为sw的安装声明周期是不会触发sw的fetch事件的。

## 页面url带参数

针对一些页面渲染结果与url参数有关的情况，上述的架构无法完成对应的本地化需求。之前的做法是在cacheList加入了入口页面的地址，无法适应带动态参数url的情况。

### 在fetch内动态缓存请求

具体做法在**动态缓存资源文件**章节有描述，不再重复描述。

### 使用通信接口通知sw缓存入口页面

#### 客户端
```js
navigator.serviceWorker.register(file).then(function (reg) {
    if (reg.installing) {
        //send_message_to_sw({pageUrl:location.href})
    }
    if (reg.active) {
        send_message_to_sw({pageUrl:location.href})
    }
    return reg;
})
```

#### sw
```js
self.addEventListener('message',function(e){
  var data=e.data;
  if(data.pageUrl){
    addPage(data.pageUrl)
  }
})
function addPage(page){
  caches.open(cacheStorageKey).then(function(cache){
    cache.add(page);
  })
}
```

在客户端的active发消息给sw，sw就能够获取到对应的页面url进行缓存。

*注：*客户端的installing事件没法使用消息接口，大家可以在sw的activate事件向客户端发出一个消息请求获取当前页面url

## 常见问题

sw文件至少要与入口页面文件在同一目录下，如：

- /sw.js 可以管理 /index.html
- /js/sw.js 不能管理 /index.html

笔者在这里踩了很久的坑...

## webpack-sw-plugin

介绍一个笔者写的webpack的sw插件，在弄sw页面的时候很方便，[github地址](https://github.com/violinux666/webpack-sw-plugin)

### 安装

```
npm install --save-dev webpack-sw-plugin
```

### webpack配置

```js
const WebpackSWPlugin = require('webpack-sw-plugin');
module.exports = {
    // entry
    // output
    plugins:[
        new WebpackSWPlugin()
    ]
}
```

### 客户端配置

```js
import worker from 'webpack-sw-plugin/lib/worker';
worker.register({
    onUpdate:()=>{
        console.log('client has a new version. page will refresh in 5s....');
        setTimeout(function(){
            window.location.reload();
        },5000)
    }
});
```

### 效果

- 自动生成页面与sw交互的体系，无需提供额外的sw文件
- 自动适配url带参数的情况
- 当webpack输出文件变化的时候，客户端的onUpdate将会被触发，上述例子中当输出文件变化时，客户端将会在5秒后进行刷新，刷新后将会使用全新的文件