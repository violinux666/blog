[原文](http://craig-russell.co.uk/2016/01/29/service-worker-messaging.html#.XCc7eJP1XvC)

*Service Workers*是一个为页面工作的后台处理器。提供离线web apps是Service Workers目前最让人感兴趣的功能，同时Service Workers能够管理一个本地的资源缓存，当网络连接状态是正常的时候，这个本地资源缓存能够自动跟服务器进行同步。这是十分酷的，但我想谈一下Service Workers的另一个用途，使用它来管理多个web页面之间的通信。

例如，你可能有一个应用打开在多个浏览器页签中。Service Workers能够更新一个页签当其他页签有一个事件触发，也可以做到当服务器发出一个消息后，所有页签的内容将被更新。

一个Service Workers可以控制多个客户端页面，例如ServiceWorker会自动的控制它范围内的所有客户端页面，它的范围是指你站点下的url，通常来讲是Service Worker script文件的路径。

在这个demo里，我们将使用三个文件 **client1.html** **client2.html** **service-worker.js**

首先我们注册service worker 在 *client1.html*

```html
<!doctype html>
<html>
<head>
    <title>Service Worker - Client 1</title>
</head>
<body>
    <script>
        if('serviceWorker' in navigator){
            // Register service worker
            navigator.serviceWorker.register('/service-worker.js').then(function(reg){
                console.log("SW registration succeeded. Scope is "+reg.scope);
            }).catch(function(err){
                console.error("SW registration failed with error "+err);
            });
        }
    </script>
</body>
</html>
```

接着我们创建一个基本的 Service worker 在*service-worker.js*

```js
console.log("SW Startup!");

// Install Service Worker
self.addEventListener('install', function(event){
    console.log('installed!');
});

// Service Worker Active
self.addEventListener('activate', function(event){
    console.log('activated!');
});
```

我不会去解释他是怎么工作的，因为在很多地方都有记录（译者注：作者的意思是很多地方都有console.log）

我们同时创建了client2.html 我们只会在client1注册serviceworker，所以不需要有重复的代码在这里。serviceworker运行的时候将会自动的控制它的作用域下的页面。

```html
<!doctype html>
<html>
<head>
    <title>Service Worker - Client 2</title>
</head>
<body>
    <script>
    </script>
</body>
</html>
```

如果你在浏览器上访问*client1.html*你应该会看到由console.log输出的注册信息。在Chrome(48+)中，你可以在开发工具的“Resouces”选项卡下点击“inspect”，为服务工作者打开一个检查器。（译者注：这个我没有找到）。当你打开client2.html在新的浏览器页签，你可以在开发工具内的“Controlled Clients”找到它

现在我们可以继续讲有趣的东西了。

---

首先我们让客户端发消息给serviceworker。所有我们需要去加一个消息的处理在service-worker.js

```js
self.addEventListener('message', function(event){
    console.log("SW Received Message: " + event.data);
});
```

现在我们增加一个发消息的函数在两个客户端里

```js
function send_message_to_sw(msg){
    navigator.serviceWorker.controller.postMessage("Client 1 says '"+msg+"'");
}
```

如果你在客户端页面的控制台内调用send_message_to_sw("Hello")，你因该可以在serviceworker的控制台内看到有消息显示

我们可以进一步的让serviceworker去响应客户端发来的消息。实现它我们需要去改良我们的send_message_to_sw函数。我们使用‘Message Channel’，Message Channel能够提供了一对端口（port）来进行通信。我们将一个引用连同消息一起发送到端口的另一端，所以Service Worker能够使用它去进行答复。我们也可以对这些响应消息做一些处理。为了方便起见，我们还使用Promise来处理等待响应。

**译者注：这里说的端口（port）用于页面与serviceworker之间的通信**

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
        navigator.serviceWorker.controller.postMessage("Client 1 says '"+msg+"'", [msg_chan.port2]);
    });
}
```

在*service-worker.js*我们修改了监听器，与消息一起发送响应在端口

```js
self.addEventListener('message', function(event){
    console.log("SW Received Message: " + event.data);
    event.ports[0].postMessage("SW Says 'Hello back!'");
});
```

现在如果在你的客户端控制台执行```send_message_to_sw("Hello").then(m => console.log(m))```，你将看到信息显示在serviceworker的控制台里，在客户端的控制台将会有答复。请注意，我们使用Promise then函数来等待响应和箭头函数，因为这样更容易去测定（译者注：这里type我翻译成控制）。

现在我们有了一个让客户端发消息给serviceworker同时serviceworker能够答复的机制。您可以使用它让客户机检查长时间运行的流程的状态，让serviceworker将消息转发给所有客户端或其他一些很酷的东西。

---

现在我们允许serviceworker广播一个事件到所有的客户端让所有客户响应。这与以前使用的机制类似，只是角色颠倒了。

首先我们在客户端增加一个消息监听器，我们增加了测试serviceworker兼容性的代码，其他的地方几乎相同。

```js
if('serviceWorker' in navigator){
    // Handler for messages coming from the service worker
    navigator.serviceWorker.addEventListener('message', function(event){
        console.log("Client 1 Received Message: " + event.data);
        event.ports[0].postMessage("Client 1 Says 'Hello back!'");
    });
}
```

接着我们给serviceworker增加一个发送消息给客户端的函数。这也跟之前很类似，只是我们需要提供给一个客户端对象（一个页面的应用），这个对象能告诉我们要往哪里发消息。

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

        client.postMessage("SW Says: '"+msg+"'", [msg_chan.port2]);
    });
}
```

serviceworker API提供了获取所有已连接客户端引用的接口。我们可以将其封装在一个方便的函数中，以便向所有客户机广播消息(注意，我们再次使用箭头函数)。

```js
function send_message_to_all_clients(msg){
    clients.matchAll().then(clients => {
        clients.forEach(client => {
            send_message_to_client(client, msg).then(m => console.log("SW Received Message: "+m));
        })
    })
}
```

现在如果我们在serviceworker的控制台内执行```send_message_to_all_clients('Hello')```,您将看到在所有客户端控制台中接收到的消息，以及在serviceworker控制台中客户机的响应。
