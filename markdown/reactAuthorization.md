[原文](https://alligator.io/react/simple-authorization/)

> 大多数的应用都需要身份验证机制和授权机制，当验证机制确认某些实体是合法用户时，授权机制将根据用户的角色和权限去决定用户是否被允许去执行这些操作

在大多数情况下，我们通常不需要特殊的模块或者库来处理授权，大多数的工具函数已经足够了。由你提供的应用内的验证或者授权解决方法是可以变化的。你可能会决定把用户的状态放到Redux去管理，你也可能创建一个专用的模块，等等...

让我们看看如何在React中处理一个简单的基于角色的授权机制

## 简单的授权机制

假设我们有一个用户对象，这个对象通常是在完成登录后通过调用一个终端获得的，它有以下的结构

```jsx
const user = {
  name: 'Jackator',
  // ...
  roles: ['user'],
  rights: ['can_view_articles']
};
```

用户有几个可以分组为角色的权限。对于你的应用，你可能只需要角色，或者只需要权限或者两者都需要，那都不是问题。REST API 可能给你嵌套在角色里的权限，那也不是问题，请记住一点，你要根据你的需求来调整解决方案。最重要的是我们拥有一个用户对象。

之后我们创建一个*auth.js*文件，里面有一些工具方法可能帮助我们去检查用户的授权情况

### auth.js

```jsx
export const isAuthenticated = user => !!user;

export const isAllowed = (user, rights) =>
  rights.some(right => user.rights.includes(right));

export const hasRole = (user, roles) =>
  roles.some(role => user.roles.includes(role));
```

通过使用Array prototype上的*some*和*includes*方法，我们检查了客户是否用友至少一个权限或者角色，这应该足以执行一些基本的权限检查

因为用户对象可以保存在任何地方，以Redux为例子，我们允许将它作为参数传递给函数。

我们最终创建了一个简单的React组件，它使用了*auth.js*里的函数为了能够根据条件去显示界面上不同的部分

### App.js

```jsx
import React from 'react';
import { render } from "react-dom";
import { hasRole, isAllowed } from './auth';

const user = {
  roles: ['user'],
  rights: ['can_view_articles']
};

const admin = {
  roles: ['user', 'admin'],
  rights: ['can_view_articles', 'can_view_users']
};

const App = ({ user }) => (
  <div>
    {hasRole(user, ['user']) && <p>Is User</p>}
    {hasRole(user, ['admin']) && <p>Is Admin</p>}
    {isAllowed(user, ['can_view_articles']) && <p>Can view Articles</p>}
    {isAllowed(user, ['can_view_users']) && <p>Can view Users</p>}
  </div>
);

render(
  <App user={user} />,
  document.getElementById('root')
);
```

我们使用 && 逻辑符进行短路评估（译者注：这里我直译了，大概的意思使用&&来决定页面内容是否展示）。在这里，如果*hasRole*或者*isAllowed*函数返回true，&&符号后面的内容建会被渲染。

尝试去将user改变为admin，你将看到admin相关的界面会被显示

## 条件路由

如果你使用了*React Router*，你可以使用相同的方法，根据条件去渲染路由：

```jsx
import React from 'react';
import { BrowserRouter, Switch, Route } from 'react-router-dom';

const App = ({ user }) => (
  <BrowserRouter>
    <Switch>
      {hasRole(user, ['user']) && <Route path='/user' component={User} />}
      {hasRole(user, ['admin']) && <Route path='/admin' component={Admin} />}
      <Route exact path='/' component={Home} />
    </Switch>
  </BrowserRouter>
);
```

*React Router* 让Route组件的声明和路由组合变得很容易，我们可以利用它：如果*<Route>*组件通过*hasRole*的校验被渲染，*Route*将会被*Router*增加和执行



