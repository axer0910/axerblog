---
title: 创建一个Swoole http 响应的类
tag: php
date: 2018-07-18
updated: 2018-07-18
---

在udp模式下工作的`swoole_server`实例中，通过`addlistener`可以额外监听一个80端口的请求，但是没有找到方法可以直接输出http响应到tcp中，于是自己简单写了一个类简单返回http响应类，这样在upd模式下既可以响应udp请求，也能处理http接口请求了：

```php
<?php

namespace Lumi\utils;
use Lumi\LumiServ;

class HttpResponse {

    private $protocol = 'HTTP/1.1';
    private $status_code = '200';
    private $content = '';
    private $headers = [];
    private $swoole_srv;
    private $fd;

    public function __construct(\swoole_server $lumiServ, $fd)
    {
        $this->swoole_srv = $lumiServ;
        $this->fd = $fd;
    }

    /**
     * 设置http头
     * @param array $headers
     * @return HttpResponse
     */
    public function setHeaders(array $headers) {
        $this->headers = array_merge($this->headers, $headers);
        return $this;
    }

    /**
     * 设置http状态码
     * @param $status_code
     * @return HttpResponse
     */
    public function setStatusCode($status_code) {
        $this->status_code = $status_code;
        return $this;
    }

    public function setContent($content) {
        $this->content = $content;
        // 设置响应体长度
        $headers = [
            'Content-Length' => strlen($content)
        ];
        $this->setHeaders($headers);
        return $this;
    }

    /**
     * 工厂获取一个http实例
     * @param LumiServ $lumiServ
     * @return HttpResponse
     */
    static public function getResponse(\swoole_server $serv, $fd) {
        $headers = [
            'Server'=>'SwooleServer',
            'Content-Type'=>'text/html;charset=utf8'
        ];

        $ins = new HttpResponse($serv, $fd);
        $ins->setHeaders($headers);

        return $ins;
    }

    /**
     * 发送http响应
     */
    public function sendResponse() {
        $response = [
            $this->protocol . ' ' . $this->status_code
        ];
        // 拼headers
        foreach($this->headers as $key => $val){
            $response[] = $key.':'.$val;
        }
        //空行
        $response[] = '';
        //响应体
        $response[] = $this->content;
        $send_data = join("\r\n",$response);
        $this->swoole_srv->send($this->fd, $send_data);
    }
}

```
使用方法：

```php
HttpResponse::getResponse($serv, $fd)->setContent($respData)->sendResponse();
```

其中`$serv`是`swoole` `onreceive`方法返回的swoole_server实例。