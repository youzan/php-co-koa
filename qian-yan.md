# 前言

近年来，在面向高并发编程的道路上，Node.js与Golang风生水起，让人们渐渐把目光从多线程模型转移到callback与CSP/Actor上，用惯了FPM多进程同步阻塞模型的PHPer中总难免有人心
动。多种EventLoop一直不温不火，而国内以swoole为代表，直接以扩展形式，提供了整套callback模型的PHP异步编程解决方案，正在逐渐地流行起来。

Node.js在JS上开花结果，也许是浏览器的DOM事件模型培养起来的callback书写习惯，与语言自身的函数式特性适合callback代码编写。但回调固有的逻辑割裂、调试维护难的问题随着node社区的繁荣逐渐显现，从老赵脑洞大开的windjs到co与Promise，方案层出不穷，最终[Promise](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)被
采纳为官方「异步编程标准规范」，从C#借鉴过来的[async/await](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/async_function)被纳入语言标准。

因swoole与Node.js的I/O模型相同，PHPer有幸在高并发命题上遭遇与node一样的问题。Closure([RFC](https://wiki.php.net/rfc/closures?cm_mc_uid=26754990333314676210612&cm_mc_sid_50200000=1490031947))一定程度从语言本身改善了异步编程的体验，受限于Zend引擎作用域实现机制，PHP因缺失词法作用域从而缺失词法闭包，Closure对象采用了use语法来显式捕获upValue到静态属性的方式(closure->func.op_array.static_variables)，我个人认为这有点像无法自动实现闭包的匿名函数。之后Nikita Popov在PHP中实现了Generator([RFC](https://wiki.php.net/rfc/generators))，并且让PHPer意
识到生成器原来可以实现非抢占任务调度([译文:在PHP中使用协程实现多任务调度](http://www.laruence.com/2015/05/28/3038.html))。我们最终可以借助于生成器实现半协程来解决该问题。

这篇文章秉承着造轮子的精神，我们从头实现一个全功能的基于生成器(Generator)的半协程调度器与相关基础组件，并基于该调度器实(chao)现(xi)JS社区当红的koa框架，最终加深我们对异步编程的理解。