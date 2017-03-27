## rightReduce与中间件compose


然后, 有心同学可能会发现, 我们的makeTravel其实是一个对函数列表进行rightReduce的函数.

```php
<?php

function compose(...$fns)
{
    return array_right_reduce($fns, function($carry, $fn) {
        return function() use($carry, $fn) {
            $fn($carry);
        };
    });
}

function array_right_reduce(array $input, callable $function, $initial = null)
{
    return array_reduce(array_reverse($input, true), $function, $initial);
}

```
将上述makeTravel替换成compose, 我们将得到完全相同的结果.


------------------------------------------------------


我们修改函数列表中函数的签名, middleware :: (Context $ctx, Generator $next) -> void,

我们把满足List<Middleware>的函数列表进行组合(rightReduce), 得到中间件入口函数,

只需要稍加改造, 我们的compose方法便可以处理生成器函数, 用来支持我们上文的半协程.

```php
<?php
function compose(...$middleware)
{
    return function(Context $ctx = null) use($middleware) {
        $ctx = $ctx ?: new Context(); // Context 参见下文
        $next = null;
        $i = count($middleware);
        while ($i--) {
            $curr = $middleware[$i];
            $next = $curr($ctx, $next);
            assert($next instanceof \Generator);
        }
        return $next;
    };
}
```

如果是中间件是\Closure, 打包过程可以可以将$this变量绑定到$ctx, 

中间件内可以使用$this代替$ctx, Koa1.x采用这种方式, Koa2.x已经废弃;

```
if ($middleware[$i] instanceof \Closure) {
    // 
    $curr = $middleware[$i]->bindTo($ctx, Context::class);
}
```

rightReduce版本

```php
<?php
function compose(array $middleware)
{
    return function(Context $ctx = null) use($middleware) {
        $ctx = $ctx ?: new Context(); // Context 参见下文        
        return array_right_reduce($middleware, function($rightNext, $leftFn) use($ctx) {
            return $leftFn($ctx, $rightNext);
        }, null);
    };
}
```