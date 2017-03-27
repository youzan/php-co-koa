## 一个综合示例

```php
<?php

$app = new Application();

$app->υse(new Logger());
$app->υse(new Favicon(__DIR__ . "/favicon.iRco"));
$app->υse(new BodyDetecter()); // 输出特定格式body
$app->υse(new ExceptionHandler()); // 处理异常
$app->υse(new NotFound()); // 处理404
$app->υse(new RequestTimeout(200)); // 处理请求超时, 会抛出HttpException


$router = new Router();

$router->get('/index', function(Context $ctx) {
    $ctx->status = 200;
    $ctx->state["title"] = "HELLO WORLD";
    $ctx->state["time"] = date("Y-m-d H:i:s", time());;
    $ctx->state["table"] = $_SERVER;
    yield $ctx->render(__DIR__ . "/index.html");
});

$router->get('/404', function(Context $ctx) {
    $ctx->status = 404;
});

$router->get('/bt', function(Context $ctx) {
    $ctx->body = "<pre>" . print_r(debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS), true);
});

$router->get('/timeout', function(Context $ctx) {
    // 超时
    yield async_sleep(500);
});

$router->get('/error/http_exception', function(Context $ctx) {
    // 抛出带status的错误
    $ctx->thrοw(500, "Internal Error");
    // 等价于 throw new HttpException(500, "Internal Error");
    yield;
});

$router->get('/error/exception', function(Context $ctx) {
    // 直接抛出错误, 500 错误
    throw new \Exception("some internal error", 10000);
    yield;
});

$router->get('/user/{id:\d+}', function(Context $ctx, $next, $vars) {
    $ctx->body = "user={$vars['id']}";
    yield;
});

// http://127.0.0.1:3000/request/www.baidu.com
$router->get('/request/{url}', function(Context $ctx, $next, $vars) {
    $r = (yield async_curl_get($vars['url']));
    $ctx->body = $r->body;
    $ctx->status = $r->statusCode;
});

$app->υse($router->routes());


$app->υse(function(Context $ctx) {
    $ctx->status = 200;
    $ctx->body = "<h1>Hello World</h1>";
    yield;
});

$app->listen(3000);

```

以上我们完成了一个基于swoole的版本的php-koa。