# Java™ Platform Standard Ed. 7

## `java.lang.concurrent`
### `ThreadPoolExecutor`
![structure-min](http://www.wailian.work/images/2018/10/18/structure-min.png)

`ThreadPoolExecutor`类提供了四个构造方法：
```
public class ThreadPoolExecutor extends AbstractExecutorService {
	public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue);
	public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory);
	public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler);
	public ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler);
	...
}
```
- `corePoolSize`：核心池的大小，这个参数跟后面讲述的线程池的实现原理有非常大的关系。在创建了线程池后，默认情况下，线程池中并没有任何线程，而是等待有任务到来才创建线程去执行任务，除非调用了`prestartAllCoreThreads()`或者`prestartCoreThread()`方法，从这2个方法的名字就可以看出，是预创建线程的意思，即在没有任务到来之前就创建`corePoolSize`个线程或者一个线程。默认情况下，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到`corePoolSize`后，就会把到达的任务放到缓存队列当中；
- `maximumPoolSize`：线程池最大线程数，这个参数也是一个非常重要的参数，它表示在线程池中最多能创建多少个线程；
- `keepAliveTime`：表示线程没有任务执行时最多保持多久时间会终止。默认情况下，只有当线程池中的线程数大于`corePoolSize`时，`keepAliveTime`才会起作用，直到线程池中的线程数不大于`corePoolSize`，即当线程池中的线程数大于`corePoolSize`时，如果一个线程空闲的时间达到`keepAliveTime`，则会终止，直到线程池中的线程数不超过`corePoolSize`。但是如果调用了`allowCoreThreadTimeOut(boolean)`方法，在线程池中的线程数不大于`corePoolSize`时，`keepAliveTime`参数也会起作用，直到线程池中的线程数为0；
- `unit`：参数`keepAliveTime`的时间单位，有7种取值
- `workQueue`：一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响
- `threadFactory`：线程工厂，主要用来创建线程
- `handler`：表示当拒绝处理任务时的策略

`ThreadPoolExecutor`类中，有几个非常重要的方法：
```
execute() // 向线程池提交一个任务，交由线程池去执行
submit() // 也是用来向线程池提交任务的，但是它和execute()方法不同，它能够返回任务执行的结果，去看submit()方法的实现，会发现它实际上还是调用的execute()方法，只不过它利用了Future来获取任务执行结果
shutdown() // 执行后停止接受新任务，会把队列的任务执行完毕。
shutdownNow() // 也是停止接受新任务，但会中断所有的任务，将线程池状态变为stop。
```

#### 实现原理
![process-min](http://www.wailian.work/images/2018/10/18/process-min.png)
![runState-min](http://www.wailian.work/images/2018/10/18/runState-min.jpg)

1. 线程池状态
    ```
    // runState is stored in the high-order bits
	private static final int RUNNING    = -1 << COUNT_BITS; // 当创建线程池后，初始时，线程池处于RUNNING状态
	private static final int SHUTDOWN   =  0 << COUNT_BITS; // 如果调用了shutdown()方法，则线程池处于SHUTDOWN状态，此时线程池不能够接受新的任务，它会等待所有任务执行完毕
	private static final int STOP       =  1 << COUNT_BITS; // 如果调用了shutdownNow()方法，则线程池处于STOP状态，此时线程池不能接受新的任务，并且会去尝试终止正在执行的任务
	private static final int TIDYING    =  2 << COUNT_BITS; // 所有任务都执行完毕，在调用 shutdown()/shutdownNow() 中都会尝试更新为这个状态
	private static final int TERMINATED =  3 << COUNT_BITS; // 当线程池处于SHUTDOWN或STOP状态，并且所有工作线程已经销毁，任务缓存队列已经清空或执行结束后，线程池被设置为TERMINATED状态
    ```
1. 任务的执行
    ```
    private final BlockingQueue<Runnable> workQueue;              // 任务缓存队列，用来存放等待执行的任务
    private final ReentrantLock mainLock = new ReentrantLock();   // 线程池的主要状态锁，对线程池状态（比如线程池大小、runState等）的改变都要使用这个锁
    private final HashSet<Worker> workers = new HashSet<Worker>();  // 用来存放工作集
    private volatile long  keepAliveTime;    // 线程存货时间   
    private volatile boolean allowCoreThreadTimeOut;   // 是否允许为核心线程设置存活时间
    private volatile int corePoolSize;     // 核心池的大小（即线程池中的线程数目大于这个参数时，提交的任务会被放进任务缓存队列）
    private volatile int maximumPoolSize;   // 线程池最大能容忍的线程数
    private volatile int poolSize;       // 线程池中当前的线程数
    private volatile RejectedExecutionHandler handler; // 任务拒绝策略
    private volatile ThreadFactory threadFactory;   // 线程工厂，用来创建线程
    private int largestPoolSize;   // 用来记录线程池中曾经出现过的最大线程数
    private long completedTaskCount;   // 用来记录已经执行完毕的任务个数
    ```
1. 线程池中的线程初始化
    ```
    public boolean prestartCoreThread() { // 初始化一个核心线程
        return addIfUnderCorePoolSize(null); // 注意传进去的参数是null
    }
    public int prestartAllCoreThreads() { // 初始化所有核心线程
        int n = 0;
        while (addIfUnderCorePoolSize(null))// 注意传进去的参数是null
            ++n;
        return n;
    }
    ```
1. 任务缓存队列及排队策略
    - `ArrayBlockingQueue`：基于数组的先进先出队列，此队列创建时必须指定大小；
    - `LinkedBlockingQueue`：基于链表的先进先出队列，如果创建时没有指定此队列大小，则默认为`Integer.MAX_VALUE`；
    - `SynchronousQueue`：这个队列比较特殊，它不会保存提交的任务，而是将直接新建一个线程来执行新来的任务。
1. 任务拒绝策略
    - 当线程池的任务缓存队列已满并且线程池中的线程数目达到`maximumPoolSize`，如果还有任务到来就会采取任务拒绝策略，通常有以下四种策略：
    - `ThreadPoolExecutor.AbortPolicy`:丢弃任务并抛出`RejectedExecutionException`异常。
    - `ThreadPoolExecutor.DiscardPolicy`：也是丢弃任务，但是不抛出异常。
    - `ThreadPoolExecutor.DiscardOldestPolicy`：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
    - `ThreadPoolExecutor.CallerRunsPolicy`：由调用线程处理该任务
1. 线程池的关闭
    - `shutdown()`：不会立即终止线程池，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务
    - `shutdownNow()`：立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务
1. 线程池容量的动态调整
    - `setCorePoolSize`：设置核心池大小
    - `setMaximumPoolSize`：设置线程池最大能创建的线程数目大小

#### 示例
- `ThreadPoolExecutorTest`

#### 合理配置线程池的大小
一般需要根据任务的类型来配置线程池大小：
- 如果是CPU密集型任务，就需要尽量压榨CPU，参考值可以设为：N*CPU* + 1
- 如果是IO密集型任务，参考值可以设置为：2 * N*CPU*

当然，这只是一个参考值，具体的设置还需要根据实际情况进行调整，比如可以先将线程池大小设置为参考值，再观察任务运行情况和系统负载、资源利用率来进行适当调整。

### `AbstractExecutorService`
```
public abstract class AbstractExecutorService implements ExecutorService {
	protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) { };
	protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) { };
	public Future<?> submit(Runnable task) {};
	public <T> Future<T> submit(Runnable task, T result) { };
	public <T> Future<T> submit(Callable<T> task) { };
	private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks, boolean timed, long nanos) throws InterruptedException, ExecutionException, TimeoutException { };
	public <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException { };
	public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException { };
	public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException { };
	public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException { };
}
```

### `ExecutorService`
```
public interface ExecutorService extends Executor {
	void shutdown();
	List<Runnable> shutdownNow();
	boolean isShutdown();
	boolean isTerminated();
	boolean awaitTermination(long var1, TimeUnit var3) throws InterruptedException;
	<T> Future<T> submit(Callable<T> var1);
	<T> Future<T> submit(Runnable var1, T var2);
	Future<?> submit(Runnable var1);
	<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> var1) throws InterruptedException;
	<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> var1, long var2, TimeUnit var4) throws InterruptedException;
	<T> T invokeAny(Collection<? extends Callable<T>> var1) throws InterruptedException, ExecutionException;
	<T> T invokeAny(Collection<? extends Callable<T>> var1, long var2, TimeUnit var4) throws InterruptedException, ExecutionException, TimeoutException;
}
```

### `Executor`
```
public interface Executor {
	void execute(Runnable command); // 用来执行传进去的任务
}
```

### `TimeUnit`
`TimeUnit`类中有7种静态属性：
```
TimeUnit.DAYS; // 天
TimeUnit.HOURS; // 小时
TimeUnit.MINUTES; // 分钟
TimeUnit.SECONDS; // 秒
TimeUnit.MILLISECONDS; // 毫秒
TimeUnit.MICROSECONDS; // 微妙
TimeUnit.NANOSECONDS; // 纳秒
```

## References
- [Java并发编程：线程池的使用](https://www.cnblogs.com/dolphin0520/p/3932921.html)
- [如何优雅的使用和理解线程池](https://crossoverjie.top/2018/07/29/java-senior/ThreadPool/)