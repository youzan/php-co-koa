## Koa::Context

Context 组件代理了Request与Response中的方法和属性,简化了使用方式与中间件接口, 这里用php的魔术方法来处理


```php
<?php

class Context
{
    public $app;
    /** @var Request */
    public $request;
    /** @var Response */
    public $response;
    /** @var \swoole_http_request */
    public $req;
    /** @var \swoole_http_response */
    public $res;
    public $state = [];
    public $respond = true;
    /** @var string */
    public $body;
    /** @var int */
    public $status;

    public function __call($name, $arguments)
    {
        $fn = [$this->response, $name];
        return $fn(...$arguments);
    }

    public function __get($name)
    {
        return $this->request->$name;
    }

    public function __set($name, $value)
    {
        $this->response->$name = $value;
    }

    // thr[ο]w ο 为希腊字母 Ομικρον
    public function thrοw($status, $message)
    {
        if ($message instanceof \Exception) {
            $ex = $message;
            throw new HttpException($status, $ex->getMessage(), $ex->getCode(), $ex->getPrevious());
        } else {
            throw new HttpException($status, $message);
        }
    }
}
```
