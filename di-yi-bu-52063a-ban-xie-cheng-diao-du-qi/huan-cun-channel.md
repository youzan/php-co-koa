## 缓存channel

接下来我们来实现带缓存的Channel:

> Sends to a buffered channel block only when the buffer is full. 

> Receives block when the buffer is empty.


```php
<?php

class BufferChannel
{
    // 缓存容量
    public $cap;

    // 缓存
    public $queue;

    // 同无缓存Channel
    public $recvCc;

    // 同无缓存Channel
    public $sendCc;

    public function __construct($cap)
    {
        assert($cap > 0);
        $this->cap = $cap;
        $this->queue = new \SplQueue();
        $this->sendCc = new \SplQueue();
        $this->recvCc = new \SplQueue();
    }

    public function recv()
    {
        return callcc(function($cc) {
            if ($this->queue->isEmpty()) {

                // 当无数据可接收时，$cc入列,让出控制流，挂起接收者协程
                $this->recvCc->enqueue($cc);

            } else {

                // 当有数据可接收时，先接收数据，然后恢复控制流
                $val = $this->queue->dequeue();
                $this->cap++;
                $cc($val, null);

            }

            // 递归唤醒其他被阻塞的发送者与接收者收发数据，注意顺序
            $this->recvPingPong();
        });
    }

    public function send($val)
    {
        return callcc(function($cc) use($val) {
            if ($this->cap > 0) {

                // 当缓存未满，发送数据直接加入缓存，然后恢复控制流
                $this->queue->enqueue($val);
                $this->cap--;
                $cc(null, null);

            } else {

                // 当缓存满，发送者控制流与发送数据入列，让出控制流,挂起发送者协程
                $this->sendCc->enqueue([$cc, $val]);

            }

            // 递归唤醒其他被阻塞的接收者与发送者收发数据，注意顺序
            $this->sendPingPong();

            // 如果全部代码都为同步，防止多个发送者时，数据全部来自某个发送者
            // 应该把sendPingPong 修改为异步执行 defer([$this, "sendPingPong"]);
            // 但是swoole本身的defer实现有bug，除非把defer 实现为swoole_timer_after(1, ...)
            // recvPingPong 同理
        });
    }

    public function recvPingPong()
    {
        // 当有阻塞的发送者，唤醒其发送数据
        if (!$this->sendCc->isEmpty() && $this->cap > 0) {
            list($sendCc, $val) = $this->sendCc->dequeue();
            $this->queue->enqueue($val);
            $this->cap--;
            $sendCc(null, null);

            // 当有阻塞的接收者，唤醒其接收数据
            if (!$this->recvCc->isEmpty() && !$this->queue->isEmpty()) {
                $recvCc = $this->recvCc->dequeue();
                $val = $this->queue->dequeue();
                $this->cap++;
                $recvCc($val);

                $this->recvPingPong();
            }
        }
    }

    public function sendPingPong()
    {
        // 当有阻塞的接收者，唤醒其接收数据
        if (!$this->recvCc->isEmpty() && !$this->queue->isEmpty()) {
            $recvCc = $this->recvCc->dequeue();
            $val = $this->queue->dequeue();
            $this->cap++;
            $recvCc($val);

            // 当有阻塞的发送者，唤醒其发送数据
            if (!$this->sendCc->isEmpty() && $this->cap > 0) {
                list($sendCc, $val) = $this->sendCc->dequeue();
                $this->queue->enqueue($val);
                $this->cap--;
                $sendCc(null, null);

                $this->sendPingPong();
            }
        }
    }
}
```