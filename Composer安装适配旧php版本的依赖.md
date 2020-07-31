---
title: Composer安装适配旧php版本的依赖
tag: frontend
date: 2019-01-22
updated: 2019-01-22
---

为了想让TP5使用Doctrine这个ORM的库，本地测试完成以后（PHP7.1）放到线上报错：unexpected public const。一查，这是php7.1的语法，但是线上环境是7.0。考虑到现在线上7.0运行稳定如果升级到7.1可能会出现不兼容风险，决定研究降级安装7适配7.0的Doctrine。
composer.json在config字段里面可以增加一个platform字段定义当前php运行的版本：
```javascript
"config": {
        "preferred-install": "dist",
        "platform": {
            "php": "7.0.11"
        }
    }
```
**修改完以后一定要运行composer update**，composer会检查当前所有已安装的包并且降级到7.0可以使用的版本。如果直接composer require一个新包，会提示7.0.11不满足xx包最低运行条件7.1.3这样的错误，一定要先composer update后再去安装需要的依赖。
最好平时开发环境php版本和线上保持一致，避免线上不兼容问题。