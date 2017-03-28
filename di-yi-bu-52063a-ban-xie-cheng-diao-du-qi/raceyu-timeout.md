## race与timeout

看到这里，你可能已经发现到我们封装的异步接口的问题了: 没有任何超时处理。

通常情况我们会为每个异步调用添加定时器，回调成功取消定时器，否则在定时器回调透传异常，例如：

```php
<?php

// helper function
function once(callable $fun)
{
    $has = false;
    return function(...$args) use($fun, &$has) {
        if ($has === false) {
            $fun(...$args);
            $has = true;
        }
    };
}

// helper function
function timeoutWrapper(callable $fun, $timeout)
{
    return function($k) use($fun, $timeout) {
        $k = once($k);
        $fun($k);
        swoole_timer_after($timeout, function() use($k) {
            // 这里异常可以从外部传入
            $k(null, new \Exception("timeout"));
        });
    };
}
```

```php
<?php
// 为callcc添加超时处理
function callcc(callable $fun, $timeout = 0)
{
    if ($timeout > 0) {
        $fun = timeoutWrapper($fun, $timeout);
    }
    return new CallCC($fun);
}


// 我们的dns查询有了超时透传异常的能力了
function async_dns_lookup($host, $timeout = 100)
{
    return callcc(function($k) use($host) {
        swoole_async_dns_lookup($host, function($host, $ip) use($k) {
            $k($ip);
        });
    }, $timeout);
}


spawn(function() {
    try {
        yield async_dns_lookup("www.xxx.com", 1);
    } catch (\Exception $ex) {
        echo $ex; // ex!
    }
});

```

但是，我们可以有更优雅通用的方式来超时处理:


```php
<?php

class Any implements Async
{
    public $parent;
    public $tasks;
    public $continuation;
    public $done;

    public function __construct(array $tasks, AsyncTask $parent = null)
    {
        assert(!empty($tasks));
        $this->tasks = $tasks;
        $this->parent = $parent;
        $this->done = false;
    }

    public function begin(callable $continuation)
    {
        $this->continuation = $continuation;
        foreach ($this->tasks as $id => $task) {
            (new AsyncTask($task, $this->parent))->begin($this->continuation($id));
        };
    }

    private function continuation($id)
    {
        return function($r, $ex = null) use($id) {
            if ($this->done) {
                return;
            }
            $this->done = true;

            if ($this->continuation) {
                $k = $this->continuation;
                $k($r, $ex);
            }
        };
    }
}
```

```php
<?php
// helper function
function await($task, ...$args)
{
    if ($task instanceof \Generator) {
        return $task;
    }

    if (is_callable($task)) {
        $gen = function() use($task, $args) { yield $task(...$args); };
    } else {
        $gen = function() use($task) { yield $task; };
    }
    return $gen();
}


function race(array $tasks)
{
    $tasks = array_map(__NAMESPACE__ . "\\await", $tasks);

    return new Syscall(function(AsyncTask $parent) use($tasks) {
        if (empty($tasks)) {
            return null;
        } else {
            return new Any($tasks, $parent);
        }
    });
}
```


我们构造了一个与Promise.race相同语义的接口，而我们之前构造Async接口则可以看成简陋版的Promise.then + Promise.catch。

```php
<?php

// 我们重新来看这个简单dns查询函数
function async_dns_lookup($host)
{
    return callcc(function($k) use($host) {
        swoole_async_dns_lookup($host, function($host, $ip) use($k) {
            $k($ip);
        });
    });
}

// 我们有了一个纯粹的超时透传异常的函数
function timeout($ms)
{
    return callcc(function($k) use($ms) {
        swoole_timer_after($ms, function() use($k) {
            $k(null, new \Exception("timeout"));
        });
    });
}

// 当我们采取race语义并发执行dns查询与超时异常函数
// 其实我们构造了一个更为灵活的超时处理方案
spawn(function() {
    try {
        $ip = (yield race([
            async_dns_lookup("www.baidu.com"),
            timeout(100),
        ]));

        $res = (yield race([
            (new HttpClient($ip, 80))->awaitGet("/"),
            timeout(200),
        ]));
        var_dump($res->statusCode);
    } catch (\Exception $ex) {
        echo $ex;
        swoole_event_exit();
    }
});
```

我们非常容易构造出更多支持超时的接口, 但我们代码看起来比之前更加清晰;


```php
<?php

class HttpClient extends \swoole_http_client
{
    public function awaitGet($uri, $timeout = 1000)
    {
        return race([
            callcc(function($k) use($uri) {
                $this->get($uri, $k);
            }),
            timeout($timeout),
        ]);
    }
    // ...
}


function async_dns_lookup($host, $timeout = 100)
{
    return race([
        callcc(function($k) use($host) {
            swoole_async_dns_lookup($host, function($host, $ip) use($k) {
                $k($ip);
            });
        }),
        timeout($timeout),
    ]);
}


// test
spawn(function() {
    try {
        $ip = (yield race([
            async_dns_lookup("www.baidu.com"),
            timeout(100),
        ]));
        $res = (yield (new HttpClient($ip, 80))->awaitGet("/"));
        var_dump($res->statusCode);
    } catch (\Exception $ex) {
        echo $ex;
        swoole_event_exit();
    }
});

```
