## call/cc

[wiki - call-with-current-continuation](https://en.wikipedia.org/wiki/Call-with-current-continuation)

> the function call-with-current-continuation, abbreviated call/cc, is a control operator

异步回调API是无法直接使用yield语法的，需要使用thunk或者promise进行转换，thunkify是将多参数函数替换成单参数函数且参数只接受回调(参见：[Thunk 函数的含义和用法](http://www.ruanyifeng.com/blog/2015/05/thunk.html))。上文中我们将回调api显式实现Async接口，显得有些麻烦，这里可以把 **“通过call参数$k传递异步结果”** 的模式抽象出来，实现一个穷人的call/cc。

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

[wiki - call-with-current-continuation](https://en.wikipedia.org/wiki/Call-with-current-continuation)

> Taking a function f as its only argument, call/cc takes the current continuation as an object and applies f to it. The continuation object is a first-class value and is represented as a function, with function application as its only operation. When a continuation object is applied to an argument, the existing continuation is eliminated and the applied continuation is restored in its place, so that the program flow will continue at the point at which the continuation was captured and the argument of the continuation then becomes the "return value" of the call/cc invocation. Continuations created with call/cc may be called more than once, and even from outside the dynamic extent of the call/cc application.

我们的call/cc只可以调用一次(Generator是单向的)，虽然我们的$k也是`first-class value`，但即使`$k($k)`进行传递，也无法达到wiki介绍的效果，PHP不支持Continuation，我们创造的半协程中的call/cc的功能有限，仅仅借用了call/cc的形式。

事实上，yield只能将控制权从Generator转移到起caller中：

[Wiki - Coroutine](https://en.wikipedia.org/wiki/Coroutine)

> Generators, also known as semicoroutines, are also a generalisation of subroutines, but are more limited than coroutines. Specifically, while both of these can yield multiple times, suspending their execution and allowing re-entry at multiple entry points, they differ in that coroutines can control where execution continues after they yield, while generators cannot, instead transferring control back to the generator's caller. That is, since generators are primarily used to simplify the writing of iterators, the yield statement in a generator does not specify a coroutine to jump to, but rather passes a value back to a parent routine.

> However, it is still possible to implement coroutines on top of a generator facility, with the aid of a top-level dispatcher routine (a trampoline, essentially) that passes control explicitly to child generators identified by tokens passed back from the generators


来看例子:


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

我们可以用相同的方式来封装swoole其他的异步api(TcpClient，MysqlClient，RedisClient...)，大家可以举一反三(建议继承swoole原生类，而不是直接实现Async)。