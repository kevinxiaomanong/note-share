官网：http://pages.release.ctripcorp.com/shark/shark-docs/

shark的本质是将UI文案和业务代码分离，实现解耦

内容团队在shark上导入UI文案的多语言化 开发团队通过client sdk去获取多语言文案



### shark名词解释

1、app 应用

2、page 用于聚合一个页面上的key

3、view 一张图片

4、key 翻译内容获取的关键标记

5、TransStatus 翻译状态

6、locale 语言+地区 在Shark中每一种翻译都有一个唯一的locale标识 例如：zh-CN en-US



### 如何使用？

即在平台上注册好东西 然后添加上翻译 发布出来即可

发布完后就可以通过sdk去访问



### sdk接入

http://conf.ctripcorp.com/pages/viewpage.action?pageId=211856057

1、引入maven坐标和插件



然后直接用Shark类去查即可







### Shark使用流程

1、新建应用 配置这个应用的tag和locale

2、新建key 然后翻译

3、新建页面 

然后维护好key和页面的关系即可 再然后就是通过Java SDK获取了



### 













