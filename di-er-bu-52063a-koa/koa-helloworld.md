## Koa - HelloWorld

以上便是全部了, 我们重点来看示例:


我们只注册一个中间件, Hello Worler Server

```php
<?php

$app = new Application();

// ...

$app->υse(function(Context $ctx) {
    $ctx->status = 200;
    $ctx->body = "<h1>Hello World</h1>";
});

$app->listen(3000);

```

我们在Hello中间件前面注册一个Reponse-Time中间件, 注意看, 我们的逻辑是连贯的;

```php
<?php

$app->υse(function(Context $ctx, $next) {
    
    $start = microtime(true);

    yield $next; // 执行后续中间件

    $ms = number_format(microtime(true) - $start, 7);

    // response header 写入 X-Response-Time: xxxms
    $ctx->{"X-Response-Time"} = "{$ms}ms";
});

```