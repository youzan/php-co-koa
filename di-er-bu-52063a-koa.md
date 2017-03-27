#
第二部分: Koa

Koa自述是下一代web框架：

> 由 Express 原班人马打造的 Koa，致力于成为一个更小、更健壮、更富有表现力的 Web 框架。
> 使用 Koa 编写 web 应用，通过组合不同的 generator，
> 可以免除重复繁琐的回调函数嵌套，并极大地提升常用错误处理效率。
> Koa 不在内核方法中绑定任何中间件，它仅仅提供了一个轻量优雅的函数库，使得编写 Web 应用变得得心应手。

像`Ruby Rack` `Python django` `Golang matini` `Node Express` `PHP Laravel` `Java Spring`，Web框架大多会有一个面向AOP的中间件模块，内部操纵Req/Res对象可选执行next动作，Koa与martini类似，都属于设计清爽的中间件web框架，采用洋葱模型middleware stack，但合并了Request与Response对象，编写更直观方便。

> Koa's middleware stack flows in a stack-like manner，
> allowing you to perform actions downstream then filter and manipulate the response upstream.
> Each middleware receives a Koa `Context` object that encapsulates an incoming
> http message and the corresponding response to that message. `ctx` is often used
> as the parameter name for the context object.

[Laravel Middleware](https://laravel.com/docs/5.4/middleware)
Laravel after方法的书写不够自然，需要中间变量保存Response; Spring的Interceptor功能强大，但编写略繁琐; Koa则简单的多，而且可以在某个中间件中灵活控制之后中间件执行时机。

其实对于web框架来讲，业务逻辑归根结底都在处理请求与相应对象，web中间件实质就是在请求与响应中间开放出来的可编排的扩展点，比如修改请求做URLRewrite，比如请求日志，身份验证...

真正的业务逻辑都可以通过middleware实现，或者说按特定顺序对中间件灵活组合编排。

我们基于PHP5.6与yz-swoole(有赞内部自研稳定版本的swoole，17年6月即将开源，敬请期待)，用少量的代码即可实(chao)现(xi)一个php-koa。

Koa2.x决定全面使用async/await，受限于PHP语法，我们这里仅实现Koa1.x，将Generator作为middleware实现形式。