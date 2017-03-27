# 附录

个人对yield与coroutine的理解与总结，有问题欢迎指正。

在上文半协程中:

1. 从抽象角度可以将「yield」理解成为CPS变换的语法糖，yield帮我们优雅的链接了异步任务序列;
2. 从控制流角度可以将「yield」理解为将程序控制权从callee(Generator)转义到caller，只有借由底层eventloop驱动，将控制权重新转移回Generator;
3. 从实现角度来看「yield」，每个生成器对象都会有自己的zend_execute_data与zend_vm_stack，调用send\next\throw方法，都需要首先备份EG中相关上下文，然后将Generator的execute_data信息恢复到EG，然后调用zend_execute_ex()从从当前上下文恢复执行执行，之后最后恢复执行前EG信息;
4. 因为ZendVM中stack与execute_data的保存与切换工作已经由Generator实现了，基于Generator构建半协程的核心问题是控制流转换;
5. yield」并没有消除回调，只是让代码从视觉上变成同步，实际仍异步执行，事实上任何异步模型，底层最后都是基于回调的。