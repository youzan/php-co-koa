## all与parallel

Any表示多个异步回调，任一回调完成则任务完成，All表示等待所有回调均执行完成才算任务完成，二者相同点是IO部分并发执行;

```php
<?php

class All implements Async
{
    public $parent;
    public $tasks;
    public $continuation;

    public $n;
    public $results;
    public $done;

    public function __construct(array $tasks, AsyncTask $parent = null)
    {
        $this->tasks = $tasks;
        $this->parent = $parent;
        $this->n = count($tasks);
        assert($this->n > 0);
        $this->results = [];
    }

    public function begin(callable $continuation = null)
    {
        $this->continuation = $continuation;
        foreach ($this->tasks as $id => $task) {
            (new AsyncTask($task, $this->parent))->begin($this->continuation($id));
        };
    }

    private function continuation($id)
    {
        return function($r, $ex = null) use($id) {
            if ($this->done) {
                return;
            }

            // 任一回调发生异常，终止任务
            if ($ex) {
                $this->done = true;
                $k = $this->continuation;
                $k(null, $ex);
                return;
            }

            $this->results[$id] = $r;
            if (--$this->n === 0) {
                // 所有回调完成，终止任务
                $this->done = true;
                if ($this->continuation) {
                    $k = $this->continuation;
                    $k($this->results);
                }
            }
        };
    }
}


function all(array $tasks)
{
    $tasks = array_map(__NAMESPACE__ . "\\await", $tasks);

    return new Syscall(function(AsyncTask $parent) use($tasks) {
        if (empty($tasks)) {
            return null;
        } else {
            return new All($tasks, $parent);
        }
    });
}
```

```php
<?php
spawn(function() {
    $ex = null;
    try {
        $r = (yield all([
            async_dns_lookup("www.bing.com", 100),
            async_dns_lookup("www.so.com", 100),
            async_dns_lookup("www.baidu.com", 100),
        ]));
        var_dump($r);

/*
array(3) {
  [0]=>
  string(14) "202.89.233.103"
  [1]=>
  string(14) "125.88.193.243"
  [2]=>
  string(15) "115.239.211.112"
}
*/
    } catch (\Exception $ex) {
        echo $ex;
    }
});
```


我们这里实现了与Promise.all相同语义的接口，或者更复杂一些，我们也可以实现批量任务以chunk方式进行作业的接口，留待读者自己完成;

至此， 我们已经拥有了 `spawn` `callcc` `race` `all` `timeout`.