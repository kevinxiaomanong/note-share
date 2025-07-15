### Java认证考试

过于极端的不可取

1、当maven依赖版本有冲突时，会尝试解决这些冲突并选择一个合适的版本，通常情况下会选择最近的版本来满足依赖关系，但如果maven选择的版本和代码有冲突那就会编译报错

2、G1垃圾收集器有内存整理的功能所以不需要考虑内存碎片

3、Flight Recorder是可以看到上下文切换、IO情况的

4、TreeMap、HashMap允许value为null

5、要判断应用的线程被Block，有CAT的Thread Dump报表、jstack命令，但是看内存是没用的，阻塞是不会影响的

6、记录Tag只可以查询某个Tag Value的日志

7、携程三中心架构 每个中心只有三分之一数据 SOA服务部署时至少在两个机房部署

8、SOA客户端不报告心跳，那是服务端的功能

9、jmap产生的内存dump文件可以用MAT工具分析 在VI上看某台服务器的GC日志

10、Cat全链路主要是做日志串联

11、日志的Source默认是记录日志的Java全类名

12、服务和应用是多对多关系，SOA服务调用时使用的是Http/https协议

13、Framework里有个Foundation类，里面有很多元信息API，但是当前服务器的CPU核数和内存大小获取不到

14、Credis集群支持DR机房有 SHAOY、SHAJQ、FRA-AWS，使用DR集群时要申请直到服务端部署完成才能使用，客户端不需要特殊配置

15、Qconfig：单个文件限制512k，单个应用中创建的文件数目没有限制，如果是有耦合关系的配置不建议直接使用qconfig

16、jstat可以查看JVM垃圾回收行为的实时统计

17、OOP规约 ：避免通过类的对象引用来访问该类的静态变量或静态方法，无谓增加编译器解析成本，直接使用类名访问即可

18、服务端熔断好像是不允许当前客户端访问



### 内功心法

**JVM运行时内存消耗的部分**

- heap 堆
- 线程用的栈，java虚拟机栈/本地方法栈，PC
- 直接内存
- 方法区实现（元空间实现1.8）



**分代内存管理**

Hotspot虚拟机的heap内存被分成young区和old区

- young区：包含一个Eden区和两个survivor区，这个区域用完时会发生Minor GC，Eden区和from区存活的对象会去to区，然后GC，结束后From区和to区交换
- old区 在survivor区存活超过一定次数会进old区，也有一部分大对象直接在old区分配，当此区域用完时会发送Full GC，Full GC会堆Young区和Old区都进行GC



**STW**

进程内的所有应用线程停止运行，如果发送STW时一个线程正在处理请求，则此请求只有STW结束后才会继续执行，则改请求的处理时间是一定大于STW时间的



对于有些concurrent并发的GC，在某些GC阶段还是需要STW，由于GC线程和应用线程同时运行，所以需要关注两个问题：

1. GC线程会消耗CPU导致系统CPU变高，当系统CPU不足时可能会导致应用线程的CPU时间片被GC线程抢占而导致处理变慢
2. GC时应用线程仍在运行，同时也会产生内存分配，所以需要提前进行Concurrent GC保证heap内可用内存支撑应用线程对内存分配的需要，否则并发GC失败重新进入STW的GC



**Young GC**

不管哪种GC算法，Young GC是一定STW的

本质上还是存活对象被copy到Survivor后，Eden区和另一个Survivor是不需要内存整理的，Young GC的耗时还是很大一部分取决于存活对象数量和引用关系复杂度（可达性分析算法来判断存活对象）

当在Survivor区存活了一定次数或是当Survivor中年龄小于等于age的对象占比超过一定阈值，那么年龄大于age的对象会直接晋升到老年代



**old GC**

针对Old区的GC



**Full GC**

- 对整个heap进行回收
- 和Heap的大小、存活数量、对象引用关系的复杂度有关



**并行收集器**

由于并行、整个回收过程STW，也因此不需要担心回收过程中产生新的对象，且CPU资源可以被GC线程充分利用，所以回收效率和吞吐都比较高，适合后台运算而不需要太多交互的应用，例如后台批处理任务，异步信息消费者，这种对响应时间不高的应用



**CMS**

并发标记清理

适合对响应时间较高、且CPU资源比较充裕的应用

- 当剩余内存不足时会退化为单线程的Full GC
- 不一定会进行内存整理 需要考虑内存碎片问题
- 分为四个阶段 初始标记、并发标记、重新标记、并发清理 其中1、3阶段会触发STW





**G1**

- 设计初衷用于替换CMS收集器，适合大堆（>=6G）
- 相比CMS会进行内存整理，不用考虑内存碎片的问题
- STW时间可控
- FUll GC时单线程执行



**jvm参数**

-Xms 初始堆大小 -Xmx最大堆大小

-Xss 每个线程stack大小 -Xmn Young区大小

-XX：+PrintGC 每次GC时打印信息

-XX +HeapDumpOnOutOfMemoryError 在内存溢出时产生一个heap dump堆快照便于分析



**jvm工具**

jps

查看java线程的具体状态，有点类似unix的ps命令，但jps只显示java进程，我们通过这个快速定位pid

jstat

监控，对堆的使用情况进行实时统计

jmap

打印出某个java线程内存中所有对象的情况，可以用于产生heap dump

jstack

查看java进程id的java进程id的堆栈信息

memory analyzer

分析heap dump文件，来排查内存泄漏问题、或是对java进程的内存进行分析

Flight Reconder

收集java进程运行时的诊断信息

vi

通过vi可以查看服务器的GC日志





### 携程Java规范

**Parent Pom**

<parent>
  <groupId>com.ctrip</groupId>
  <artifactId>super-pom</artifactId>
  <version>1.0.0</version>
</parent>

里面有CtripGroup  Super Rule，定义了集团Java项目都需要遵守的规范全集



**super Rule**

从2017年3月开始，TARS系统会对Maven类型项目进行严格的检查，不过会直接build失败，携程借助enforcer插件来进行规范化，具体规范如下

必须使用Ctrip Super POM

groupId规范：我们是数据智能部应该是com.ctrip.di

verison规范：必须是三段数字，之后加或不加-SNAPSHOT

当有聚合模块的父子工程时，聚合模块以及其各个子模块应使用相同的版本号

Nexus库填充好地址

maven插件需要用artifactId为maven-compiler-plugin的

release版本不能被覆盖，需要修改Pom中的version

有些组件不能依赖：

解决办法：用 mvn dependency:tree -Dverbose -Dincludes=:commons-logging 这个命令，看工程在何处引入了commons-logging，然后去掉不该被依赖的组件

组件依赖的版本要唯一，在直接或间接的路径中都指向同一版本

正式版本不能依赖于SNAPSHOT版本的组件



**Bom**

Bom定义了一整套相互兼容的jar包版本集合，引入后我们将依赖的版本交给bom管理

公司内部创建的所有Maven项目必须通过BOM的方式引入框架、大数据等公共部门提供的组件

Bom的使用方式：

先在dependencyManagement中声明依赖，scope为import



**日志**

原则上不记录在本地，统一记录到CLOG/CAT

方式：

1、slf4j-api 

- 使用slf4j的API记录
- 使用log4j2或logback的Appender把日志统一收集到Clog或CK，提供数据查询与指标聚合
- 对于第三方类库 使用slf4j桥接模式来接管



**点火VI**

即应用初始化，应用在启动时会启动一个点火线程，在应用发布时如果机器点火失败是不会被拉入集群的

其意义在于检查应用配置和外部依赖正确，并让应用充分预热后才开始处理外部流量

流程一般有：

- 对依赖的外部资源进行初始化，如：预先建立数据库、消息队列等外部服务的线程池，这类工作一般是对应框架组件的点火插件会完成
- 对应用进行预热，例如加载数据到应用本地内存缓存，自己实现点火插件



**部署**

基本原则

- 单服务器单应用
- Tomcat端口http默认为8080，https为8443



classpath：META-INF/app.properties 里面要放app.id和jdkVersion

server.properties:存放一些服务器级别的配置和信息