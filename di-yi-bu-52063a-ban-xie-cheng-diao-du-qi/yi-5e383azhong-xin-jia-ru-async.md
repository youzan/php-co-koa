## 异常: 重新加入Async

重新加入Async，修改continuation的签名，加入异常参数：

```php
<?php
interface Async
{
    // continuation :: (mixed $r， \Exception $ex) -> void
    public function begin(callable $continuation);
}

final class AsyncTask implements Async
{
    public function next($result = null, $ex = null)
    {
        try {
            // ...
            if ($this->gen->valid()) {
                if ($value instanceof \Generator) {
                    $value = new self($value);
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
            // ...
        }
    }
}
```

```php
<?php
$trace = function($r, $ex) {
    if ($ex instanceof \Exception) {
        echo "cc_ex:" . $ex->getMessage(), "\n";
    }
};

class AsyncException implements Async
{
    public function begin(callable $cc)
    {
        swoole_timer_after(1000, function() use($cc) {
            $cc(null, new \Exception("timeout"));
        });
    }
}

function newSubGen()
{
    yield 0;
    $async = new AsyncException();
    yield $async;
}

function newGen($try)
{
    $start = time();
    try {
        $r1 = (yield newSubGen());
    } catch (\Exception $ex) {
        // 捕获subgenerator抛出的异常
        if ($try) {
            echo "catch:" . $ex->getMessage(), "\n";
        } else {
            throw $ex;
        }
    }
    echo time() - $start, "\n";
}

// 内部try-catch异常
$task = new AsyncTask(newGen(true));
$task->begin($trace);
// output:
// catch:timeout
// 1

// 异常传递至AsyncTask的最终回调
$task = new AsyncTask(newGen(false));
$task->begin($trace);
// output:
// cc_ex:timeout
```
