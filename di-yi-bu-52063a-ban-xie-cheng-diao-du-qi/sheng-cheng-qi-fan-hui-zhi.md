## 生成器返回值

PHP7支持通过[`Generator::getReturn`](http://php.net/manual/en/generator.getreturn.php)获取生成器方法return的返回值。

PHP5中我们约定使用Generator最后一次yield值作为返回值，为最终需要嵌套Generator的返回值做铺垫。

```php
<?php

final class AsyncTask
{
    public function begin()
    {
        return $this->next();
    }

    // 添加return传递每一次迭代的结果，直到向上传递到begin
    public function next($result = null)
    {
        $value = $this->gen->send($result);

        if ($this->gen->valid()) {
            return $this->next($value);
        } else {
            return $result;
        }
    }
}

function newGen()
{
    $r1 = (yield 1);
    $r2 = (yield 2);
    echo $r1, $r2;
    yield 3;
}
$task = new AsyncTask(newGen());
$r = $task->begin(); // output: 12
echo $r; // output: 3
```