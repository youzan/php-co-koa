## callcc


```php
<?php

// CallCC instanceof Async
class CallCC implements Async
{
    public $fun;

    public function __construct(callable $fun)
    {
        $this->fun = $fun;
    }

    public function begin(callable $continuation)
    {
        $fun = $this->fun;
        $fun($continuation);
    }
}

function callcc(callable $fn)
{
    return new CallCC($fn);
}
```

我们创造的半协程中的callcc的功能有限, yield只能将控制权从Generator转移到起caller中:

[Wiki - Coroutine](https://en.wikipedia.org/wiki/Coroutine)

> Generators, also known as semicoroutines, are also a generalisation of subroutines, but are more limited than coroutines. Specifically, while both of these can yield multiple times, suspending their execution and allowing re-entry at multiple entry points, they differ in that coroutines can control where execution continues after they yield, while generators cannot, instead transferring control back to the generator's caller. That is, since generators are primarily used to simplify the writing of iterators, the yield statement in a generator does not specify a coroutine to jump to, but rather passes a value back to a parent routine.

> However, it is still possible to implement coroutines on top of a generator facility, with the aid of a top-level dispatcher routine (a trampoline, essentially) that passes control explicitly to child generators identified by tokens passed back from the generators


我们引入的callcc实际上是人肉进行的thunky, 来看例子:


```php
<?php

function async_sleep($ms)
{
    return callcc(function($k) use($ms) {
        swoole_timer_after($ms, function() use($k) {
            $k(null);
        });
    });
}

function async_dns_lookup($host)
{
    return callcc(function($k) use($host) {
        swoole_async_dns_lookup($host, function($host, $ip) use($k) {
            $k($ip);
        });
    });
}

class HttpClient extends \swoole_http_client
{
    public function async_get($uri)
    {
        return callcc(function($k) use($uri) {
            $this->get($uri, $k);
        });
    }

    public function async_post($uri, $post)
    {
        return callcc(function($k) use($uri, $post) {
            $this->post($uri, $post, $k);
        });
    }

    public function async_execute($uri)
    {
        return callcc(function($k) use($uri) {
            $this->execute($uri, $k);
        });
    }
}

// 这里!
spawn(function() {
    $ip = (yield async_dns_lookup("www.baidu.com"));
    $cli = new HttpClient($ip, 80);
    $cli->setHeaders(["foo" => "bar"]);
    $cli = (yield $cli->async_get("/"));
    echo $cli->body, "\n";
});

```

我们可以用相同的方式来封装swoole剩余的异步api(TcpClient,MysqlClient,RedisClient...), 大家可以举一反三(建议继承swoole原生类, 而不是直接实现Async);