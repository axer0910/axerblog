---
title: 简单实现一个中间件
tag: frontend
date: 2018-12-17
updated: 2018-12-17
---

中间件是常见的设计模式，常见于在某个流程中插入一个扩展的功能实现，实现对流程的改造，拦截，分流等效果。

```javascript
function Middleware() {
  this.middlewares = [];
  this.hook1 = false;
  this.hook2 = false;
}
// 加一个中间件
Middleware.prototype.use = function(fn) {
  this.middlewares.push(fn);
}

Middleware.prototype.middlewareRunner = function(middleWareFn) {
  return new Promise(resolve => {
    middleWareFn(() => {
      resolve();
    })
  })
}

Middleware.prototype.go = async function(afterGoFn) {
  // 先执行所有中间件，完毕后执行fn
  for(let middleware of this.middlewares) {
    await this.middlewareRunner(middleware);
  }
  afterGoFn();
}

var middleware = new Middleware();

middleware.use(function(next) {
  var self = this;
  console.log('run settime out1');
  setTimeout(function() {
      self.hook1 = true;
      next();
  }, 500);
});

middleware.use(function(next) {
  var self = this;
  console.log('run settime out2');
  setTimeout(function() {
      self.hook2 = true;
      next();
  }, 1500);
});

middleware.use(function(next) {
  var self = this;
  console.log('run settime out3');
  setTimeout(function() {
      self.hook2 = true;
      next();
  }, 500);
});

// 运行：
// 效果
// 输出run settime out1
// 间隔500ms再输出run settime out2
// 间隔1500ms在输出run settime out3
middleware.go(function() {
  console.log(this.hook1); 
  // true
  console.log(this.hook2); 
  // true
});
```