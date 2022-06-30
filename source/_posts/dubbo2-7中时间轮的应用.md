---
title: dubbo2.7中时间轮的应用
copyright: true
date: 2020-05-21 11:32:42
tags:
	- dubbo时间轮
categories:
	- duubo
---

## 前言 
dubbo内部有比较多定时任务的管理功能，JDK也提供了Timer和DelayedQueue等工具类，可以实现简单的定时任务管理，其底层实现就是使用的堆这种数据结构，存取的时间复杂度是O(nlogN)，无法支持大量的定时任务。dubbo内部采用了时间轮的方式来管理定时任务。应用场景比如：dubbo的心跳机制、dubbo客户端超时检测等。

时间轮是一种高效的、批量管理的定时任务的调度模型。时间轮一般会实现一个环形结构，类似于时钟，分为很多槽，每个槽代表一个时间间隔，每个槽使用双向链表来存储定时任务；指针周期性地跳动，跳动到一个槽位，执行对应的定时任务。

![image-20220701053550790](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/image-20220701053550790.png)

注意下单层时间轮的容量和精度是有限的，如果时间跨度特比大，精度要求很高，或者海量定时任务需要调度的场景，通常会使用多级时间轮以及持久化存储的方案。
比如多级时间轮以及持久化存储与时间轮结合，每级时间轮的时钟周期不一样，比如年级别时间轮、月级别时间轮、日级别时间轮、毫秒级别时间轮，定时任务在创建时先持久化，在时钟指针接近时预读到内存，并且需要定期清理磁盘上的过期任务。


Dubbo中，时间轮的实现方式是主要在dubbo-common的org.apache.dubbo.common.timer包中。dubbo和netty的实现基本一致，netty时间轮一个应用场景简单提下，Redisson实现分布式锁提供了watchdog锁续期的功能，为了避免每加锁一次起一个线程去扫描是否需要续期以及执行续期逻辑带来的压力，采用了netty的时间轮来注册续期任务，只用一个线程和合适的时间周期完成了续期逻辑。

### 核心接口

Timer接口：定义了定时器的基本行为。核心方法是newTimeout方法，提交一个定时任务（TimerTask）返回关联的Timeout对象。
```java
/**
 * 定义了定时器的基本行为
 * 核心方法是newTimeout方法：提交一个定时任务（TimerTask）并且返回关联的Timeout对象，类似于线程池中提交任务
 * Schedules {@link TimerTask}s for one-time future execution in a background
 * thread.
 */
public interface Timer {

    /**
     * Schedules the specified {@link TimerTask} for one-time execution after
     * the specified delay.
     * 向时间轮中提交一个定时任务
     *
     * @return a handle which is associated with the specified task
     * @throws IllegalStateException      if this timer has been {@linkplain #stop() stopped} already
     * @throws RejectedExecutionException if the pending timeouts are too many and creating new timeout
     *                                    can cause instability in the system.
     */
    Timeout newTimeout(TimerTask task, long delay, TimeUnit unit);

    /**
     * Releases all resources acquired by this {@link Timer} and cancels all
     * tasks which were scheduled but not executed yet.
     *
     * @return the handles associated with the tasks which were canceled by
     * this method
     */
    Set<Timeout> stop();

    /**
     * the timer is stop
     *
     * @return true for stop
     */
    boolean isStop();
}
```

HashedWheelTimer实现类：Timer接口的实现类。通过时间轮算法实现了一个定时器。执行过程为：
- 根据当前时间轮指针选定对应的槽。
- 遍历槽上的定时任务（HashedWheelTimeout），对每个定时任务进行计算，是当前时钟周期则去除，如果不是则将任务中的剩余时钟周期-1，代表距离执行又接近了一圈。


TimerTask接口：所有定时任务都要继承TimerTask接口。
```java
/**
 * 所有定时任务都需要继承的接口
 * A task which is executed after the delay specified with
 * {@link Timer#newTimeout(TimerTask, long, TimeUnit)} (TimerTask, long, TimeUnit)}.
 */
public interface TimerTask {

    /**
     * Executed after the delay specified with
     * {@link Timer#newTimeout(TimerTask, long, TimeUnit)}.
     *
     * @param timeout a handle which is associated with this task
     */
    void run(Timeout timeout) throws Exception;
}
```

Timeout接口：TimerTask中run()方法的参数，可以查看定时任务的状态，还可以操作取消定时任务

![image-20220701053735936](https://zlj1217-blog-image.oss-cn-hongkong.aliyuncs.com/mac_picgo/image-20220701053735936.png)



HashedWheelTimeout：Timeout接口的唯一实现，是HashedWheelTimer的内部类。扮演两个角色：

- 第一个，时间轮中双向链表的节点，即定时任务TimerTask在HashedWheelTimer中的容器
- 第二个，定时任务TimerTask提交到HashedWheelTimer之后的句柄，用于在时间轮外查看和控制定时任务。

### HashedWheelTimeout的核心字段
```java
/**
     * 实现了Timeout接口 内部类 HashedWheelTimeout是Timeout的唯一实现。 两个作用
     * 1.时间轮中双向链表的节点，定时任务TimerTask在HashedWheelTimer中的容器
     * 2.TimerTask提交到HashedWheelTimer之后返回的句柄（Handle)，用于在时间轮外部查看和控制定时任务
     */
    private static final class HashedWheelTimeout implements Timeout {

        private static final int ST_INIT = 0;
        private static final int ST_CANCELLED = 1;
        private static final int ST_EXPIRED = 2;
        // 状态控制
        private static final AtomicIntegerFieldUpdater<HashedWheelTimeout> STATE_UPDATER =
                AtomicIntegerFieldUpdater.newUpdater(HashedWheelTimeout.class, "state");

        private final HashedWheelTimer timer;
        // 实际被调度的任务
        private final TimerTask task;
        
        // 定时任务被执行的时间 单位纳秒
        // 计算公式：currentTime（创建 HashedWheelTimeout 的时间） + delay（任务延迟时间） - startTime（HashedWheelTimer 的启动时间）
        private final long deadline;

        @SuppressWarnings({"unused", "FieldMayBeFinal", "RedundantFieldInitialization"})
        // 状态有三种 INIT(0)、CANCELLED(1)、EXPIRED(2)
        private volatile int state = ST_INIT; // 状态字段

        /**
         * RemainingRounds will be calculated and set by Worker.transferTimeoutsToBuckets() before the
         * HashedWheelTimeout will be added to the correct HashedWheelBucket.
         * 当前任务剩余的时钟周期数。时间轮表示的时间长度有限  在任务到期时间与当前时刻的时间差，超过时间轮单圈能表示的时长
         * 就出现套圈的情况，这时需要该字段表示剩余的时钟周期
         */
        long remainingRounds;

        /**
         * 当前定时任务在链表中的前驱和后继节点
         *  单线程操作不需要加锁控制
         */
        HashedWheelTimeout next;
        HashedWheelTimeout prev;

        /**
         * The bucket to which the timeout was added
         */
        HashedWheelBucket bucket;
```

### HashedWheelTimeout的核心方法
- isCancelled()、isExpired()、state()方法：主要用来检查HashedWheelTimeout的状态。
- cancel()方法：将当前HashedWheelTimeout状态设置为CANCELLED，将当前HashedWheelTimeout添加到canceledTimeouts队列等待销毁。
- expire()方法：当任务到期时，会调用该方法将当前HashedWheelTimeout设置为Expired状态，然后调用其中的TimerTask的run()方法执行定时任务。
- remove()方法：将当前的HashedWheelTimeout从时间轮中删除。

### HashedWheelTimer 时间轮
上面有提到HashedWheelTimer实现类是时间轮的具体实现，工作原理是根据当前时间轮的指针选定对应的槽（HashedWheelBucket），从双向链表的头节点开始迭代，对每个HashedWheelTimeout进行计算，属于当前时钟周期则取出运行，不属于则将其时钟周期-1，等待下一圈的判断。

#### 核心字段
```java
// 实现了Timer接口 通过时间轮算法实现了一个定时器
    // 根据当前时间轮指针选定对应的槽（HashedWheelBucket），
// 从双向链表头开始遍历，对每个定时任务（HashedWheelTimeout）进行计算，属于当前时钟周期取出运行，否则将 剩余时钟周期数 减1
public class HashedWheelTimer implements Timer {

    /**
     * may be in spi?
     */
    public static final String NAME = "hased";

    private static final Logger logger = LoggerFactory.getLogger(HashedWheelTimer.class);

    private static final AtomicInteger INSTANCE_COUNTER = new AtomicInteger();
    private static final AtomicBoolean WARNED_TOO_MANY_INSTANCES = new AtomicBoolean();
    private static final int INSTANCE_COUNT_LIMIT = 64;
    // 状态变更器
    private static final AtomicIntegerFieldUpdater<HashedWheelTimer> WORKER_STATE_UPDATER =
            AtomicIntegerFieldUpdater.newUpdater(HashedWheelTimer.class, "workerState");

    // 真正执行的定时任务逻辑
    private final Worker worker = new Worker();
    private final Thread workerThread;

    private static final int WORKER_STATE_INIT = 0;
    private static final int WORKER_STATE_STARTED = 1;
    private static final int WORKER_STATE_SHUTDOWN = 2;

    /**
     * 时间轮当前状态
     * 0 - init, 1 - started, 2 - shut down
     */
    @SuppressWarnings({"unused", "FieldMayBeFinal"})
    private volatile int workerState;

    // 时间指针每次加1所代表的实际时间 单位为纳秒 即槽与槽之间的时间间隔
    private final long tickDuration;
    
    // 时间轮中的槽 即时间轮的环形队列 一般为大于且最靠近n的2的幂次方
    private final HashedWheelBucket[] wheel;
    
    // 掩码 mask=wheel.length-1 执行ticks & mask能定位到对应的时钟槽
    private final int mask;
    
    // 确认时间轮中的startTime的闭锁
    private final CountDownLatch startTimeInitialized = new CountDownLatch(1);
    
    // 两个队列是对于添加的定时任务和取消的定时任务的缓冲。
    // 缓冲外部提交时间轮的定时任务
    private final Queue<HashedWheelTimeout> timeouts = new LinkedBlockingQueue<>();
    // 暂存取消的定时任务 会被销毁掉
    private final Queue<HashedWheelTimeout> cancelledTimeouts = new LinkedBlockingQueue<>();
    
    // 当前时间轮剩余的定时任务总数
    private final AtomicLong pendingTimeouts = new AtomicLong(0);
    // 阈值
    private final long maxPendingTimeouts;

    // 时间轮启动时间 提交到时间轮的定时任务deadline字段值以该时间为起点进行计算
    private volatile long startTime;
}

```

#### newTimeout方法提交定时任务
```java
 // 时间轮 去加入一个新的任务
    @Override
    public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }

        //剩余定时任务数+1
        long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();

        if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
            // 阈值判断
            pendingTimeouts.decrementAndGet();
            throw new RejectedExecutionException("Number of pending timeouts ("
                    + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
                    + "timeouts (" + maxPendingTimeouts + ")");
        }

        // 确定时间轮的startTime start()方法会初始化时间轮，确定startTime
        // 启动workerThread 开始执行worker任务
        start();

        // 计算deadline
        // Add the timeout to the timeout queue which will be processed on the next tick.
        // During processing all the queued HashedWheelTimeouts will be added to the correct HashedWheelBucket.
        long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;

        // Guard against overflow.
        if (delay > 0 && deadline < 0) {
            deadline = Long.MAX_VALUE;
        }
        // 封装为HashedWheelTimeout 加入到队列中
        HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
        timeouts.add(timeout);
        return timeout;
    }
```

newTimeout主要做了几件事：
- 维护了时间轮中剩余定时任务数量字段
- start()方法
    - 计算了时间轮的startTime方法，且修改时间轮的状态
    - worker线程调用start()方法，开启执行扫描时间轮的线程。
- 根据startTime来计算定时任务要调度的deadline，当前时间+延时时间 - 启动时间。
- 封装task为HashedWheelTimeout添加到timeouts执行队列中。


#### worker线程扫描时间轮并执行任务
在worker线程的run()方法中：
```java
    private final class Worker implements Runnable {
        private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();

        // tick周期
        private long tick;

        @Override
        public void run() {
            // Initialize the startTime.
            startTime = System.nanoTime();
            if (startTime == 0) {
                // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
                startTime = 1;
            }

            // Notify the other threads waiting for the initialization at start().唤醒外边的线程
            startTimeInitialized.countDown();

            do {
                // 等待下一个执行周期 即槽之间的时间间隔 比如每个槽之间是1s执行一次
                final long deadline = waitForNextTick();
                if (deadline > 0) {
                    // 确认索引
                    int idx = (int) (tick & mask);
                    // 清理已取消的定时任务
                    processCancelledTasks();
                    // 对应的槽
                    HashedWheelBucket bucket =
                            wheel[idx];
                    // 转移缓存在timeouts队列中的已提交定时任务到时间轮对应的槽中
                    transferTimeoutsToBuckets();
                    // 处理当前槽中的定时任务
                    bucket.expireTimeouts(deadline);
                    tick++;
                }
            } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED); // 模拟时间轮转动

            // Fill the unprocessedTimeouts so we can return them from stop() method.
            for (HashedWheelBucket bucket : wheel) {
                bucket.clearTimeouts(unprocessedTimeouts);
            }
            for (; ; ) {
                HashedWheelTimeout timeout = timeouts.poll();
                if (timeout == null) {
                    break;
                }
                if (!timeout.isCancelled()) {
                    // 状态变更之后 未被加入到槽中的未取消任务加入到unprocessedTimeouts队列
                    unprocessedTimeouts.add(timeout);
                }
            }
            processCancelledTasks();
        }
        
        //  bucket.expireTimeouts 执行定时任务
         /**
         * Expire all {@link HashedWheelTimeout}s for the given {@code deadline}.
         * 遍历双向链表中的全部 HashedWheelTimeout 节点。 在处理到期的定时任务时，会通过 remove() 方法取出， 
         * 并调用其 expire() 方法执行；对于已取消的任务，通过 remove() 方法取出后直接丢弃；对于未到期的任务，
         * 会将 remainingRounds 字段（剩余时钟周期数）减一。
         */
        void expireTimeouts(long deadline) {
            HashedWheelTimeout timeout = head;

            // process all timeouts
            while (timeout != null) {
                HashedWheelTimeout next = timeout.next;
                if (timeout.remainingRounds <= 0) {
                    next = remove(timeout);
                    if (timeout.deadline <= deadline) {
                        // 调用expire 内部会执行定时任务的run方法
                        timeout.expire();
                    } else {
                        // The timeout was placed into a wrong slot. This should never happen.
                        throw new IllegalStateException(String.format(
                                "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
                    }
                } else if (timeout.isCancelled()) {
                    // 取消直接remove
                    next = remove(timeout);
                } else {
                    // 未到期任务的remainingRounds - 1
                    timeout.remainingRounds--;
                }
                timeout = next;
            }
        }
```
时间轮一次转动的流程：
- 时间轮转动，时间周期开始
- 清理用户主动取消的任务，会记录在cancelledTimeouts队列中，在每次指针转动的时候，时间轮也会清理该队列。
- 将缓存在timeouts队列中的定时任务转移到时间轮对应的槽中。
- 根据当前指针定位槽，遍历双向链表，来执行对应的任务，方法实现在HashedWheelBucket.expireTimeouts方法中：
    - 循环遍历双向链表，当定时任务的remainingRounds小于等于0，则说明是当前时钟周期内的任务，判断是否达到了deadline（满足时间的触发，兜底判断，一般在时钟周期内都会满足），如果满足调用expire()方法执行任务，内部会调用TimerTask的run方法，即真正的定时任务的逻辑。
    - 如果用户取消，则直接remove()从链表上摘除。
    - 继续遍历下一个Timeout节点。
- 时间轮一直是运行状态，则重复上述轮询的操作；如果时间轮为停止状态，则遍历每个槽位，来清除注册的定时任务。最后再清理cancelledTimeouts队列中用户取消的任务。


## Dubbo中时间轮的应用

### 客户端的超时检查
客户端发起调用时会创建一个DefaultFuture，用于发起远程调用且阻塞同步等待结果。

```java
// DefaultFuture的newFuture方法
public static DefaultFuture newFuture(Channel channel, Request request, int timeout, ExecutorService executor) {
        final DefaultFuture future = new DefaultFuture(channel, request, timeout);
        future.setExecutor(executor);
        // ThreadlessExecutor needs to hold the waiting future in case of circuit return.
        if (executor instanceof ThreadlessExecutor) {
            ((ThreadlessExecutor) executor).setWaitingFuture(future);
        }
        // timeout check
        // 创建客户端的超时检查任务 时间轮去定时检查是否超时
        timeoutCheck(future);
        return future;
    }
    
    // timeoutCheck方法
        private static void timeoutCheck(DefaultFuture future) {
        // 超时时间的检查
        TimeoutCheckTask task = new TimeoutCheckTask(future.getId());
        future.timeoutCheckTask = TIME_OUT_TIMER.newTimeout(task, future.getTimeout(), TimeUnit.MILLISECONDS);
    }

```
可以看到timeoutCheck方法中创建了一个TimeoutCheckTask（实现了TimerTask接口），然后利用时间轮调用newTimeout加入了一个定时任务。

客户端检查超时公用的时间轮：
```java
public static final Timer TIME_OUT_TIMER = new HashedWheelTimer(
            new NamedThreadFactory("dubbo-future-timeout", true),
            30,
            TimeUnit.MILLISECONDS);
```

TimeoutCheckTask的逻辑：
```java
// 超时检查的任务
    private static class TimeoutCheckTask implements TimerTask {

        private final Long requestID;

        TimeoutCheckTask(Long requestID) {
            this.requestID = requestID;
        }

        // 超时检查的逻辑 在达到对应的超时时间触发
        @Override
        public void run(Timeout timeout) {
            // 根据requestId从Future缓存中获取future
            DefaultFuture future = DefaultFuture.getFuture(requestID);
            // 
            if (future == null || future.isDone()) {
            // 完成正常返回
                return;
            }

            // 否则响应超时 isSent区分是客户端执行超时，还是服务端的超时。
            if (future.getExecutor() != null) {
                future.getExecutor().execute(() -> notifyTimeout(future));
            } else {
                notifyTimeout(future);
            }
        }

        private void notifyTimeout(DefaultFuture future) {
            // create exception response.
            // 客户端在超时之后创建一个超时的返回
            Response timeoutResponse = new Response(future.getId());
            // set timeout status.
            // 根据future的isSent确定状态是客户端响应超时 还是 服务端响应超时
            timeoutResponse.setStatus(future.isSent() ? Response.SERVER_TIMEOUT : Response.CLIENT_TIMEOUT);
            timeoutResponse.setErrorMessage(future.getTimeoutMessage(true));
            // handle response.
            // 响应超时
            DefaultFuture.received(future.getChannel(), timeoutResponse, true);
        }
    }
```

### 与注册中心交互的失败重试
```java
 private void addFailedRegistered(URL url) {
        // 如果注册失败 会添加到重试任务到时间轮 进行后面的异步重试
        FailedRegisteredTask oldOne = failedRegistered.get(url);
        if (oldOne != null) {
            return;
        }
        FailedRegisteredTask newTask = new FailedRegisteredTask(url, this);
        oldOne = failedRegistered.putIfAbsent(url, newTask);
        if (oldOne == null) {
            // never has a retry task. then start a new task for retry.
            // Failback 容错。 如果注册失败 启动一个时间轮去异步重试注册节点 （时间轮的一个应用）
            retryTimer.newTimeout(newTask, retryPeriod, TimeUnit.MILLISECONDS);
        }
    }
```

其中FailedRegisteredTask是TimerTask的实现，代表一个失败重新注册到注册中心的任务，内部就是调用了注册服务的逻辑（比如zk去创建临时节点）；retryTimer是一个时间轮，30毫秒去转动指针扫描槽来执行任务。

```java
    // Timer for failure retry, regular check if there is a request for failure, and if there is, an unlimited retry
    private final HashedWheelTimer retryTimer;

    public FailbackRegistry(URL url) {
        super(url);
        this.retryPeriod = url.getParameter(REGISTRY_RETRY_PERIOD_KEY, DEFAULT_REGISTRY_RETRY_PERIOD);

        // since the retry task will not be very much. 128 ticks is enough.
        // 利用时间轮注册重试和注册中心连接的任务
        retryTimer = new HashedWheelTimer(new NamedThreadFactory("DubboRegistryRetryTimer", true), retryPeriod, TimeUnit.MILLISECONDS, 128);
    }
```