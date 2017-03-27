## 抽象异步模型

对回调模型抽象出异步接口Async;

只有一个方法的接口通常都可以使用闭包代替，区别在于interface引入新类型，闭包则不会。


```php
<?php

interface Async
{
    public function begin(callable $callback);
}

// AsyncTask符合Async定义， 实现Async
final class AsyncTask implements Async
{
    public function next($result = null)
    {
        $value = $this->gen->send($result);

        if ($this->gen->valid()) {
            // \Generator -> Async
            if ($value instanceof \Generator) {
                $value = new self($value);
            }

            if ($value instanceof Async) {
                $async = $value;
                $continuation = [$this, "next"];
                $async->begin($continuation);
            } else {
                $this->next($value);
            }

        } else {
            $cc = $this->continuation;
            $cc($result);
        }
    }
}
```

两个简单的对回调接口转换例子:

```php
<?php

// 定时器修改为标准异步接口
class AsyncSleep implements Async
{
    public function begin(callable $cc)
    {
        swoole_timer_after(1000, $cc);
    }
}

// 异步dns查询修改为标准异步接口
class AsyncDns implements Async
{
    public function begin(callable $cc)
    {
        swoole_async_dns_lookup("www.baidu.com", function($host, $ip) use($cc) {
            // 这里我们会发现， 通过call $cc， 将返回值作为参数进行传递， 与callcc相像
            // $ip 通过$cc 从子生成器传入父生成器， 最终通过send方法成为yield表达式结果
            $cc($ip);
        });
    }
}

function newGen()
{
    $r1 = (yield newSubGen());
    $r2 = (yield 2);
    $start = time();
    yield new AsyncSleep();
    echo time() - $start, "\n";
    $ip = (yield new AsyncDns());
    yield "IP: $ip";
}
$task = new AsyncTask(newGen());

$trace = function($r) { echo $r; };
$task->begin($trace);
// output:
// 1
// IP: 115.239.210.27

```
