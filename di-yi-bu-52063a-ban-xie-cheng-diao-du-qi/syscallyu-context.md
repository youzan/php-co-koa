## `Syscall`与`Context`

按照nikic的思路引入与调度器内部交互的Syscall：`Syscall :: AsyncTask $task -> mixed`，将需要执行的函数打包成Syscall，通过yield返回迭代器，可以从Syscall参数获取到当前迭代器对象，这里提供了一个外界与AsyncTask交互的扩展点。

我们借此演示如何添加跨生成器上下文，在嵌套生成器共享数据，解耦生成器之间依赖。

```php
<?php

final class AsyncTask implements Async
{
    public $gen;
    public $continuation;
    public $parent;

    // 我们在构造器添加$parent参数, 把父子生成器链接起来, 使其可以进行回溯.
    public function __construct(\Generator $gen, AsyncTask $parent = null)
    {
        $this->gen = new Gen($gen);
        $this->parent = $parent;
    }

    public function begin(callable $continuation)
    {
        $this->continuation = $continuation;
        $this->next();
    }

    public function next($result = null, $ex = null)
    {
        try {
            if ($ex) {
                $value = $this->gen->throw_($ex);
            } else {
                $value = $this->gen->send($result);
            }

            if ($this->gen->valid()) {
                // 这里注意优先级, Syscall 可能返回\Generator 或者 Async
                if ($value instanceof Syscall) { // Syscall 签名见下方
                    $value = $value($this);
                }

                if ($value instanceof \Generator) {
                    $value = new self($value, $this);
                }

                if ($value instanceof Async) {
                    $cc = [$this, "next"];
                    $value->begin($cc);
                } else {
                    $this->next($value, null);
                }
            } else {
                $cc = $this->continuation;
                $cc($result, null);
            }
        } catch (\Exception $ex) {
            if ($this->gen->valid()) {
                $this->next(null, $ex);
            } else {
                $cc = $this->continuation;
                $cc($result, $ex);
            }
        }
    }
}
```


```php
<?php

// Syscall将 (callable :: AsyncTask $task -> mixed) 包装成单独类型
class Syscall
{
    private $fun;

    public function __construct(callable $fun)
    {
        $this->fun = $fun;
    }

    public function __invoke(AsyncTask $task)
    {
        $cb = $this->fun;
        return $cb($task);
    }
}


// 因为PHP对象属性与数据均为Hashtable实现, 且恰巧生成器对象本身无任何属性,
// 我们这里把 我们把context kv数据附加到根生成器对象上
// 最终我们实现的 Context Get与Set函数
function getCtx($key, $default = null)
{
    return new Syscall(function(AsyncTask $task) use($key, $default) {
        while($task->parent && $task = $task->parent);
        if (isset($task->gen->generator->$key)) {
            return $task->gen->generator->$key;
        } else {
            return $default;
        }
    });
}

function setCtx($key, $val)
{
    return new Syscall(function(AsyncTask $task) use($key, $val) {
        while($task->parent && $task = $task->parent);
        $task->gen->generator->$key = $val;
    });
}



// Test
function setTask()
{
    yield setCtx("foo", "bar");
}

function ctxTest()
{
    yield setTask();
    $foo = (yield getCtx("foo"));
    echo $foo;
}

$task = new AsyncTask(ctxTest());
$task->begin($trace); // output: bar

```