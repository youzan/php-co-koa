## FutureTask与fork

在多线程代码中我们经常会遇到这种模型，将一个耗时任务， new一个新的Thread或者通常放到线程池后台执行，当前线程执行另外任务，之后通过某个api接口阻塞获取后台任务结果。

Java童鞋应该对这个概念非常熟悉——JDK给予直接支持的Future。

同样的模型我们可以利用介绍过channel对多个协程做进行同步来实现， 代码很简单:


```php
<?php

go(function() {
    $start = microtime(true);

    $ch = chan();

    // 开启一个新的协程, 异步执行耗时任务
    spawn(function() use($ch) {
        yield delay(1000);
        yield $ch->send(42); // 通知+传送结果
    });

    yield delay(500);
    $r = (yield $ch->recv()); // 阻塞等待结果
    echo $r; // 42

    // 我们这里两个耗时任务并发执行, 总耗时约1000ms
    echo "cost ", microtime(true) - $start, "\n";
});

```

事实上我们也很容易把Future模型移植过来

```php
<?php
final class FutureTask
{
    const PENDING = 1;
    const DONE = 2;
    private $cc;

    public $state;
    public $result;
    public $ex;

    public function __construct(\Generator $gen)
    {
        $this->state = self::PENDING;

        $asyncTask = new AsyncTask($gen);
        $asyncTask->begin(function($r, $ex = null)  {
            $this->state = self::DONE;
            if ($cc = $this->cc) {
                // 有cc, 说明有call get方法挂起协程, 在此处唤醒
                $cc($r, $ex);
            } else {
                // 无挂起, 暂存执行结果
                $this->result = $r;
                $this->ex = $ex;
            }
        });
    }

    public function get()
    {
        return callcc(function($cc) use($timeout) {
            if ($this->state === self::DONE) {
                // 获取结果时, 任务已经完成, 同步返回结果
                // 这里也可以考虑用defer实现, 异步返回结果, 首先先释放php栈, 降低内存使用
                $cc($this->result, $this->ex);
            } else {
                // 获取结果时未完成, 保存$cc, 挂起等待
                $this->cc = $cc;
            }
        });
    }
}


// helper
function fork($task, ...$args)
{
    $task = await($task); // 将task转换为生成器
    yield new FutureTask($task);
}
```


还是刚才那个例子, 我们改写为FutureTask版本:


```php
<?php
go(function() {
    $start = microtime(true);

    // fork 子协程执行耗时任务
    /** @var $future FutureTask */
    $future = (yield fork(function() {
        yield delay(1000);
        yield 42;
    }));

    yield delay(500);

    // 阻塞等待结果
    $r = (yield $future->get());
    echo $r; // 42

    // 总耗时仍旧只有1000ms
    echo "cost ", microtime(true) - $start, "\n";
});
```


再进一步, 我们扩充FutureTask的状态, 阻塞获取结果加入超时选项:


```php
<?php

final class FutureTask
{
    const PENDING = 1;
    const DONE = 2;
    const TIMEOUT = 3;

    private $timerId;
    private $cc;

    public $state;
    public $result;
    public $ex;

    // 我们这里加入新参数, 用来链接futureTask与caller父任务
    // 这样的好处比如可以共享父子任务上下文
    public function __construct(\Generator $gen, AsyncTask $parent = null)
    {
        $this->state = self::PENDING;

        if ($parent) {
            $asyncTask = new AsyncTask($gen, $parent);
        } else {
            $asyncTask = new AsyncTask($gen);
        }

        $asyncTask->begin(function($r, $ex = null)  {
            // PENDING or TIMEOUT
            if ($this->state === self::TIMEOUT) {
                return;
            }

            // PENDING -> DONE
            $this->state = self::DONE;

            if ($cc = $this->cc) {
                if ($this->timerId) {
                    swoole_timer_clear($this->timerId);
                }
                $cc($r, $ex);
            } else {
                $this->result = $r;
                $this->ex = $ex;
            }
        });
    }

    // 这里超时时间0为永远阻塞,
    // 否则超时未获取到结果, 将向父任务传递超时异常
    public function get($timeout = 0)
    {
        return callcc(function($cc) use($timeout) {
            // PENDING or DONE
            if ($this->state === self::DONE) {
                $cc($this->result, $this->ex);
            } else {
                // 获取结果时未完成, 保存$cc, 开启定时器(如果需要), 挂起等待
                $this->cc = $cc;
                $this->getResultTimeout($timeout);
            }
        });
    }

    private function getResultTimeout($timeout)
    {
        if (!$timeout) {
            return;
        }

        $this->timerId = swoole_timer_after($timeout, function() {
            assert($this->state === self::PENDING);
            $this->state = self::TIMEOUT;
            $cc = $this->cc;
            $cc(null, new AsyncTimeoutException());
        });
    }
}
```

因为引入parentTask参数, 需要将父任务隐式传参, 

而我们执行通过Syscall与执行当前生成器的父任务交互,

所以我们重写fork辅助函数, 改用Syscall实现:


```php
<?php

/**
 * @param $task
 * @return Syscall
 */
function fork($task)
{
    $task = await($task);
    return new Syscall(function(AsyncTask $parent) use($task) {
        return new FutureTask($task, $parent);
    });
}

```

下面看一些关于超时的示例:


```php
<?php

go(function() {
    $start = microtime(true);

    /** @var $future FutureTask */
    $future = (yield fork(function() {
        yield delay(500);
        yield 42;
    }));

    // 阻塞等待超时, 捕获到超时异常
    try {
        $r = (yield $future->get(100));
        var_dump($r);
    } catch (\Exception $ex) {
        echo "get result timeout\n";
    }

    yield delay(1000);

    // 因为我们只等待子任务100ms, 我们的总耗时只有 1100ms
    echo "cost ", microtime(true) - $start, "\n";
});

go(function() {
    $start = microtime(true);

    /** @var $future FutureTask */
    $future = (yield fork(function() {
        yield delay(500);
        yield 42;
        throw new \Exception();
    }));

    yield delay(1000);

    // 子任务500ms前发生异常, 已经处于完成状态
    // 我们调用get会当即引发异常
    try {
        $r = (yield $future->get());
        var_dump($r);
    } catch (\Exception $ex) {
        echo "something wrong in child task\n";
    }

    // 因为耗时任务并发执行, 这里总耗时仅1000ms
    echo "cost ", microtime(true) - $start, "\n";
});

```