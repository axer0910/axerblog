---
title: 简单的Js mock服务器自动检测文件更新重载
tag: frontend
date: 2019-12-23
updated: 2019-12-23
---


原理就是检测文件修改后杀进程并且重启进程
使用`cross-spawn`避免windows下启动进程找不到的问题
使用`tree-kill`避免linux下面有时候kill进程无效的问题
使用`node-watch`递归检测文件夹内容修改

```javascript
const spawn = require('cross-spawn');
const kill = require('tree-kill');
const childProcess = require('child_process');
const fs = require('fs');
const watch = require('node-watch');
os = require('os');
let ps = spawn('bash', ['runmock.sh']);

ps.stdout.on('data', (data) => {
  console.log(data.toString());
});

console.info('start listen json mock server at port: 30005');

watch('./mock', { recursive: true }, (event, filename) => {
  console.info('reload mock server');
  if (os.platform() === 'win32') {
    childProcess.exec('taskkill /pid ' + ps.pid + ' /T /F');
  } else {
    console.log('kill ' + ps.pid);
    kill(ps.pid);
  }
  ps = spawn('bash', ['runmock.sh']);
});

```
