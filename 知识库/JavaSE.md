### CompletableFuture详解

https://mp.weixin.qq.com/s?__biz=Mzg4Nzc3NjkzOA==&mid=2247488046&idx=1&sn=2bb0b6dc4576278ff2e7f9b917cb6fe8&chksm=cf8461ebf8f3e8fd013d08c5028d41281444b1ac1d60f1706c841c4b444a4235d82c84644b9b#rd

在Java中用来表示异步计算的结果，并提供了很多API供我们处理结果

内部使用result属性来保存计算结果，以及若干属性waiters来保存等待结果的任务，一旦计算完成CompletableFuture会通知所有等待结果的任务



看源码会发现：它里面有两个属性

```
volatile Object result;       // Either the result or boxed AltResult
volatile Completion stack;    // Top of Treiber stack of dependent actions
```

result存储当前CF的计算结果，而stack表示当前CF完成后要触发的依赖动作，动作以栈的方式存储，stack表示栈顶元素

在设计思想上CF类似观察者模式，即每个CF可以看做一个被观察者，其内部的Completion类型的链表stack会存储到注册到其中的观察者，被观察者完成后会弹栈stack属性依次通知注册到其中的观察者



大致的执行流程

1、通过CompletableFuture类的去创建新的CompletableFuture对象，并通过supplyAsync定义需要再后台线程执行的异步任务

2、一旦CF对象创建并定义了异步任务会立即在后台线程中开始执行并返回一个代表异步计算结果的CF对象

3、异步任务执行过程，当任务完成会设置自己的结果值将状态标记已完成

4、注册回调方法 例如thenrun、thenAccept、thenapply注册回调函数 当然回调函数也可以是异步的 这样当异步任务结束或是异常了 回调函数会被触发

5、我们可以通过thencombine、allof、anyof来将多个CF对象进行组合，形成更复杂的异步任务处理流程

6、通过exceptionally handled来注册异常处理函数，当异步任务出现异常时这些处理函数会被触发

7、使用get或join来阻塞当前线程并等待CF对象完成并获取result 如果要取消可以通过cancel（）取消异步任务执行



具体方法介绍

异步执行任务：

- supplyAsync：异步执行一个有返回值的供应商
- runAsync：异步执行一个没有返回值的任务

链式操作

- thenrun 返回一个void空值的CF对象，接收一个Runnable类型（没有参数 没有返回值）
- thenaccept 接收一个Consumer函数（参数T泛型 没有返回值）
- thenApply 接收function，有返回值

异常处理

- whenComplete 没有返回值
- thenComplete 有返回值

异步任务组合

- allof 将一组CF作为参数，返回一个新CF，这个CF需要在所有CF完成后才算完成
- anyof 任意一个CF完成就算完成
- thencombine 指定两个CF完成后怎么处理 得到新CF

取值与状态

- join 
- get
- getnow 会返回默认值

依赖

- getNumberOfDependents 当前依赖其他异步任务的数量

并发限制

通过使用线程池来限制CF的并发执行数量，将线程池传递给CF



总结

run --- 入参Runnable

supply --- Supplier

Accept --- consumer

apply --- function

either -- 谁先完成消费谁

both -- 两个任务都完成





# JavaGuide





## JavaSE上

write once，run anywhere是早期java的宣传口号，Java通过字节码+虚拟机的技术实现了跨平台性

但计算机技术发展到现在，其实跨平台性的实现不再那么独特，这里引发一些对跨平台的思考：



### 跨平台的思考

所谓的跨平台其实是让程序在不同操作系统（Windows、Linux、macOS）、硬件架构（如x86、ARM）上无需修改即可运行，我们来分析一些编程语言实现跨平台的途径

1、基于“中间层/运行时”的跨平台

.NET（C#）编译为MSIL(微软中间语言)再由CLR运行时解释/编译为机器码，其实类似Java的字节码+JVM了

还有类似Python/JavaScript，这类解释型语言，python只需要目标平台安装了对应版本的解释器即可，代码无需编译为机器码，直接在python解释器运行

JS写的前端代码可在任何浏览器运行

2、基于“交叉编译”的跨平台（直接生成目标平台机器码）

编译型语言通过交叉编译工具链，在一个平台直接生成另一个平台的可执行文件，无需在目标平台重新编译，实现“一次编码，多平台编译”

Golang是静态编译型语言，自带强大的交叉编译工具链，开发者在windows本地直接编译出Linux、macOS等平台的可执行文件

C/C++ 也是类似，其本身不直接跨平台，但通过不同平台的编译器（windows的MSVC、Linux的GCC）可将同一套代码编译为对应平台的机器码

3、基于“虚拟机/容器”的跨平台（部署层抽象）

不依赖语言本身的特性，而是通过外部工具（虚拟机、容器）提供统一的运行环境，屏蔽底层系统差异

Docker容器：将应用及依赖（库、环境变量）打包成容器镜像，镜像可在任何按照Docker引擎的平台运行，即一次构建、多平台运行

虚拟机：通过虚拟机软件（VMware、VirtualBox）在物理机上模拟出完整的操作系统（如Linux），应用在虚拟机内运行，与物理机系统无关，性能开销高于容器



得益于虚拟化技术的发展，可以通过Docker实现跨平台，目前看来Java强大的生态是最大的卖点之一



### JavaSE与Java EE的思考

JavaSE：是所有Java应用的地基，包含核心语法（类、接口、泛型）、基础类库（如java.lang、java.util）、JVM等，是Java运行的最小环境，任何java程序都必须基于JavaSE运行

JavaEE：是在JavaSE基础上为企业级应用定义的规范集合（而非具体实现），包含一系列API和服务标准，比如Web层有Servlet，JSP 业务层有EJB 数据层有JPA

所以在实际工作中你会感觉都在使用JavaSE，因为JavaEE是规范而非工具，实际开发中它的功能被更易用的框架封装了，开发者直接接触框架和JavaSE基础语法，例如Servlet规范定义如何处理Http请求，但具体实现这个规范的是Tomcat、Jetty等Web容器

并且Spring框架以简化JavaEE开发为目标，用更轻量的方式实现了JavaEE的核心功能（依赖注入、事务管理），因此现在企业级开发主流都是Spring生态





### JVM JRE JDK

JVM针对不同系统（windows、mac、linux）有特定实现，用来运行java字节码，很多其他语言例如Kotlin、Jruby通过各自编译器编译成.class文件并最终在JVM不同平台上运行，而JVM不止有一种，只要满足JVM规范都可以开发，目前使用最广的是HotSpot虚拟机

JRE是运行已编译Java程序所需环境，包含JVM和Java基础类库Class Library两部分

而JDK是一个功能齐全的Java开发工具包，包含JRE以及编译器javac和其他工具













