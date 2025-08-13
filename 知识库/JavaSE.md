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





### JVM JRE JDK 字节码

JVM针对不同系统（windows、mac、linux）有特定实现，用来运行java字节码，很多其他语言例如Kotlin、Jruby通过各自编译器编译成.class文件并最终在JVM不同平台上运行，而JVM不止有一种，只要满足JVM规范都可以开发，目前使用最广的是HotSpot虚拟机

JRE是运行已编译Java程序所需环境，包含JVM和Java基础类库Class Library两部分

而JDK是一个功能齐全的Java开发工具包，包含JRE以及编译器javac和其他工具



**JDK9新特性扩展**

从JDK9开始就不再区分JDK和JRE了，JDK被重构成了94个模块+jlink工具，java可以通过jlink工具创建只包含所依赖的JDK模块的自定义运行时镜像，这样可以减少Java运行时环境大小而不是不管什么应用都是同样的JRE



**字节码**

面向JVM虚拟机的代码，Java程序经过javac编译成字节码.class，再由解释器&JIT变成机器码，再运行

那么在.class->机器码这一步是由解释器逐条将字节码解释为机器码来执行，因此性能上Java通常不如C++这类编译型语言，所以为了优化Java的性能，JVM在解释器之外引入了JIT编译器：程序运行时解释器首先发挥作用代码直接执行，执行过程中JVM会收集程序运行的信息，如果某方法或代码块在一定时间调用次数超过某个阈值就会被编译存入code Cache，下次执行该段代码时就直接从code Cache中读取机器码执行了，这也解释了为什么Java是编译与解释共存的语言



HotSpot采用惰性评估的方法，根据二八定律，消耗大部分系统资源的只有一小部分热点代码，这也就是需要JIT编译的地方，JVM每次根据代码被执行情况收集信息做出相应优化，因此执行次数越多它速度也越快



一般编译型语言开发效率较低，执行效率较快，例如C、C++、Go

而解释型语言开发效率较快，执行效率较低，例如Java、python，而即时编译技术就是为改善解释型语言效率发展出来的技术，例如Java执行时将部分字节码直译为机器码



**AOT技术**

JDK9引入了新的编译模式AOT（Ahead Of Time Compliation），这种模式会在程序被执行前就将其编译成机器码，属于静态编译，避免了JIT预热等方面开销，可以减少内存占用提高Java启动速度

但AOT编译无法支持Java一些动态特性，如反射、动态代理、JNI，很多框架和库（Spring、CGLIB）都用到了这些特性，例如CGLIB动态代理的原理就是修改字节码文件，如果AOT提前编译这些框架和库没办法直接使用了



**Oracle JDK VS open JDK**

历史了解：2006年SUN公司将Java开源也就是有了open JDK，后来2009年Oracle收购了Sun公司，于是在OpenJDK的基础上搞了一个Oracle JDK，Oracle JDK是不开源的，在Java8-Java11加了一些特有的功能和工具，但在Java11之后，OpenJDK和Oracle JDK功能基本一致了



**Java VS C++**

两者都是面向对象的语言，而面向对象的三大特征：封装、继承和多态

- Java不提供指针直接访问内存，程序内存更安全,且具备自动内存管理垃圾回收机制，无需手动释放内存

- Java类是单继承的，C++类支持多重继承，但Java类可以实现多个接口

  

### Java基础语法

#### 1、自增自减移位运算符

```
`int a = 9;
int b = a++;
int c = ++a;
int d = c--;
int e = --d;`
```

最后：a = 11` 、`b = 9` 、 `c = 10` 、 `d = 10` 、 `e = 10



在移位操作中，被操作的数据视为二进制数，就是将其向左或向右移动若干位的运算，例如HashMap（JDK1.8）

中的hash方法

```
static final int hash(Object key) {
    int h;
    // key.hashCode()：返回散列值也就是hashcode
    // ^：按位异或
    // >>>:无符号右移，忽略符号位，空位都以0补齐
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
```

可以看到这里做了无符号右移，将二进制a向右移动b位，高位补零

然后做了个按位异或

这是将哈希码的高16位与低16位进行异或，让高16位参与哈希计算，减少哈希冲突

HashMap通过 （n-1）&hash集散元素在数组中的位置，n是数组长度，必须为2的幂，

例如n=16，那么n-1为15，二进制其实是28个0+4个1，那么任何数与15进行按位与结果只保留低四位

那么如果哈希码的高16位变化大低16位变化小则低4位可能频繁重复导致大量哈希冲突，通过(h ^ h>>>16)

将高16位的特征混入低16位:

- 高16位不变，因为无符号右移，高位补0和源数据异或还是原来的值
- 但是低16位会被异或改变，相当于高位特征融入低16位使得低16位随机性增强减少哈希冲突

还有就是对null的处理，HashMap的null键始终放在数组第0位

那至于为什么是16，因为int类型是32位，右移16位正好折中交换



为什么使用移位运算符：

1. 高效：对应CPU的移位指令，通常在一个时钟周期完成，相比下乘法和除法等算术运算在硬件层面需要更多时钟周期来完成
2. 节省内存：通过移位操作可以使用一个整数（int或long）来存储多个布尔值或标志位

常用于快速乘以或除以2的幂次方

- << 左移运算符，高位丢弃低位补零，x << n相当于x乘以2的n次方（在不溢出的情况下）
- 右移运算符，高位补符号位，低位丢弃，正数高位补0负数高位补1，相当于x除以2的n次方
- 无符号右移，即高位都补0处理

如果移位的位数超过数值所占用的位数，会先求余再操作，也就是说：`x<<42`等同于`x<<10`，`x>>42`等同于`x>>10`，`x >>>42`等同于`x >>> 10`。

移位操作符实际上支持的类型只有`int`和`long`，编译器在对`short`、`byte`、`char`类型进行移位前，都会将其转换为`int`类型再操作。



**补充一些编码知识**

int是Java中32位有符号整数，其二进制包含符号位和数值位，最高位为符号位（`0`表示正数，`1`表示负数）

正数表示很简单就是直接其数值的二进制，但负数的标识需依赖原码、反码、补码的概念，计算机实际存储补码，反码只是中间过渡形式

反码的设计核心是想用加法器实现减法



负数的反码：符号位不变其他位取反

补码：反码+1



正0的原码：0000000

负0:10000000取反1111111加一-->变成1+32个0，最高位溢出了正好变成了00000

而计算机存储的都是补码，那么0其实就只有一种存储那就是全0

那100000代表什么呢？ 首先先减1变成011111111 再取反变成10000000

用10000000来表示-2,147,483,648



#### 2、基本类型与包装类型

Java里有八种基本类型：byte、short、int、long、float、double、char、boolean

然后每种基本类型都有对应的包装类型：Byte、Short、Integer、Short、Float、Double、Character、Boolean

除了定义一些常量和局部变量之外，我们在一些方法参数、对象属性上很少用基本类型，并且包装类型可以用于泛型，但基本类型不可以

基本类型的局部变量存储在JVM栈的局部变量表，成员变量存在JVM虚拟机的堆/元空间中，而包装类型属于对象类型几乎所有的对象实例都存在堆中(JIT会对对象进行逃逸分析，如果某一个对象并没有逃逸到方法外部，那么就可能通过标量替换实现栈上分配，避免堆上分配内存)

基本类型有默认值且不为null，包装类型不赋值就是null

所有整型包装类对象之间值的比较用euals



包装类型存在缓存机制，`Byte`,`Short`,`Integer`,`Long` 这 4 种包装类默认创建了数值 **[-128，127]** 的相应类型的缓存数据，`Character` 创建了数值在 **[0,127]** 范围的缓存数据，`Boolean` 直接返回 `TRUE` or `FALSE`。

```
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static {
        // high value may be configured by property
        int h = 127;
    }
}
```

```
Integer i1 = 33;
Integer i2 = 33;
System.out.println(i1 == i2);// 输出 true

Integer i1 = 40;
Integer i2 = new Integer(40);
System.out.println(i1==i2);//输出false 
/**
Integer i1=40 这一行代码会发生装箱
等价于 Integer i1=Integer.valueOf(40)所以i1用的是缓存中的对象
但Integer i2 = new Integer(40) 会直接创建新的对象。
*/
```

推荐所有整型包装类对象之间值的比较全部用equals方法，因为一旦不在缓存范围内值相同用==会返回false

装箱其实就是调用了 包装类的`valueOf()`方法，拆箱其实就是调用了 `xxxValue()`方法，这个可以通过看字节码看出来



**浮点数为什么会精度丢失**

这个和计算机保存浮点数的机制有很大关系。我们知道计算机是二进制的，而且计算机在表示一个数字时，宽度是有限的，无限循环的小数存储在计算机时，只能被截断，所以就会导致小数精度发生损失的情况。这也就是解释了为什么浮点数没有办法用二进制精确表示。通过指数和尾数保存的

`BigDecimal` 可以实现对浮点数的运算，不会造成精度丢失。通常情况下，大部分需要浮点数精确运算结果的业务场景（比如涉及到钱的场景）都是通过 `BigDecimal` 来做的。





## JavaSE中













