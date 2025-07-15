## 一、项目信息

daas接口地址：

http://daas.ops.ctripcorp.com/api-config
搜索 stamkt

prd：http://conf.ctripcorp.com/display/dsjyaiyy/STA+Marketing+Campaign+Performance+Monitoring+Solution+-+PRD

UI：https://www.figma.com/design/GwYuojd6mPzSyYbTsBnt4Q/%E8%BF%AA%E6%8B%9C%E6%95%B0%E6%8D%AE%E5%A4%A7%E5%B1%8F?node-id=316-2052&p=f&t=oUYZdlPQrj0qBWSC-0



STA前端应用：https://captain.release.ctripcorp.com/app/100055477/info

后端应用：https://captain.release.ctripcorp.com/app/100055203/info



测试环境查找验证码：http://databank.fws.qa.nt.ctripcorp.com/DataBank/LoginRegister.jsp



## 二、接口文档

1、获取



UI与指标对齐：

Saudi order&trend

http://daas.ops.ctripcorp.com/api/v1/adm_sci_prj_stamkt_index_mi

搜索指数--订单指数 （月维度）



Competing Dest

http://daas.ops.ctripcorp.com/api/v1/adm_sci_prj_stamkt_index_mi

短线搜索指数



Saudi best things to do(分页接口)



竞对国家信息:

http://daas.ops.ctripcorp.com/api-config/1456

长短线



月度指标这块：

GMV:

缺少一个酒店取消率



{
    "Trains": "http://kefu.train.ctripcorp.com/offline/order/train/detail?orderNumber=",
    "Hotel": "http://supplierweb.order.hotel.ctripcorp.com/order/orderdetail/",
    "Flight": "http://process.flight.ctripcorp.com/fltorder/order/index?module=61386&orderid=",
    "Piao": "http://ordermanage.ttd.ctripcorp.com/v2/detail?orderid=",
    "Activity": "http://htlint.ctripcorp.com/OrderOperate/Order/OrderDetail/",
    "Car": "http://order.car.ctripcorp.com/sdo/customerservice/dist/#/orderInfo/info?orderId=",
    "Tuan": "http://supplierweb.order.hotel.ctripcorp.com/order/orderdetail/",
    "QiChe": "http://customer.qiche.ctripcorp.com/#/offline/orderDetailBus?lineType=1&orderNumber=",
    "Cruise": "http://order.cruise.ctripcorp.com/detail?orderId=",
    "Customized": "http://order.package.ctripcorp.com/tour/order_index/operateOrder?orderId=",
    "Exchange": "http://forex.ctripcorp.com/fx-order-web/Order/Detail?OrderID=",
    "Market": "http://offline.ypmall.ctripcorp.com/mall-business/index#/serverOrderList?orderID=",
    "FishTrip": "http://bnb.ctripcorp.com/bnb-oms/order/",
    "MktSupermember": "http://adv.ctripcorp.com/membercardoffline/?module=221128#/guestCardSearch/",
    "Ship": "http://customer.qiche.ctripcorp.com/#/offline/orderDetailBus?lineType=2&orderNumber=",
    "SceneryHotel": "http://order.shx.ctripcorp.com/SHX-Order-OrderOperate/OrderDetail/OrderDetail?orderID=",
    "Meeting": "http://order.mtg.ctripcorp.com/mtg-order-ordermis/OrderDetail/OrderDetail/",
    "Sport": "http://sport.market.ctripcorp.com/sport-offline/#/info/",
    "Guider": "http://order.package.ctripcorp.com/Tour-Order-Orderoperate/OperateOrder.aspx?orderid=",
    "FinIns": "http://htlint.ctripcorp.com/OrderOperate/Order/OrderDetail/",
    "Mice": "http://kefu.train.ctripcorp.com/offline/order/train/detail?orderNumber=",
    "Vacation": "http://membersint.members.ctripcorp.com/modulejump/tNetv.aspx?module=4126&location=frame&OrderID="
}



## 三、代码优化

handler/Advice

可以看到这个类里面两个核心接口：

- beforeBodtWrite: 当supports方法返回true时，该方法会在响应体被写入之前调用，允许对控制器的返回值进行二次处理，这里我们主要是将返回值包装成统一的RestResponse格式







sheet

我们确定一下怎么拼sheet

长短线拼一个

按照rank+type排序



htl和flt拼一个

前半部分放htl 后半部分flt flt的excel里的star按空处理





GMV 除以1w --> 取整（.前面取整 后面保留两位有效数字00做展示）

AVG price： （直接取整+两位数字展示00）

长短线竞争系数（小数 真实两位数字）

star 0-3星整合 方便展示 防止过多

机票取消率 保留四位有效数字 excel下载样式带百分号5.68% 





## 四、海外目的地代码迁移



```
.DS_Store
.idea
.vscode
node_modules
.next
package-lock.json
next.config.js
release
.history
eslint-report.json
build
_nfes_types
coverage
```





### 迁移问题-如何实现多个海外目的地基于同一trip.com体系的权限校验

首先我们看Umi框架的前端应用配置文件app.ts

```
getInitialState
```

- 这些数据可在整个应用的组件中通过 `useModel('@@initialState')` 访问。



```
render
```

Umi的渲染拦截器，在应用首次渲染前执行登录校验，控制页面是否允许访问

如果是开发环境：直接跳过登录校验，方便本地开发调试

生产环境：通过getSession检查登录状态

如果登录了将maskedEmail存入localStorage 正常渲染

如果未登录 那么清楚本地存储的邮箱 跳转到登录页面，并携带当前页面URL作为回调地址，登录成功后自动返回原页面



request：

请求拦截器

所有请求发送前，自动从localStorage中读取maskedEmail 并添加到请求头中





现在我们其实希望：

- 用户访问www.trip.com/overseas/sta 

首先校验用户登录态--->提示无权限



现在有个问题是：

如果用户用自己的账号登入，其实页面也能看到

所以我们其实少了一个鉴权的逻辑，具体鉴权我们放在页面组件上面

但我在想这样会不会存在单点登录问题





### 优化点：

- 酒店星级三星以下过滤
- 城市-星级（倒序）组合排序
- 机酒竞争系数（保留三位）
- GMV除以一百万+保留两位有效数字 前端加Million处理



## 五、海外目的地架构方面

1、海外用户访问：

默认解析并接入到akamai，akamai是CDN服务提供商

akamai就近接入后默认转发到SIN-AWS的SLB

SIN-AWS的SLB通过path，结合当前应用配置策略判断这部分流量：

1. SIN-AWS本地处理
2. 转至IBU私有VPC
3. 转发回SH

SIN-AWS到SH 链路也是先AWS->HK: 公网 然后HK->SH 专线 这也是默认链路



2、国内用户访问trip.com

由于trip.com未本案



**CDN内容分发网络**

内容指的是：静态资源比如图片、视频、文档、JS/CSS/HTML

分发网络：即将静态资源分发到位于多个不同地址位置机房中的服务器上这样就可以实现静态资源的就近访问

并且减轻服务器以及带宽的负担



然后我们再聊流量链路SOTP



