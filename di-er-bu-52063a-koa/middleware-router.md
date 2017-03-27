## Middleware: Router

路由是httpServer必不可少的组件，我们使用nikic的[FastRoute](https://github.com/nikic/FastRoute)来实现一个路由中间件：


```php
<?php

use FastRoute\Dispatcher;
use FastRoute\RouteCollector;
use Minimalism\A\Server\Http\Context;

// 这里继承FastRoute的RouteCollector
class Router extends RouteCollector
{
    public $dispatcher;

    public function __construct()
    {
        $routeParser = new \FastRoute\RouteParser\Std();
        $dataGenerator = new \FastRoute\DataGenerator\GroupCountBased();
        parent::__construct($routeParser, $dataGenerator);
    }

    // 返回路由中间件
    public function routes()
    {
        $this->dispatcher = new \FastRoute\Dispatcher\GroupCountBased($this->getData());
        return [$this, "dispatch"];
    }

    // 路由中间件的主要逻辑
    public function dispatch(Context $ctx, $next)
    {
        if ($this->dispatcher === null) {
            $this->routes();
        }

        $uri = $ctx->url;
        if (false !== $pos = strpos($uri, '?')) {
            $uri = substr($uri, 0, $pos);
        }
        $uri = rawurldecode($uri);

        // 从Context提取method与url进行分发
        $routeInfo = $this->dispatcher->dispatch(strtoupper($ctx->method), $uri);
        switch ($routeInfo[0]) {
            case Dispatcher::NOT_FOUND:
                // 状态码写入Context
                $ctx->status = 404;
                yield $next;
                break;
            case Dispatcher::METHOD_NOT_ALLOWED:
                $ctx->status = 405;
                break;
            case Dispatcher::FOUND:
                $handler = $routeInfo[1];
                $vars = $routeInfo[2];

                // 从路由表提取处理器
                yield $handler($ctx, $next, $vars);
                break;
        }
    }
}


$router = new Router();
$router->get('/user/{id:\d+}', function(Context $ctx, $next, $vars) {
    $ctx->body = "user={$vars['id']}";
});

// $route->post('/post-route', 'post_handler');
$router->addRoute(['GET', 'POST'], '/test', function(Context $ctx, $next, $vars) {
    $ctx->body = "";
});

// 分组路由
$router->addGroup('/admin', function (RouteCollector $router) {
    // handler :: (Context $ctx, $next, array $vars) -> void
    $router->addRoute('GET', '/do-something', 'handler');
    $router->addRoute('GET', '/do-another-thing', 'handler');
    $router->addRoute('GET', '/do-something-else', 'handler');
});

$app->υse($router->routes());

```

我们已经拥有了一个支持`多方法` `正则` `参数匹配` `分组`功能的路由中间件。
