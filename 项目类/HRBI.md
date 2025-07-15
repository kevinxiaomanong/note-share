### 项目背景

21年初立项开始开发 主要是可群、书基。俊松开发







## 项目技术点

### 一、Smart-doc

http://conf.ctripcorp.com/pages/viewpage.action?pageId=1800680279





## 项目踩坑点

### 一、CkClientUtil并发账密信息丢失

本质是之前使用双重检查锁的实现上有点疏忽，单例成员变量少了volatile修饰符

这可能会导致指令重排序，返回一个尚未完全构建的对象

因为instace = new Singleton()这行代码的执行可以分为三步：

1. 分配内存空间
2. 调用构造函数，初始化对象
3. 将instance指向分配的空间

由于指令重排序的现象可能3先于2执行，这样instance不为null了，下一个线程getInstance的时候

发现instance不为null，于是出现问题，那么在这个例子中就是

后续instance里的datasource信息没封装，就被进程获取到了从而出现问题

我们可以对instance加上volatile修饰来禁止指令重排序解决













































