接下来我们来看Koa的仅有四个组件, Application, Context, Request, Response, 依次给予实现:

[Koa-guide](https://github.com/guo-yu/Koa-guide)

-----------------------------------------------------------------------------------------------

### Koa::Application


> The application object is Koa's interface with node's http server and handles the registration

> of middleware, dispatching to the middleware from http, default error handling, as well as

> configuration of the context, request and response objects.


我们的App模块是对swoole_http_server的简单封装, 在onRequest回调中对中间件compose并执行:

```php
<?php

class Application
{
    /** @var \swoole_http_server */
    public $httpServer;
    /** @var Context Prototype Context */
    public $context;
    public $middleware = [];
    public $fn;

    public function __construct()
    {
        // 我们构造一个Context原型
        $this->context = new Context();
        $this->context->app = $this;
    }

    // 我们用υse方法添加符合接口的中间件
    // middleware :: (Context $ctx, $next) -> void
    public function υse(callable $fn)
    {
        $this->middleware[] = $fn;
        return $this;
    }

    // compose中间件 监听端口提供服务
    public function listen($port = 8000, array $config = [])
    {
        $this->fn = compose(...$this->middleware);
        $config = ['port' => $port] + $config + $this->defaultConfig();
        $this->httpServer = new \swoole_http_server($config['host'], $config['port'], SWOOLE_PROCESS, SWOOLE_SOCK_TCP);
        $this->httpServer->set($config);
        // 省略绑定 swoole HttpServer 事件, start shutdown connect close workerStart workerStop workerError request
        // ...
        $this->httpServer->start();
    }

    public function onRequest(\swoole_http_request $req, \swoole_http_response $res)
    {
        $ctx = $this->createContext($req, $res);
        $reqHandler = $this->makeRequestHandler($ctx);
        $resHandler = $this->makeResponseHandler($ctx);
        spawn($reqHandler, $resHandler);
    }

    protected function makeRequestHandler(Context $ctx)
    {
        return function() use($ctx) {
            yield setCtx("ctx", $ctx);
            $ctx->res->status(404);
            $fn = $this->fn;
            yield $fn($ctx);
        };
    }

    protected function makeResponseHandler(Context $ctx)
    {
        return function($r = null, \Exception $ex = null) use($ctx) {
            if ($ex) {
                $this->handleError($ctx, $ex);
            } else {
                $this->respond($ctx);
            }
        };
    }

    protected function handleError(Context $ctx, \Exception $ex = null)
    {
        if ($ex === null) {
            return;
        }

        if ($ex && $ex->getCode() !== 404) {
            sys_error($ctx);
            sys_error($ex);
        }

        // 非 Http异常, 统一500 status, 对外显示异常code
        // Http 异常, 自定义status, 自定义是否暴露Msg
        $msg = $ex->getCode();
        if ($ex instanceof HttpException) {
            $status = $ex->status ?: 500;
            $ctx->res->status($status);
            if ($ex->expose) {
                $msg = $ex->getMessage();
            }
        } else {
            $ctx->res->status(500);
        }

        // force text/plain
        $ctx->res->header("Content-Type", "text"); // TODO accepts
        $ctx->res->write($msg);
        $ctx->res->end();
    }

    protected function respond(Context $ctx)
    {
        if ($ctx->respond === false) return; // allow bypassing Koa

        $body = $ctx->body;
        $code = $ctx->status;

        if ($code !== null) {
            $ctx->res->status($code);
        }
        // status.empty() $ctx->body = null; res->end()

        if ($body !== null) {
            $ctx->res->write($body);
        }

        $ctx->res->end();
    }

    protected function createContext(\swoole_http_request $req, \swoole_http_response $res)
    {
        // 可以在Context挂其他组件 $app->foo = bar; $app->listen();
        $context = clone $this->context;

        $request = $context->request = new Request($this, $context, $req, $res);
        $response = $context->response = new Response($this, $context, $req, $res);

        $context->app = $this;
        $context->req = $req;
        $context->res = $res;

        $request->response = $response;
        $response->request = $request;

        $request->originalUrl = $req->server["request_uri"];
        $request->ip = $req->server["remote_addr"];

        return $context;
    }
}

```