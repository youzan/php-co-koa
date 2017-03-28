## 生成器委托

PHP7中支持[`delegating generator`](https://wiki.php.net/rfc/generator-delegation)，可以自动展开`subgenerator`;

> A “subgenerator” is a Generator used in the <expr> portion of the yield from <expr> syntax.

我们需要在PHP5支持子生成器，将子生成器最后yield值作为父生成器yield表达式结果，仅只需要加两行代码，递归的产生一个`AsyncTask`对象来执行子生成器即可。

```php
<?php

final class AsyncTask
{
    public function next($result = null)
    {
        $value = $this->gen->send($result);

        if ($this->gen->valid()) {
            if ($value instanceof \Generator) {
                $value = (new self($value))->begin();
            }
            return $this->next($value);

        } else {
            return $result;
        }
    }
}
```

```php
<?php
function newSubGen()
{
    yield 0;
    yield 1;
}

function newGen()
{
    $r1 = (yield newSubGen());
    $r2 = (yield 2);
    echo $r1, $r2;
    yield 3;
}
$task = new AsyncTask(newGen());
$r = $task->begin(); // output: 12
echo $r; // output: 3

```
