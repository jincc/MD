##调度器
RACScheduler 所做的就是：调度这些 blocks (schedule blokcs)。我们可以通过那些调度方法所返回的 RACDisposable 对象来取消那些 scheduling blocks。
###RACImmediateScheduler
只支持同步 scheduling，显然，这样一个调度器，没法取消 scheduling，所以它那些方法返回的 disposables 啥也不会做（实际上，它那些 scheduling 方法返回的是nil）
###RACQueueScheduler
使用 GCD 队列来 scheduling blocks
###RACSubscriptionScheduler
如果当前线程有调度器（调度器可以与线程相关联起来：associated）那它就将 scheduling 转发给这个线程的调度器；否则就转发给默认的 background queue 调试器