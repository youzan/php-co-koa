## 其他有趣的组件

异步回调API是无法直接使用yield语法的，需要使用thunk或者promise进行转换，thunkfy就是将多参数函数替换成单参数函数且参数只接受回调(参见：[Thunk 函数的含义和用法](http://www.ruanyifeng.com/blog/2015/05/thunk.html))。上文中我们将回调API显式实现Async接口，显得有些麻烦，这里可以把 **“通过参数传递异步结果回调度器”** 这个模式抽象出来，实现一个穷人的call/cc。