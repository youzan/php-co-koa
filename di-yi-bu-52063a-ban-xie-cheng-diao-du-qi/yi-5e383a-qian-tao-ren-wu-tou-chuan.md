## 异常: 嵌套任务透传

重新处理生成器嵌套，需要将子生成器异常抛向父生成器。

当生成器迭代过程发生未捕获异常，生成器将会被关闭，`Generator::valid`返回false，未捕获异常会从生成器内部被抛向父作用域，嵌套子生成器内部的未捕获异常必须最终被抛向根生成器的calling frame，PHP7中`yield-from`对嵌套子生成器resume时产生的异常，采取`goto try_again传递`+`while`方式层层向上抛出，我们的代码因为递归迭代的原因，未捕获异常需要逆递归栈帧方向层层上抛，性能方面有改进余地。

```php
<?php

final class AsyncTask
{
    public function next($result = null, \Exception $ex = null)
    {
        try {
            if ($ex) {
                // c. 直接抛出异常
                // $ex来自子生成器， 调用父生成器throw抛出
                // 这里实现了 try { yield \Generator; } catch(\Exception $ex) { }
                // echo "c -> ";
                $value = $this->gen->throw_($ex);
            } else {
                // a2. 当前生成器可能抛出异常
                // echo "a2 -> ";
                $value = $this->gen->send($result);
            }

            if ($this->gen->valid()) {
                if ($value instanceof \Generator) {
                    // a3. 子生成器可能抛出异常
                    // echo "a3 -> ";
                    $value = (new self($value))->begin();
                }
                // echo "a4 -> ";
                return $this->next($value);
            } else {
                return $result;
            }
        } catch (\Exception $ex) {
            // !! 当生成器迭代过程发生未捕获异常， 生成器将会被关闭， valid()返回false,
            if ($this->gen->valid()) {
                // b1. 所以， 当前分支的异常一定不是当前生成器所抛出， 而是来自嵌套的子生成器
                // 此处将子生成器异常通过(c)向当前生成器抛出异常
                // echo "b1 -> ";
                return $this->next(null, $ex);
            } else {
                // b2. 逆向(递归栈帧)方向向上抛 或者 向父生成器(如果存在)抛出异常
                // echo "b2 -> ";
                throw $ex;
            }
        }
    }
}
```

```php
<?php
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
        echo $ex->getMessage();
    }
    $r2 = (yield 2);
    yield 3;
}
$task = new AsyncTask(newGen());
$r = $task->begin(); // output: e
echo $r; // output: 3

```
