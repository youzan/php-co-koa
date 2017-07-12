# 第一部分: 半协程调度器

谈及koa(1.x)首先得说co，co与Promise是JSer在解决回调地狱(callback hell)问题前仆后继的众多产物之一。

co其实是Generator的自动执行器(半协程调度器): 通过yield显式操纵控制流让我们可以做到以近乎同步的方式书写非阻塞代码。

Promise是一套比较完善的方案，但关于如何实现Promise本身超出本文范畴， 且PHP没有大量异步接口的历史包袱需要thunks方案做转换。

综上所述， 我们的调度器仅基于一个简单的接口，来抽象异步任务：

```php
<?php 
interface Async
{
    /**
     * 开启异步任务，完成时执行回调，任务结果或异常通过回调参数传递
     * @param callable $callback
     *      continuation :: (mixed $result = null, \Exception|null $ex = null)
     * @return void 
     */
    public function begin(callable $callback);
}
```

1. 对co库不了解的同学可以先参考[阮一峰 - co 函数库的含义和用法](http://www.ruanyifeng.com/blog/2015/05/co.html)
2. co新版与旧版的区别在于对thunks的支持， 4.x只支持Promises.

我们首先构建Koa的基础设施，渐进的实现一个约50+行代码的精练的半协程调度器。
