## spawn

为了易用性，我们为`AsyncTask`的创建一个可灵活传递参数函数入口。

```php
<?php

/**
 * spawn one semicoroutine
 *
 * @internal param callable|\Generator|mixed $task
 * @internal param callable $continuation function($r = null, $ex = null) {}
 * @internal param AsyncTask $parent
 * @internal param array $ctx Context可以附加在 \Generator 对象的属性上
 *
 *  第一个参数为task
 *  剩余参数(优先检查callable)
 *      如果参数类型 callable 则参数被设置为 Continuation
 *      如果参数类型 AsyncTask 则参数被设置为 ParentTask
 *      如果参数类型 array 则参数被设置为 Context
 */
function spawn()
{
    $n = func_num_args();
    if ($n === 0) {
        return;
    }

    $task = func_get_arg(0);
    $continuation = function() {};
    $parent = null;
    $ctx = [];

    for ($i = 1; $i < $n; $i++) {
        $arg = func_get_arg($i);
        if (is_callable($arg)) {
            $continuation = $arg;
        } else if ($arg instanceof AsyncTask) {
            $parent = $arg;
        } else if (is_array($arg)) {
            $ctx = $arg;
        }
    }

    if (is_callable($task)) {
        try {
            $task = $task();
        } catch (\Exception $ex) {
            $continuation(null, $ex);
            return;
        }
    }

    if ($task instanceof \Generator) {
        foreach ($ctx as $k => $v) {
            $task->$k = $v;
        }
        (new AsyncTask($task, $parent))->begin($continuation);
    } else {
        $continuation($task, null);
    }
}
```