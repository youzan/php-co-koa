## 说明

1. 下文中协程均指代使用生成器实现的半协程，具体概念参见[Wiki](https://en.wikipedia.org/wiki/Coroutine)。
2. 下文中耗时任务指代I/O或定时器，非CPU计算。
3. `广告` 继TSF之后，我司去年了开源[Zan Framework](http://zanphp.io/)，内部的半协程调度器已经解决了swoole中回调接口的代码书写问题。
4. 下文实例代码，限于篇幅，每部分仅呈现改动部分， 其余省略。