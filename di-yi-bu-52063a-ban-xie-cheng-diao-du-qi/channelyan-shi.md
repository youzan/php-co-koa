## channel演示

这是我们最终得到的接口：


```php
<?php

function go()
{
    spawn(...func_get_args());
}

function chan($n = 0)
{
    if ($n === 0) {
        return new Channel();
    } else {
        return new BufferChannel($n);
    }
}
```


第一个典型例子, PINGPONG

与golang的channel类似, 我们可以在两个semicoroutine之间做同步


```php
<?php

// 构建两个单向channel, 我们只单向收发数据

$pingCh = chan();
$pongCh = chan();

go(function() use($pingCh, $pongCh) {
    while (true) {
        echo (yield $pingCh->recv());
        yield $pongCh->send("PONG\n");

        // 递归调度器实现, 需要引入异步的方法退栈, 否则Stack Overflow...
        // 或者考虑将send或者recv以defer方式实现
        yield async_sleep(1);
    }
});
    
go(function() use($pingCh, $pongCh) {
    while (true) {
        echo (yield $pongCh->recv());
        yield $pingCh->send("PING\n");

        yield async_sleep(1);
    }
});

// start up
go(function() use($pingCh) {
    echo "start up\n";
    yield $pingCh->send("PING");
});

// output:
/*
start up
PING
PONG
PING
PONG
PING
PONG
PING
...
*/

```

当然, 我们可以很轻易构建一个生产者-消费者模型


```php
<?php

$ch = chan();

// 生产者1
go(function() use($ch) {
    while (true) {
        yield $ch->send("producer 1");
        yield async_sleep(1000);
    }
});

// 生产者2
go(function() use($ch) {
    while (true) {
        yield $ch->send("producer 2");
        yield async_sleep(1000);
    }
});

// 消费者1
go(function() use($ch) {
    while (true) {
        $recv = (yield $ch->recv());
        echo "consumer1: $recv\n";
    }
});

// 消费者2
go(function() use($ch) {
    while (true) {
        $recv = (yield $ch->recv());
        echo "consumer2: $recv\n";
    }
});

// output:
/*
consumer1 recv from producer 1
consumer1 recv from producer 2
consumer1 recv from producer 1
consumer2 recv from producer 2
consumer1 recv from producer 2
consumer2 recv from producer 1
consumer1 recv from producer 1
consumer2 recv from producer 2
consumer1 recv from producer 2
consumer2 recv from producer 1
consumer1 recv from producer 1
consumer2 recv from producer 2
......
*/

```

chan 自身是first-class, 所以可传递


```php
<?php

// 我们通过一个chan来发送另一个chan
// 然后等待接收到这个chan的semicoroutine回送数据

$ch = chan();

go(function() use ($ch) {
    $anotherCh = chan();
    yield $ch->send($anotherCh);
    echo "send another channel\n";
    yield $anotherCh->send("HELLO");
    echo "send hello through another channel\n";
});

go(function() use($ch) {
    /** @var Channel $anotherCh */
    $anotherCh = (yield $ch->recv());
    echo "recv another channel\n";
    $val = (yield $anotherCh->recv());
    echo $val, "\n";
});

// output:
/*
send another channel
recv another channel
send hello through another channel
HELLO
*/
```

我们通过控制channel缓存大小 观察输出结果

```php
<?php
$ch = chan($n);

go(function() use($ch) {
    $recv = (yield $ch->recv());
    echo "recv $recv\n";
    $recv = (yield $ch->recv());
    echo "recv $recv\n";
    $recv = (yield $ch->recv());
    echo "recv $recv\n";
    $recv = (yield $ch->recv());
    echo "recv $recv\n";
});

go(function() use($ch) {
    yield $ch->send(1);
    echo "send 1\n";
    yield $ch->send(2);
    echo "send 2\n";
    yield $ch->send(3);
    echo "send 3\n";
    yield $ch->send(4);
    echo "send 4\n";
});

// $n = 1;
// output:
/*
send 1
recv 1
send 2
recv 2
send 3
recv 3
send 4
recv 4
*/

// $n = 2;
// output:
/*
send 1
send 2
recv 1
recv 2
send 3
send 4
recv 3
recv 4
*/

// $n = 3;
// output:
/*
send 1
send 2
send 3
recv 1
recv 2
recv 3
send 4
recv 4
*/

```


一个更具体的生产者消费者的例子:


```php
<?php

// 缓存两个结果
$ch = chan(2);

// 从channel接口请求写过写文件
go(function() use($ch) {
    $file = new AsyncFile("path/to/save");
    while (true) {
        list($host, $status) = (yield $ch->recv());
        yield $file->write("$host: $status\n");
    }
});

// 请求并写入chan
go(function() use($ch) {
    while (true) {
        $host = "www.baidu.com";
        $resp = (yield async_curl_get($host));
        yield $ch->send([$host, $resp->statusCode]);
    }
});

// 请求并写入chan
go(function() use($ch) {
    while (true) {
        $host = "www.bing.com";
        $resp = (yield async_curl_get($host));
        yield $ch->send([$host, $resp->statusCode]);
    }
});

// output:
```


channel的发送与接受没有超时机制, golang可以select多个chan实现超时处理,

我们可以做一个select设施, 或者在send于recv接受直接添加超时参数, 扩展接口功能,

留待读者自行实现. 