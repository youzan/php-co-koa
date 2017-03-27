## Middleware: 请求超时

请求超时控制也是不可或缺的中间件：


```php
<?php

class RequestTimeout implements Middleware
{
    public $timeout;
    public $exception;

    private $timerId;

    public function __construct($timeout, \Exception $ex = null)
    {
        $this->timeout = $timeout;
        if ($ex === null) {
            $this->exception = new HttpException(408, "Request timeout");
        } else {
            $this->exception = $ex;
        }
    }

    public function __invoke(Context $ctx, $next)
    {
        yield race([
            callcc(function($k) {
                $this->timerId = swoole_timer_after($this->timeout, function() use($k) {
                    $k(null, $this->exception);
                });
            }),
            function() use ($next){
                yield $next;
                if (swoole_timer_exists($this->timerId)) {
                    swoole_timer_clear($this->timerId);
                }
            },
        ]);
    }
}

$app->υse(new RequestTimeout(2000));
```

也可以结合FastRoute构造出一个按路由匹配请求超时的中间件，留给读者自行实现。