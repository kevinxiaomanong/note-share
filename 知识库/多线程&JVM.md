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



### Java如何调用C程序？本地方法栈&线程栈













































