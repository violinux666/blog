> Service workers 本质上充当Web应用程序与浏览器之间的代理服务器，也可以在网络可用时作为浏览器和网络间的代理。它们旨在（除其他之外）使得能够创建有效的离线体验，拦截网络请求并基于网络是否可用以及更新的资源是否驻留在服务器上来采取适当的动作。他们还允许访问推送通知和后台同步API。(引用自：[链接](https://developer.mozilla.org/zh-CN/docs/Web/API/Service_Worker_API))

上面是MDN对ServiceWorker的描述，SW的API很多，学习起来很不方便，本文将对重要的业务场景进行封装，简化读者的学习成本。