---
title: Vue router 模拟小程序栈结构前进后退路由
tag: frontend
date: 2019-02-17
updated: 2019-02-17
---

  在单页应用里如果只用路由的go(-1)方法后退，或者一直使用浏览器的后退按钮，容易陷入后退循环，例如:A页面→B页面→C页面→B页面→C页面→B页面→C页面→B页面→C页面，只使用h5后退会变成C页面→B页面C页面→B页面C页面→B页面C页面→B页面→A页面。想到的解决方案是模拟微信小程序路由的方式，将所有走过的页面放到一个栈里面，push路由入栈，检测到后退事件就出栈。解决的难点是怎么判断h5的后退事件。


#### 官方完整的导航解析流程
1. 导航被触发。
2. 在失活的组件里调用离开守卫。
3. 调用全局的 beforeEach 守卫。
4. 在重用的组件里调用 beforeRouteUpdate 守卫 (2.2+)。
5. 在路由配置里调用 beforeEnter。
6. 解析异步路由组件。
7. 在被激活的组件里调用 beforeRouteEnter。
8. 调用全局的 beforeResolve 守卫 (2.5+)。
9. 导航被确认。
10. 调用全局的 afterEach 钩子。
11. 触发 DOM 更新。
12. 用创建好的实例调用 beforeRouteEnter 守卫中传给 next 的回调函数。

beforeEach钩子里有next()语句用来解析当前导航栏上路由地址。当用户点击后退的时候实际上导航栏上的地址已经更新为h5的history后退的地址（这个时候可能是错误的后退地址）并同时触发`popstate`事件。这个时候出栈历史纪录栈，并且在popstate事件回调中强制将当前路由替换为出栈的路由即可。

<!-- more -->

```javascript
let isPopState = false;
let routeStack = [];

window.addEventListener('popstate', function(event) {
  console.log('popstate fired!');
  isPopState = true;
  if (routeStack.length === 0) {
    router.replace('/');
  } else {
    let route = routeStack.pop();
    console.log('pop routeStack result', routeStack);
    router.replace(route.fullPath);
  }
});

router.beforeEach((to, from, next) => {
  console.log('beforeEach', to);
  setTimeout(() => {
    // 延时让popstate事件先运行再resolve路由结果
    next();
  }, 100);
});

router.afterEach((to, from) => {
  console.log('do after each');
  if (!isPopState) {
    // 不是由popstate事件触发的，入栈历史路由
    routeStack.push(from);
    console.log('push routeStack result', routeStack);
  }
  isPopState = false;
});
```