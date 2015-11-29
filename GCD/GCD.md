##几个概念
##sync同步:
任务完成之后才会做下一个任务（block）
##async异步：
立即返回，不会等待任务结束

##串行队列:dispatch_queue_create/main
里面的任务会依次进行
##并行队列
并发执行,里面的任务会遵循FIFO，但是任务可以任何顺序完成


----
![](/Users/Jincc/Desktop/MD/GCD/97583D20-794B-4105-9A10-276D83B9692D.png)

##信号量

