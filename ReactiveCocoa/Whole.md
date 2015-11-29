##响应式编程就是异步数据流编程。
##信号就是一个异步数据流，即一个将要以时间为序的事件序列.
万物都是流。

我们现在要实现一个button点击计数的功能.

`clickStream.map(transfer -> 1).scan(累加 return f(+))`
