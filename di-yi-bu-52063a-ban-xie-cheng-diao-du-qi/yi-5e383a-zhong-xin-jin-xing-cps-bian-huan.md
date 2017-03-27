## 异常: 重新进行CPS变换

我们把加入异常处理的代码重新修改为CPS方式;

```php
<?php
final class AsyncTask
{
    public $continuation;

    public function begin(callable $continuation)
    {
        $this->continuation = $continuation;
        $this->next();
    }

    public function next($result = null, \Exception $ex = null)
    {
        try {
            if ($ex) {
                $value = $this->gen->throw_($ex);
            } else {
                $value = $this->gen->send($result);
            }

            if ($this->gen->valid()) {
                if ($value instanceof \Generator) {
                    // 注意这里
                    $continuation = [$this, "next"];
                    (new self($value))->begin($continuation);
                } else {
                    $this->next($value);
                }
            } else {
                // 迭代结束 返回结果
                $cc = $this->continuation; // cc指向 父生成器next方法 或 用户传入continuation
                $cc($result, null);
            }
        } catch (\Exception $ex) {
            if ($this->gen->valid()) {
                // 抛出异常
                $this->next(null, $ex);
            } else {
                // 未捕获异常
                $cc = $this->continuation; // cc指向 父生成器next方法 或 用户传入continuation
                $cc(null, $ex);
            }
        }
    }
}


function tt()
{
    yield;
    throw new \Exception("e");
}
function t()
{
    yield tt();
    yield 1;
}

$task = new AsyncTask(t());
$trace = function($r, $ex) {
    if ($ex) {
        echo $ex->getMessage(); // output: e
    } else {
        echo $r;
    }
};
$task->begin($trace);



function newSubGen()
{
    yield 0;
    throw new \Exception("e");
    yield 1;
}

function newGen()
{
    try {
        $r1 = (yield newSubGen());
    } catch (\Exception $ex) {
        echo $ex->getMessage(); // output: e
    }
    $r2 = (yield 2);
    yield 3;
}
$task = new AsyncTask(newGen());
$trace = function($r, $ex) {
    if ($ex) {
        echo $ex->getMessage();
    } else {
        echo $r; // output: 3
    }
};
$task->begin($trace); // output: e

```