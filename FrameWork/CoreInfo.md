### 1、概述

用户的隐私核心数据包括联系电话、邮箱、证件号码等，为了确保用户核心数据不被非法使用和泄漏

公司发起了神盾项目，Coreinfo服务是神盾辅助工具，提供了从服务端对用户核心数据进行脱敏/反脱敏操作

特别的如果需要用到解密功能需要先申请





## 2、Java接入

注意，接入时一定要遵循不可变更原则，即用户调用加密接口拿到的密文，不可再对其进行二次加工，否则解密操作会失败/混乱

举个例子： 021-1234567*12345的电话号码，神盾是只对电话号码的主体号码进行加密，假设加密后得到的密文是123abc7，这时候如果再把前后段拼接起来即021-123abc7*12345的话，再拿回给神盾解密是会报错的

此外调用批量接口时源数据个数有数量限制，一次批量请求不允许超过100个数据



maven依赖引入：

<dependency>
        <groupId>com.ctrip.arch</groupId>
        <artifactId>coreinfo-service-client</artifactId>
    </dependency>

注意如果应用不运行在Web容器还需要提供额外的依赖



基本上引入依赖后就可以使用coreinfo的client使用了

目前提供了新的加解密接口，基本使用方式：

为需要进行加解密操作的String或List<String>字段 添加CoreInfoField注解

然后调用CoreInfoClient.getInstance.encryptBean()/decryptBean()将需要处理的对象传给神盾客户端

client会对对象的标注字段自动执行抽取和回填逻辑



注意入参不是必须包含神盾字段，接口会递归扫描所有字段，如果没有定义则不执行任何动作



**CoreInfoField注解说明**

包含三个属性：

- keyType 必需 待处理的数据类型
- destination  可选 即处理结束后需要替换的字段名称 默认原地替换 否则会将结果赋值给指定字段
- acceptOperation 表示该字段只接受加密/解密操作 默认接收任何操作



**encryptBean/decryptBean接口说明**

新接口和老接口的主要区别是：输入的数据源由infokey变更为任意自定义对象

新增BeanRequestContext以及BeanResponse配合实现自定义功能



我看了下以前老项目用coreinfo，还是老的api即InfoKey和InfoData这一套，最好新改一下



## 3、注意事项

**数据类型**

coreinfo服务会针对不同的数据类型用不同的数据验证方法和加密算法，所以这是一个必传的参数

目前keytype是一个枚举



**状态码定义**

神盾定义了一些Response状态码，来表明每条数据的具体处理结果状态



















