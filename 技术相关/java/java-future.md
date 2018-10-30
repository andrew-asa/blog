# 零散知识点
+ Runnable,Callable一般情况下是配合ExecutorService来使用的
+ Feature 只是一个接口，亦或说一种规范定义如下

```
public interface Future<V> {
    <!--取消任务-->
    boolean cancel(boolean mayInterruptIfRunning);
    <!--是否已经取消-->
    boolean isCancelled();
    <!--是否已经完成-->
    boolean isDone();
    <!--获取返回结果，同步等待-->
    V get() throws InterruptedException, ExecutionException;
    <!--获取返回结果，同步等待，多长时间超时-->
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

# 参考资料
+ 1:[彻底理解Java的Future模式 https://www.cnblogs.com/cz123/p/7693064.html](https://www.cnblogs.com/cz123/p/7693064.html)
+ 2:[并发编程 Promise, Future 和 Callback http://ifeve.com/promise-future-callback/](http://ifeve.com/promise-future-callback/)
+ 3:[Promise-在Java中以同步的方式异步编程 https://juejin.im/post/5b126065e51d4506bd72b7cc](https://juejin.im/post/5b126065e51d4506bd72b7cc)
+ 4:[Java并发编程：Callable、Future和FutureTask https://www.cnblogs.com/dolphin0520/p/3949310.html](https://www.cnblogs.com/dolphin0520/p/3949310.html)