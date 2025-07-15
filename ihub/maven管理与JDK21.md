### JDK21升级

#### 升级方式

总的来说JDK21升级的方式有两种：

1、本地升级（推荐）

即不改变项目的Spring版本，只是做一些适配到JDK21,或者可以使用fx提供的升级脚本

2、跨Spring版本

即我们会把Spring版本也升级到Spring6/SpringBoot3，然后把bom也改用Java21后缀的版本

但是注意高版本的Spring使用Jakarta包而不再使用Javax包



#### 升级问题

1、Java module化

java常用的jar其实只是类文件的容器，

java9+引入模块功能有导出exports和open限制，即允许代码引用和允许反射访问

所以大部分的报错都是模块限制的问题：即某一个模块没有把某一个类的能力给另一个模块，因为一般我们都不会把应用打包成模块，所以一般都是unnamed module

解决方式是：在JVM参数加上--add-opens/--add-exports

tomcat应用打一个extraenv.sh 加JAVA_OPTS 注意格式

注意如果你只是加在maven-surefire里 那也只是影响跑UT时候的效果



2、Lombok低版本的编译问题

- 升级lombok版本到1.18.30以上



3、高版本字节码兼容问题

升级组件版本：

- VI ： 不低于0.11.57、
- SpringBoot 2.x



此外这里有个思考点：

一个用Java21编译并发布的Jar包，可以被使用Java8编译的应用引用嘛



4、UT coverage 0

首先要看测试用例有没有执行，maven会扫描当前classpath下面有哪些测试框架，例如Junit4、Junit5

对应的版本用对应的测试框架去跑，发现如果Junit4和Junit5同时存在的情况，会优先使用Junit5去跑

还有个点是jacoco，因为pipeline覆盖率的计算是依赖jacoco agent，通过字节码插桩的方式来统计

代理和Java版本是有关系的，需要升级到0.8.11，还需要改命令行参数，需要加上@{argLine}

如果覆盖了surefire插件需要加上这个



#### Java9-21的新特性

1、虚拟线程 ！

早期版本Java线程和系统线程是1:1的



#### Maven依赖管理机制

基本原则：

1、最短路径优先

2、路径相同先声明者优先

3、显式指定优先

4、优先的不仅是版本，而是整个依赖配置



#### Maven实践

BOM优先：包含一系列依赖的版本定义，保证各个依赖之间版本的兼容性

FrameWorkBom

常见用法为：

通过dependencyManagment将其import进来，然后在引入组件的时候就不需要写版本号了

大的版本代表年份 8=2024

然后SpringBoot3专用 -Java21



存在风险的用法：使用bom但是又单独指定某些组件的版本

这样可能长期以来会导致一些兼容性问题

可以在gitlab上看complie这个环节，然后会罗列出哪些组件的版本和bom里不一致



exclusion：将指定依赖从定义exclusions的依赖的依赖树中排除

非常不建议使用，还是建议用dependencymanagement来统一版本



依赖问题分析工具：

mvn dependency:tree 去拿到依赖树



















