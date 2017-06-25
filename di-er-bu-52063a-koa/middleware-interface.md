## Middleware Interface

我们为Middleware声明一个接口，让稍微复杂一点的中间件以类的形式存在， 实现该接口的类实例自身满足callable类型，我们的中间件接受任何callable，所以，这并不是必须的，仅仅是为了更好组织代码，且对PSR4的autoload友好;

```php
<?php

interface Middleware
{
    public function __invoke(Context $ctx, $next);
}
```

函数与类并没有孰优孰劣：

> Objects are state data with attached behavior; 

> Closures are behaviors with attached state data and without the overhead of classes.

我们可以在对象形式的中间件附加更多的状态, 以应对复杂的场景。
