## channel与协程间通信

虽然已经构建了基于yield的半协程, 之前所有讨论都集中在单个协程, 我们可以再深入一步,

构造带有阻塞语义的协程间通信原语--channel, 这里按照golang的channel来实现;

[playground](https://tour.golang.org/concurrency/2)

> By default, sends and receives block until the other side is ready.

> This allows goroutines to synchronize without explicit locks or condition variables.

相比golang, 我们因为只有一个线程, 对于chan发送与接收的阻塞的处理, 

最终会被转换为对使用chan的协程的控制流的控制.