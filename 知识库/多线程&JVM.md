## 经典问题



### Java线程与OS线程的对应关系

其实这本质是一个关于线程模型的问题，核心是在“语言层面的线程抽象”与“操作系统提供的线程资源”之间建立高效的映射关系



JDK21之前，Java中默认的线程（称为平台线程，Platform Thread）与操作系统线程是一对一的映射关系

- 通过new Thread()或ThreadPool创建线程时，JVM通过操作系统的原生API向操作系统申请一个OS线程
- Java线程的声明周期与操作系统线程完全绑定：
- OS负责线程的调度（如CPU时间片分配），JVM仅负责java线程的操作映射到OS线程上



虚拟线程是一种轻量级线程，它的设计目标是突破1：1模型的限制，与OS线程是多对多的映射关系

- 虚拟线程是由JVM绑定的“用户态线程”，不直接绑定OS线程
- 多个虚拟线程可以共享同一个OS线程（称为载体线程，Carrier Thread）：当虚拟线程执行代码时会短暂挂载到一个载体线程上运行，当虚拟线程遇到阻塞操作（如IO等待、锁等待）时JVM会将其卸载，并释放载体线程，让其他虚拟线程可以使用该载体线程
- 虚拟线程的调度由JVM内部的调度器（ForkJoinPool工作线程）负责，属于用户态调度开销低，OS只负责调度载体线程



如何使用虚拟线程：

```
    @Bean(name = "asyncTaskExecutor")
    public ThreadPoolTaskExecutor asyncTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setThreadFactory(Thread.ofVirtual().factory());
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(12);
        executor.setQueueCapacity(30);
        executor.setKeepAliveSeconds(30);
        executor.setThreadNamePrefix("http-daas-async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
```

SpringBoot3+&Spring6 对虚拟线程提供了原生支持，只需要setThreadFactory配置虚拟线程工厂即可

虚拟线程池通常不需要设置核心线程数、最大线程数等（因为虚拟线程极轻量）保留配置仅为兼容，实际运行时虚拟线程数量不受这些参数限制



**虚拟线程在其他语言的例子**

许多编程语言都有类似 Java 虚拟线程的 “轻量级线程” 概念，它们本质上都是**用户态线程**（由语言 runtime 或虚拟机管理，而非直接映射到操作系统线程），目的是在保持高并发能力的同时降低资源开销。其中 Go 语言的**协程（Goroutine）** 是最具代表性的例子之一。

核心目标一致：**用更低的资源开销支持更高的并发量**



### 手写Java线程池

```
package org.elon.thread;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;

/**
 * 优化点：
 * 1、在execute里 我们再启动一个线程后马上尝试将任务放入队列，这可能导致：
 * 新线程尚未开始从队列消费任务 导致任务无法加入队列从而触发拒绝策略
 * 我们应该让新创建的线程直接处理当前任务，而非先将任务放入队列
 * 这其实和JDK的ThreadPoolExecutor的工作原理类似
 */
public class SimpleThreadPool {

    private final BlockingQueue<Runnable> taskQueue;

    private final List<WorkerThread> workers;

    private final int corePoolSize;

    private final int maxPoolSize;

    private final long keepAliveTime;

    private volatile boolean isShutdown;

    @FunctionalInterface
    public interface RejectedExecutionHandler {
        void rejectedExecution(Runnable r, SimpleThreadPool executor);
    }

    private RejectedExecutionHandler rejectedExecutionHandler = (r, executor) -> {
        throw new RuntimeException("任务" + r + "被拒绝执行,线程池已达最大容量");
    };

    public SimpleThreadPool(int corePoolSize, int maxPoolSize, long keepAliveTime, int queueCapacity) {
        this.corePoolSize = corePoolSize;
        this.maxPoolSize = maxPoolSize;
        this.keepAliveTime = keepAliveTime;
        this.taskQueue = new LinkedBlockingQueue<>(queueCapacity);
        this.workers = new ArrayList<>(maxPoolSize);
        this.isShutdown = false;
        for (int i = 0; i < corePoolSize; i++) {
            WorkerThread worker = new WorkerThread();
            workers.add(worker);
            worker.start();
        }
    }

    public void shutdown() {
        isShutdown = true;
        for (WorkerThread worker : workers) {
            worker.interrupt();
        }
    }
    public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
        if (handler != null) {
            this.rejectedExecutionHandler = handler;
        }
    }

    public void execute(Runnable task) {
        if(isShutdown){
            throw new IllegalStateException("threadpool is already shutdown");
        }
        boolean added = taskQueue.offer(task);

        if(added){
            return;
        }

        synchronized (workers) {
            if(workers.size()<maxPoolSize){
                WorkerThread workerThread = new WorkerThread(task);
                workers.add(workerThread);
                workerThread.start();
                return;
            }
        }
        rejectedExecutionHandler.rejectedExecution(task, this);
    }

    private class WorkerThread extends Thread {

        private Runnable initialTask;

        public WorkerThread() {
            this(null);
        }

        public WorkerThread(Runnable initialTask) {
            this.initialTask = initialTask;
        }


        @Override
        public void run() {
            if(initialTask!=null){
                try{
                    initialTask.run();
                }catch (Exception e){
                    System.out.println("任务执行出错：" + e.getMessage());
                }
            }

            while (!isInterrupted()) {

                try {
                    Runnable task;

                    if (workers.indexOf(this) < corePoolSize) {
                        task = taskQueue.take();
                    } else {
                        task = taskQueue.poll(keepAliveTime, java.util.concurrent.TimeUnit.MILLISECONDS);
                    }

                    if (task != null) {
                        try {
                            task.run();
                        } catch (Exception e) {
                            System.out.println("任务执行出错：" + e.getMessage());
                        }
                    } else {
                        if (workers.indexOf(this) >= corePoolSize) {
                            synchronized (workers) {
                                workers.remove(this);
                            }
                            break;
                        }
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    }

}
```

```
public void submit(Runnable task) {
    if(isStoped){
        throw new IllegalStateException("threadpool has stopped");
    }
    boolean isAdded = queue.add(task);
    if (!isAdded) {
        synchronized (workThreadList){
            if (workThreadList.size() < maxPoolSize) {
                WorkThread workThread = new WorkThread(false);
                workThreadList.add(workThread);
                workThread.start();
                return;
            }
        }
        throw new RuntimeException("Task Queue is full! workers is max");
    }
}
```































