## Middleware: 全局异常处理

我们在岩浆的实例其实已经注意到了，compose 的连接方式，让我们有能力精确控制异常。

Koa中间件最终行为强依赖注册顺序，比如我们这里要引入的异常处理，必须在业务逻辑中间件前注册，才能捕获后续中间件中未捕获异常，回想一下我们的调度器实现的异常传递流程。


```php
<?php

class ExceptionHandler implements Middleware
{
    public function __invoke(Context $ctx, $next)
    {
        try {
            yield $next;
        } catch (\Exception $ex) {
            $status = 500;
            $code = $ex->getCode() ?: 0;
            $msg = "Internal Error";

            // HttpException的异常通常是通过Context的throw方法抛出
            // 状态码与Msg直接提取可用
            if ($ex instanceof HttpException) {
                $status = $ex->status;
                if ($ex->expose) {
                    $msg = $ex->getMessage();
                }
            }
            // 这里可这对其他异常区分处理
            // else if ($ex instanceof otherException) { }

            $err = [ "code" => $code,  "msg" => $msg ];
            if ($ctx->accept("json")) {
                $ctx->status = 200;
                $ctx->body = $err;
            } else {
                $ctx->status = $status;
                if ($status === 404) {
                    $ctx->body = (yield Template::render(__DIR__ . "/404.html"));
                } else if ($status === 500) {
                    $ctx->body = (yield Template::render(__DIR__ . "/500.html", $err));
                } else {
                    $ctx->body = (yield Template::render(__DIR__ . "/error.html", $err));
                }
            }
        }
    }
}

$app->υse(new ExceptionHandler());

```

可以将FastRoute与Exception中间件结合，很容可以定制一个按路由匹配注册的异常处理器，留待读者自行实现。


顺序问题再比如session中间件，需要优先于业务处理中间件, 

而像处理404状态码的中间件，则需要在upstream流程中(逆序)的收尾阶段，我们仅仅只关心next后逻辑：

```php
<?php
function notfound(Context $ctx, $next)
{
    yield $next;

    if ($ctx->status !== 404 || $ctx->body) {
        return;
    }

    $ctx->status = 404;

    if ($ctx->accept("json")) {
        $ctx->body = [
            "message" => "Not Found",
        ];
        return;
    }

    $ctx->body = "<h1>404 Not Found</h1>";
}
```
