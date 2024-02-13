### 一、概述

qschedule是一个分布式任务调度系统，服务于各种在线/离线应用，提供任务调度能力

支持多种配置方式，还有监控告警



### 二、创建Job

所以只需要

QSchedule采用自注册方式，即应用发布后会自动将任务汇报到调度平台，然后就可以在调度平台找到对应的任务，进行调度相关的配置

推荐使用@QSchedule注解方式

即在类上添加注解，确保被Spring扫描，在方法上添加@QSchedule

标签并加上全局唯一的jobName，确保当前类中只有一个方法加了注解（多了可能导致QSchedule注解扫描不到）

`@Component
public class SleepWorker {
    private static final Logger localLogger = LoggerFactory.getLogger(SleepWorker.class);

    @QSchedule(value = "ctrip.fx.jack.qschedule.demo.sleep")
    public void doWorker() {
        final TaskMonitor monitor = TaskHolder.getKeeper();
        Logger logger = monitor.getLogger();
        localLogger.info("start sleep worker");
        logger.info("start sleep worker");
        try {
            Thread.sleep(1000 * 60);
        } catch (InterruptedException e) {
            localLogger.error("sleep worker is interrupted");
            logger.error("sleep worker is interrupted");
        }
        localLogger.error("sleep worker over");
        logger.error("sleep worker over");
    }
}



但有时候系统有非常多的任务，而这些任务的执行逻辑是一样的，只不过可能对于不同的任务要处理

不同表的数据，那么其实对于这么多的任务来讲 只需要一个参数表示表名就行

因此提供@QScheduleList的功能，只需在qconfig上配一个properties文件

然后kv为jobname + 描述 指明对应配置文件

然后再调度的方法里，通过parameter.getJobName就可以拿到你配在qconfig里的jobname了



### 三、编辑Job

QSchedule只支持软暂停方式，即置标记位的方式



Qschedule的团队认为直接杀死任务例如中断线程是非常危险的操作，因此提供了协商的停止任务方式

即管理后台停止任务时，调度中心会发送停止信息给机器，机器收到消息后会设置一个标记位，而跑任务的代码

可以根据这个消息位的取值然后决定是否停止自己的工作，未响应标记只能通过重启进程来停止任务

通过taskMonitor的isStopped来判断



管理后台使用：

可以停止任务的后续调度

或是停止当前正在执行的任务，如果这次任务还在执行，Qschedule客户端会将停止标志位设置为true，用户代码需要根据标志位自己结束任务，即上述的isStopped方法



### 四、玩转Qshcedule

Qshcedule的设计思想是自注册

然后把繁杂的配置、功能项放在界面操作



此外还有一个是

我们认为直接杀死这个任务例如直接把线程kill掉其实是一个很危险的操作，

qschedule于是采用了这种标志位的方法



下面说说Qshcedule界面的使用方法



1、设置任务拆分

目的--让多台机器同时执行，加速任务，所有分片是并发执行，所有分片结束一次调度才算完成

例如假设现在有100张表需要处理，如果你的任务是单机执行，会影响执行速度



2、执行时长告警



3、失败重试



4、任务负载均衡

即让不同的worker去响应某次调度



5、执行参数设置

可以被Parameter拿到



6、机器worker上下线

因为任务跑其实还是在我们客户端的机器自己跑任务的，可以配置哪些服务器参与跑任务



6、任务手动执行、删除





### 补充：

配置表达式cron

这是一种很方便的调度表达式规则，很多调度的中间件都支持这种



一共有七位 从秒、分、时、日、月、周、年，最后一位选填

https://docs.fx.ctripcorp.com/docs/qschedule/reference/cron



每一位取值有不同范围，然后也支持一些通配符

*通配 ？不定 -范围

然后还有一些L、W 表示最后、最近工作日这样

具体需求可以到时候查表即可

























