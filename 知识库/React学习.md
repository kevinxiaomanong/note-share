### 学习地址

React官网：https://zh-hans.react.dev/learn/tutorial-tic-tac-toe



### 一、快速入门

#### 创建和嵌套组件

React应用程序是由组件组成的，一个组件是UI的一部分，它拥有自己的逻辑和外观，组件可以小到一个按钮也可以大到整个页面

React组件是返回标签的JavaScript函数

例如：

```javascript
function MyButton() {
  return(
    <button>I'm a button</button>
  );
}
```

至此我们已经声明了MyButton，现在把它嵌套到另一个组件中：

```javascript
export default function Square() {
  return <>
    <button className="square">X</button>
    <button className="square">X</button>
    <MyButton></MyButton>
  </>;
}
```

你可能已经注意到<MyButton>这个标签是以大写字母开头的，可以据此识别React组件，React组件必须以大写字母开头，而HTML标签必须是小写字母

而export default关键字指定了文件中的主要组件



#### **使用JSX编写标签**

上面所使用的标签语法被称为JSX，它是可选的，但大多数React项目会使用JSX，因为它很方便，所有我们推荐的本地开发工具都开箱即用地支持JSX

JSX比HTML更严格，你必须闭合标签如<br />而且你的组件也不能返回多个JSX标签，必须将它们包裹到一个共享的父级中

如果你有大量的HTML需要移植到JSX中可以使用在线转换器：https://transform.tools/html-to-jsx



#### **添加样式**

在React中你可以使用className来指定一个CSS的class，它和HTML的class属性工作方式相同：

<img className="avatar">

然后你可以在另一个单独的CSS文件中为它编写CSS规则：

/* In your CSS */

.avatar{

​	border-radius: 50%;

}

此外React并没有规定你如何添加CSS文件，最简单的方式是使用HTML的<link>标签。



#### 显示数据

JSX会让你把标签放到JavaScript中，而大括号会让你回到JS中，这样你就可以从你的代码中嵌入一些变量并展示给用户，例如这将显示user.name:



特别注意

style={{}}并不是一个特殊的语法，而是style={} JSX大括号内的一个普通{}对象，当你的样式依赖于JS变量时你可以使用style属性



#### 条件渲染

React没有特殊的语法来编写条件语句，因此你使用的就是普通js代码



#### 渲染列表

在你的组件中 可以通过map函数将这个数组转换为<li>标签构成的列表

注意<li>有一个key属性，对于列表中的每一个元素其实都应该传递一个字符串或是数字给key，用于唯一标识

通常key来自你的数据，例如ID



#### 响应事件

可以通过在组件中声明事件处理函数来响应事件：

千万onClick={handleClick}的结尾不要加小括号，即不要调用事件处理函数：你只需要把函数传递给事件即可

当用户点击按钮时React会调用你传递的事件处理函数



#### 更新页面

通常你会希望你的组件“记住”一些信息并展示出来，比如一个按钮被点击的次数，要做到这一点，你需要再你的组件中添加state

首先从React引入useState

不过如果你多次渲染同一个组件，每个组件会有自己的state，互不影响



#### 使用Hook

以use开头的函数被称为Hook，useState是React提供的一个内置Hook，可以使用其他内置Hook或是组合现有的Hook来编写属于你自己的Hook

Hook比普通函数更为严格，你只能在你的组件或其他Hook的顶层调用Hook，如果你想在一个条件或循环中使用useState，需要新提取一个新的组件并在组件内部使用它



#### 一些常见Hook

1、useModel

用于在函数组件中访问和使用全局状态

在UMI JS内 model用来管理全局状态 通常模型定义在src/models目录下，每个模型文件会导出一个对象

假设我们在models下有global.ts:

```typescript
// src/models/global.ts
import { useState } from 'react';

export default () => {
  const [name, setName] = useState('Default Name');
  return {
    name,
    setName,
  };
};
```

那么后续const { name } = useModel('global');

我们就可以获取到global模型的状态和方法



2、useAccess

这是UMIJS中用于权限控制的一个Hook，它的具体实现逻辑通常在src.access.ts中定义

会被UmiJS自动加载和使用

还有个问题是我的access.ts是这样定义的 我找不到initialState这个入参是在哪里加载的

```
export default (initialState: API.UserInfo) => {
  // 在这里按照初始化数据定义项目中的权限，统一管理
  // 参考文档 https://umijs.org/docs/max/access
  const canSeeAdmin = !!(
    initialState && initialState.name !== 'dontHaveAccess'
  );
  return {
    canSeeAdmin,
  };
};
```

initialState这个和getInitialState有关，这个函数通常定义在src/app.ts文件中

```
export async function getInitialState(): Promise<{ name: string }> {
  return { name: '@umijs/max' };
}
```

这样把数据传递给access.ts里的initialState



#### 组件间共享数据

每个MyButton有自己独立的count，当每个按钮被点击时只有被点击按钮的count才会改变

但有时经常又需要组件共享数据并一起更新，你需要将各个按钮的state向上移动到最接近包含所有按钮的组件之中

我们可以使用JSX的大括号向MyButton传递信息



#### Prop方式传递

我们可以将state状态或是处理函数以Prop的方式传递给下层组件，即使用JSX的大括号传递信息

这样子组件的函数会被回调给上级组件内定义的函数

从而实现状态上升，使得组件内共享数据



#### const和let定义变量区别

const是定义常量，定义时必须初始化，后续不可改变，但如果是数组或对象是可以的，是可以改变其内容的，因为数组和对象的引用没变

let定义变量，定义时可以不用初始化，后续可改变

两者都是块级别作用域



### 二、井字棋游戏

**状态提升**

命名规范：通过在React中使用onSomething命名代表事件的跑props，使用handleSomething命名处理这些事件的函数



#### 一些写法

```
const nextSquares = squares.slice();
```

改变数组的时候通常先创建副本slice然后再改

const [history, setHistory] = useState([Array(9).fill(null)]);

这里的history其实是一个数组,容纳后初始化阶段，里面是放了一个元素，该元素是包含九个null的数组，即代表gameStart的状态

后续我们每次操作，都会向里面添加



### 三、React哲学

React可以改变你对可见设计和应用构建的思考，当你使用React构建用户界面时，你首先会把它分解成一个个组件，然后把这些组件连接在一起





从原型开始：

#### 步骤一：将UI拆解为组件层级结构

将UI拆解为组件层级结构，一开始在绘制原型中的每个组件和子组件周围绘制盒子并命名它们

然后取决于使用北京，可以考虑通过不同的方式将设计分割为组件：

- **程序设计**---单一功能原理，一个组件理想情况下应仅做一件事情，但随着功能的持续增长它应该被分解为更小的子组件
- CSS 思考将把类选择器用于何处
- 设计 思考如何组织布局的层级

如果你的JSON结构非常棒，经常发现将其映射到UI中的组件结构是一件自然的事情，

![image-20240904190535786](D:\note\工作文档\知识库\assets\image-20240904190535786.png)

#### 步骤二：使用React构建一个静态版本

在拥有组件层级结构之后，就可以根据你的数据模型，构建一个不带任何交互的UI渲染代码版本

通常是先构建一个静态版本比较简单，然后再一个个添加交互

有意思的是构建一个静态版本需要写大量的代码，并不需要什么思考，但添加交互需要大量的思考，却不需要大量的代码。

构建应用程序的静态版本来渲染你的数据模型，将构建组件并复用其他的组件，然后使用props进行传递数据。

Props是从父组件向子组件传递数据的一种方式

你既可以通过从层级结构更高组件开始自上而下构建，也可以从更低层级组件“自下而上”进行构建。在简单的例子中，自上而下构建通常更简单；而在大型项目中自下而上构建更简单

在构建你的组件之后，即拥有一个渲染数据模型的可复用组件库，因为这是一个静态应用程序，组件仅返回JSX。最顶层组件将接收你的数据模型作为props，这被称为单向数据流，因为数据从树的顶层组件传递到下面的组件



#### 步骤三： 找出UI精简且完整的state表示

为了使UI可交互，需要用户更改潜在的数据结构，你将可以使用state进行实现

考虑将state作为程序需要记住改变数据的最小集合。组织state最重要的一条原则是保持它DRY不要自我重复

计算出你应用程序需要的绝对精简state表示，按需计算其他一切，下面有几条简单判断是否是state的原则？

- 随着时间推移保持不变？ 不是state
- 通过props从父组件传递？ 不是state
- 是否可以通过已经存在的state&props计算得到？不是state

相反如果随着时间的推移而变化，并且无法从任何东西中计算出来，那么很可能就是state



在React中有两种模型数据：props和state，下面比较它们

- props其实像是你传递的参数到函数，它们使父组件可以传递数据给子组件，定制它们的展示
- state像是组件的内存，它使组件可以对一些信息保持追踪，并根据交互来改变，例如Button可以保持对state追踪

两者是不同的，但它们可以同时工作，父组件将经常在state中放置一些信息，并且作为子组件的属性向下传递至它的子组件



#### 步骤四：验证state应该被放置在哪

在找出应用程序所需的state后，你需要验证哪个组件是通过改变state实现可响应的，或者拥有这个state

记住React使用单向数据流，通过组件层级结构从父组件传递数据至子组件



对于应用程序里的每一个state：

1. 验证每一个基于特定state渲染的组件
2. 寻找它们的父组件
3. 决定state放置的地方



#### 步骤五 添加反向数据流

即深层结构的表单组件需要再顶层组件中更新state

因此哪些setState的函数就需要通过props的形式传递给深层次组件

让深层次组件回调





### 四、React与组件

react是构建用户界面的基础库，Ant Design提供了丰富的UI组件库，Umijs提供了完整的前端开发框架

看起来React是基础 然后Ant Design是UI组件库 UmiJS是一个前端框架

#### UMI JS

我们既然现在用UMI JS生成出了代码结构，现在可以研究一下每个工程目录文件的意义

```bash
.

├── config

│   └── config.ts （配置文件 包含Umi所有非运行时配置）

├── dist （执行umi build后产物默认输出文件夹）

|---public （存放固定的静态资源 如果存放/public/images.png 则开发可以通过/image.png访问到 构建后会被拷贝到输出文件夹）

├── mock

│   └── app.ts｜tsx （存放mock文件 该目录下所有.ts/.js文件会被mock服务加载从而提供模拟数据）

├── src

│   ├── .umi

│   ├── .umi-production  （.umi这两个文件是临时文件目录 不要提交到git）

│   ├── layouts （全局布局）

│   │   ├── BasicLayout.tsx

│   │   ├── index.less

│   ├── models

│   │   ├── global.ts

│   │   └── index.ts

│   ├── pages  （）

│   │   ├── index.less

│   │   └── index.tsx

│   ├── utils // 推荐目录

│   │   └── index.ts

│   ├── services // 推荐目录

│   │   └── api.ts

│   ├── app.(ts|tsx) （运行时配置文件，在这里扩展运行时能力，例如修改路由修改render方法）

│   ├── global.ts （Umi区别于其他前端框架没有显式程序主入口 所以如果有需要有全局逻辑优先写入）

│   ├── global.(css|less|sass|scss) (全局样式文件)

│   ├── overrides.(css|less|sass|scss) （高优先级全局样式）

│   ├── favicon.(ico|gif|png|jpg|jpeg|svg|avif|webp)

│   └── loading.(tsx|jsx) （全局加载组件）

├── node_modules

│   └── .cache

│       ├── bundler-webpack

│       ├── mfsu

│       └── mfsu-deps

├── .env （环境变量）

├── plugin.ts  （项目级Umi插件）

├── .umirc.ts // 与 config/config 文件 2 选一 （该文件优先级更高）

├── package.json

├── tsconfig.json

└── typings.d.ts
```



**TypeScript**

可以看到Umi默认开启Ts，脚手架生成的项目内置文件都是以xx.ts|tsx为主的

如果想在配置时拥有TS的语法提示，可以在配置的地方包一层defineConfig()

相比JS来讲ts引入了静态类型系统，这是微软开发和维护的编程语言，旨在使得大型JS项目好维护

我们可以编写代码时指定类型，且ts能通过上下文推断

TS需要编译成js才能运行



#### antd组件

DatePicker 日期选择框

https://ant-design.antgroup.com/components/date-picker-cn#%E4%BB%A3%E7%A0%81%E6%BC%94%E7%A4%BA







### 五、前端运行机制

你想我们的前端项目是基于UmiJS生成的，我们在控制台输入npm run start项目就运行起来了

那么背后是怎么运行起来的呢？

**1、读取package.json文件**

npm run start会运行package.json文件中定义scripts部分的start脚本

通常会调用UmiJS的开发服务器命令umi dev 

2、执行UmiJS命令

启动开发服务器，会执行以下操作：

- 加载配置 读取项目中config/config.ts文件的配置 或者是项目下的.umirc.ts（这个优先级比config更高）
- 编译代码 使用Webpack或Vite编译项目代码
- 启动开发服务器 启动一个本地开发服务器 通常运行在http://localhost:8000，并且支持热更新

3、Webpack编译

- UmiJS内部使用Webpack或Vite来处理模块打包和编译
- 会读取所有依赖和源代码，应用各种加载器和插件处理不同类型文件
- 最终生成一个运行的捆绑包

我们项目使用webpack编译工具，可以将项目里的各种资源打包成bundle

webpack使用一个入口文件作为构建的起点，通常是应用主文件：src/index.js

UmiJS入口文件的配置是通过配置文件来实现的，



如何编写index.js文件？





4、启动开发服务器

开发服务器会监听代码文件的变化，修改代码时会重新编译受影响的模块，并通过热更新将变化应用到浏览器，这能大幅提高开发效率



5、服务器访问

这里的服务器由Webpack Dev Server提供，并且集成在UmiJS中，

这个开发服务器可以配置代理，将特定的API请求转发到后端服务器，这对于前后端分离的开发模式非常有用，可以避免跨越问题

且具备实时重载和热模块更换功能，有效提高开发效率





### 六、从React脚手架代码入门

#### 理论

部署发布是执行npm build命令 产物会默认生成到./dist目录下

完成构建后就可以把dist目录部署到服务器上了



**约定式路由**

以pages/*文件夹的文件层级结构来生成路由表

配置式路由，componet若写相对路径，将从该文件夹为起点开始寻找文件



**路由**

在Umi应用是单页应用，页面地址的调整都是在浏览器端完成的，不会重新请求服务端获取html，所有的页面由不同的组件构成，页面的切换其实就是不同组件的切换，你只需要在配置中把不同的路由路径和对应的组件关联上



**Mock**

什么是Mock数据：在前后端约定好API接口以后，前端可以使用Mock数据来在本地模拟出API应该要返回的数据，这样前后端开发可以同时进行，不会因为后端API还在开发而导致前端的工作被阻塞

在/mock目录中的userAPI.ts会被Umi视为Mock文件来处理



而Mock文件默认导出一个对象，而对象的每个Key 对应了一个Mock接口 值则是接口的返回数据



Umi默认开启Mock功能，如果不需要的话从配置文件关闭

```ts
export default {

  mock: false,

};
```



引入Mock.js来方便生成随机的模拟数据， 



```ts
import mockjs from 'mockjs';
export default {  // 使用 mockjs 等三方库  'GET /api/tags': mockjs.mock({    'list|100': [{ name: '@city', 'value|1-100': 50, 'type|0-2': 1 }],  }),};
```



**代理**

即允许客户端通过代理服务与另一个终端进行非直接的连接

在dev环境所有的网络请求都会通过本地server响应分发

```ts
export default {

  proxy: {

    '/api': {

      'target': 'http://jsonplaceholder.typicode.com/',

      'changeOrigin': true,

      'pathRewrite': { '^/api' : '' },

    },

  },

}
```

一般我们使用这个能力来解决开发中的跨域访问问题，因为浏览器存在同源策略，之前我们会让服务端配合使用CORS策略来绕过跨域访问问题，现在可以在本地代理解决

值得注意的是proxy暂时只能解决开发的跨域访问问题，如果在生产上发生跨域问题的话，需要将类似的配置转移到Nginx容器上



**样式**

Umi默认支持LESS，SASS,SCSS样式的导入，你可以按照引入CSS文件的方式引入并使用

我们的脚手架代码：

```
import lessStyles from './index.less';
```

然后后续引用：

```
<div className={lessStyles.container}>
  <Guide name={trim(name)} />
</div>
```

这是./index.less文件下的内容：

```
.container {
  padding-top: 80px;
}
```



#### 页面实现

**Home index.tsx**

这里通过usemodel用到了个全局状态模型：在models/global.ts定义

**Access index.tsx**

 这里通过useAccess() 调用了access.ts的方法，这里面有入参initialState

这个值在app.ts里getInitialState定义 初始化时候调用

**Table**

component实现：

1、CreateForm

可以看到在文件里定义了props接口

然后借这个props定义了创建的表单



2、UpdateForm

导入部分：

导入了几个Ant Design Pro组件



3、然后看index.tsx

这里面引入了很多ant-design/pro的组件 后续如果遇到如何使用相关组件

可以上https://procomponents.ant.design/components/table去查询





### 七、从IM二期前端代码解析

这一期代码主要看pages/home

现在我们开了二期线上的权限，可以参照着看



#### 请求发起

首先对于api的定义都在services的api.tsx下面

对外export async function，然后后续调用，然后注意有些方法有params入参，这就靠后续组件来封装



这里是组件内的调用，这种写法使用了对象解构赋值的语法，并给某些解构出来的变量提供了默认值

```
const {
  data: indexDetails = [],
  run: runIndexDetail,
  mutate,
  loading: detailLoading,
  status: detailStatus,
  refresh: refreshDetail,
} = useRequestWithStatus(queryIndexDetail, {
  manual: true,
  onSuccess: (data) => {
    if (data.length > 0) {
      setSelectedDetail(`${data?.[0].index}`);
    } else {
      setSelectedDetail(undefined);
    }
  },
});
```

然后这里的useRequestWithStatus是一个自定义的React Hook，然后我们解析一下这个hook

```
import { useRequest } from '@@/plugin-request'; 导入请求Hook
import { STATUSENUMS } from '@/constants'; 导入常量

const useRequestWithStatus = (service, options) => {
  const { data, error, loading, ...rest } = useRequest(service, { ...options});
  const status = loading ? STATUSENUMS.LOADING : error ? STATUSENUMS.FAILED : data && data.length > 0 ? STATUSENUMS.SUCCESS : STATUSENUMS.NO_DATA;

  return { status, data, error, loading, ...rest };
};

export default useRequestWithStatus;
```





#### 工程目录组织

config/config.ts：配置文件

mock：模拟api，

src/assets ：静态资源

scripts: captain的启动脚本

src：

​	assets:静态资源

​	components:组件

​	hooks：自定义钩子

​	constants:常量

​	services:服务api定义

​	utils：下载 cookies方法



#### 组件返回拆解

src/components:	

formDisplayItem 



div和span是html中非常常用的标签，div是大的块级元素用于较大区域的布局和分区，span是行内元素，多个span在同一行内显示



#### 变量与函数拆解

![image-20240923124442799](D:\note\工作文档\知识库\assets\image-20240923124442799.png)



#### div标签拆解

home这个大组件 返回的逻辑 外面被

return(

<>
 {

}   
<>

)

包裹住

而里面如果没有权限 --- 返回NoPermission组件

如果有权限返回：

1、查询条件 一层div



2、然后是筛选框 一层div 这一层里嵌套Card 下一层嵌套ProFrom表单 

然后接Flex 在里面是ProFormSelect筛选项



这里有个ref属性 是用来做锚点定位的



#### 组件解析-antd

**Card**

通用卡片容器，可以承载文字列表等

这里有个useRef的Hook，允许你访问和操作组件实例，

**Flex**

对齐的弹性布局容器 适合设置元素间的间距 和各种水平垂直对齐方式

与Space组件的区别：

Space为内联元素提供间距，Flex为块级元素提供间距

**Form**

表单，项目内使用antd-pro的ProForm



#### 组件解析-antd-pro

其实antd-pro更多是antd的升级，大部分api其实完全可以在antd里找到答案

**ProFormSelect+ProFormDependency**

前者是选择框组件 用于在表单中创建下拉选择框

后者是依赖组件，用于监听其他表单项的变化，并根据这些变化动态更新自身值或状态

这里可以参考protest组件



#### 组件解析-modal





#### 自定义hook-useRequestWithStatus 

**解构赋值语法：**

可以从数组或对象中提取值，并将其赋值给变量，使得代码简洁易读

数组解构示例：

const [a, b] = [1, 2];
console.log(a); // 1
console.log(b); // 2
对象解构示例：

const { name, age } = { name: 'Alice', age: 25 };
console.log(name); // Alice
console.log(age); // 25

对象解构重命名

const { name: userName, age: userAge } = { name: 'Alice', age: 25 };
console.log(userName); // Alice
console.log(userAge); // 25

顺便提一下：这里解构还可以设置默认值



**useRequest**

处理异步请求的一个React Hook，封装一些常见的异步操作和状态管理逻辑

import { useState, useEffect } from 'react';

function useRequest(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch(url);
        if (!response.ok) {
          throw new Error('Network response was not ok');
        }
        const result = await response.json();
        setData(result);
      } catch (error) {
        setError(error);
      } finally {
        setLoading(false);
      }
    };

   fetchData();

  }, [url]);

  return { data, loading, error };
}

export default useRequest;



**useEffect**

用于在函数组件中处理副作用

useEffect(() => {
  // 这里是副作用代码
  return () => {
    // 这里是清理代码
  };
}, [依赖项数组]);

两个参数：第一个是执行副作用代码，用于依赖项变化时清理副作用

第二个是依赖项数组，决定了副作用函数何时执行，如果数组中某个依赖项发生变化，就会重新执行。

如果依赖项数组为空，副作用函数只会在组件挂载和卸载时执行



在useRequest中的使用，

import { useState, useEffect } from 'react';

function useRequest(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch(url);
        if (!response.ok) {
          throw new Error('Network response was not ok');
        }
        const result = await response.json();
        setData(result);
      } catch (error) {
        setError(error);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
  }, [url]); // 依赖项数组包含 url，当 url 改变时重新执行副作用

  return { data, loading, error };
}

export default useRequest;

通常用于组件挂载时发起数据请求



看下这里引入的代码：

```
const {
  data: indexOverviewList = [],
  run: runIndexOverview,
  loading: overViewLoading,
  status: overViewStatus,
  refresh: refreshOverview,
} = useRequestWithStatus(queryIndexOverview, {
  manual: true,
  onSuccess: (data) => {
    if (data?.length > 0) {
      setSelectedIndex(data?.[0].index);
    } else {
      setSelectedIndex(undefined);
    }
  },
});
```

这里传入了manual：true 意味着需要手动调用runIndexOverview来触发请求



下面来看看怎么接入



**map用法**

有点像stream的map算子

函数可以接受三个参数：

currentValue：当前被处理的元素

index：当前元素的索引

array：调用map方法的数组本身



#### 吸顶的实现

```
<div
  style={{
    opacity: isShowFormAffix ? 1 : 0,
    position: 'absolute',
    top: 0,
    left: 0,
    backgroundColor: '#232c3e',
    zIndex: isShowFormAffix ? 1000 : 0,
    transition: 'opacity 0.3s',
    maxHeight: 1000,
    height: '32px',
    // height: isShowFormAffix ? 'auto' : 0,
    padding: '8px 32px',
    width: '100%',
  }}
>
  {
    <Flex gap={8} wrap={'nowrap'} align={'baseline'}>
      <FormDisplayItem label="BU" value={buListEnum?.[form.getFieldValue('bu')]} />
      <FormDisplayItem label="日期"
                       value={form.getFieldValue('dateType') == 'HOLIDAY' ?
                         getDisplayHoliday(holidayOptions, form.getFieldValue('holiday'), form.getFieldValue('year'))
                         : form.getFieldValue('date')?.map(d => d.format(dateFormatDisplayEnums[form.getFieldValue('dateType')])).join(' ~ ')} />
      <FormDisplayItem label="地区" value={regionEnum?.[form.getFieldValue('region')]} />
    </Flex>
  }
</div>
```

可以看到通过一个div的方式来接入

div的style里面封装一个是否展示的css来实现

```
<div style={{
  position: 'absolute',
  top: 32,
  left: 0,
  backgroundColor: '#232c3e',
  zIndex: isShowOverviewAffix ? 1000 : 0,
  maxHeight: 1000,
  opacity: isShowOverviewAffix ? 1 : 0,
  transition: 'opacity 0.3s',
  height: 180,
  display: 'flex',
  flexDirection: 'column',
  gap: 8,
  width: '100%',
}}>
  {<ItemCarousel data={indexOverviewList} showYoy={showYoy} showMom={showMom}
                 selectedIndex={selectedIndex}
                 onClick={handleIndexSelect} />
  }
  <div style={{
    height: '1px',
    background: '#424d63',
  }}></div>
  <div style={{
    padding: '0px 24px 8px 24px',
  }}>
    <Segmented
      value={selectedDetail}
      options={options}
      style={{
        border: '1px solid #2e343b',
      }}
      onChange={handleMenuClick} />
  </div>
</div>
```

这里是第二个标签页的实现

从这第二个标签页这里可以看到用了antd的Segmented分段控制器：

#### 组件解析-Segmented

用于展示多个选项并允许用户选择其中单个选项。

import React from 'react'; import { Segmented } from 'antd'; const Demo: React.FC = () => (  <Segmented<string>    options={['Daily', 'Weekly', 'Monthly', 'Quarterly', 'Yearly']}    onChange={(value) => {      console.log(value); // string    }}  /> ); export default Demo;

我们可以在onchange里来实现锚点的功能

然后我们想一想怎么实现吸顶隐藏

首先通过ahooks去设置显示比例，然后外层包一个<div> 然后styles去控制

div的里面就放segmented去实现锚点，然后再加一个加载的按钮，就可以实现这个功能了

#### 钩子研究--`useMemoizedFn`





### 八、IM一期迭代实现

#### 筛选项功能开发：

待完成：

1、接入后端数据 现在来看 因为我们要实现用户所在Bu优先 所以BU的数据得从后端出

2、时间框选择--->没有数据的日期要禁用掉 

3、节假日维度 要加一个展示 完成



这里我们得修改一下后端实现的接口了

我们只返回需要的字段，所以前端需要拿到哪些数据呢？

- 是否有权限
- BUList
- BU最新日期数据
- 节假日日期



目前已经成功改造完后端接口，现在准备接入后端服务



#### 概览页面开发

![image-20240926140209360](D:\note\工作文档\知识库\assets\image-20240926140209360.png)



现在就是说开发这个模块：

目前想的是 基于现有组件自己去实现一个组件来展示

但是怎么根据现有的一些组件去封装出来呢？

还有这个趋势图应该怎么做呢？



鹏飞答疑：

这个实现的思路没有问题，趋势图的话其实这里已经是一个真实的趋势图了，只不过省略了一些信息，另外这个右移动实现思路是固定宽度让前端自适应，并且这个表格实现上其实可以这么做：拆分成三个表格，我们可以使用React的antd组件



那么就必须要熟悉一下antd的table组件了



芹林答疑：

可以使用echarts组件来实现 附上了代码和官方调试器



这么看来 调研后 是有方向实现的



目前实现难点：

1、表头加样式  这个可以修改title

2、label+指标名  这个封装一下 tag

3、是否标红 修改echarts的点参数

4、hover点----使用Tooltip



我们现在就先实现一个 指标概览数据 即 一个card里 封装一个指标的数据

一个指标要完成三个看板



看板的实现我们参考react官方





#### 组件研究--table

展示行列数据



#### 组件研究--tag

我们在标签页上加一个组件



#### 框架研究--echarts

https://echarts.apache.org/examples/zh/editor.html?code=PYBwLglsB2AEC8sDeAoWsAeBBDEDOAXMmurGAJ4gCmRARAMYCGYVA5sAE7m0A0J6AE2aMiAbVoBZGL1i0AKgFcqM2gHUqAlXIAWClQDEOEFQGVmphdFoBdEgF8-6cjnxEkDknipGqhWKNRSWCEwEX8ADgAmAAYeWABOAGZIuPjogEZUxIAWOPTItLzExNjYdOTo60dSCmo6ABsIaGVq9ABbRg4AawAFYCawN35SELCA4aDamlkOjFoJ2DsqhbxyNoAjYHq6ECb5oI8g1Y2tumgYFuGIFjaTCnrpwKDYei3OOg5WdcYACnTolJlADseRBZQAlPtSHZ7LY7EA&_source=echarts-doc-preview



研究一下官网

https://echarts.apache.org/handbook/zh/how-to/chart-types/line/smooth-line

平滑曲线图：折线图的变形，只需要将折线图的smooth属性设置为true即可



纵坐标默认是数值 横坐标设置为类目型



折线图样式设置--这里异常点要出红色点，可以通过series.itemStyle来设置颜色

很好 这样明天就可以把第二个表格做上去了



1、契约变了 前后端交互方式变了

2、后端服务改了 前端页面改了 调样式



#### Hook研究-useEffect

用于在函数组件中执行副作用操作，包括数据获取

接受两个函数

- 必选 执行副作用的代码
- 依赖函数 当数组中变量发生变化时，副作用函数会重新执行，如果没有提供依赖数组，在每次渲染后执行，如果提供了一个空数组，副作用函数在组件挂载和卸载时执行一次



**开发进展：**
第三个表格接入：



todo:

研究一下两个组件怎么对接起来？

然后是看下年份展示这块怎么去做

然后导航栏



#### 趋势详情开发

这里我们参考使用modal组件来实现：

这一块其实很像二期里的详情页，代码可以做参考一下：即二期的detail组件



开发进展：
1、首先在echarts里把折线图先画出来

**如何让纵坐标展示百分比**

需要在series的data属性中使用百分比数值 并确保y轴的axisLabel显示百分比符号

**展示每个点**

```javascript
 symbol: "circle",  // 修改为显示每个点
```

实现当鼠标移动到某个点时展示数值

**加按钮切换**

`<input type="radio" value="dataset2" name="dataset" onChange={handleRadioChange} checked={selectedDataset === 'dataset2'}/>`

value：单选按钮的值

name：所有具有相同name属性的单选按钮被视为一个组，用户只能从中选择一个

onchange：当单选按钮状态发生变化，会调用这个函数

check：标识单选按钮是否被选择



思考一下实现思路：

这一块其实本质上是一个弹窗页面：

大概要完成的功能点：

1. 弹窗接入
2. 弹窗页面的实现-title
3. 三个label和echarts联动
4. 一个table



#### 组件研究--Modal

展示一个对话框，提供标题，内容区，操作区，可以在当前页面打开一个浮层承载



#### 钩子研究---useInViewport

这是ahook库中的一个自定义钩子，用于检测一个DOM元素是否在视口内，非常适合用于实现懒加载、元素吸顶等功能

使用接入：

1、安装ahooks库 然后再组件内使用useInViewport钩子



useInViewport接受两个参数：

1、ref：一个React的ref对象，指向要检测的DOM元素

2、options（可选）：配置对象，可以包含以下属性：

- threshold: 阈值，表示元素进入视口的比例，取值范围为0-1，默认为0
- root：指定视口的根元素，默认为window
- rootMargin：视口的外边距 （即可以扩展或缩小跟元素的边界）

返回值：一个数组，包含两个值：

1. inViewport：boolean表示元素是否在视口内
2. ratio：表示元素在视口内的课件比例



#### 钩子研究--useMemo

```javascript
const memoizedValue = useMemo(() => {
  // 计算过程
  return computedValue;
}, [dependency1, dependency2, ...]);
```

接受一个创建函数和一个依赖项数组，当依赖项数组中的值发生变化时，useMemo会重新计算并返回创建函数的方法值，如果依赖项没有变化则会返回上一次计算的值

我们可以用这个来监听曝光度、从而实时计算是否展示这个变量

这就是二期实现的原理



#### 钩子研究--useRequest

这是一个用于处理异步请求的React Hook，第一个参数是一个请求函数，返回一个Promise

第二个参数是一个配置对象，可以包含各种选项，比如请求成功或失败的回调函数、请求的依赖项等

useRequest返回一个对象，通常包含请求的状态loading、请求结果data、和一些方法如run

这里的run就是第二个参数options里manual为true时 手动跑



#### 滚动栏研究

其实我们想实现的是隐藏滚动条仍然允许内容滚动

可以再参照着二期的实现来做 看下为什么我们会



### 九、一些前端开发Tips

在Js和Ts中访问对象的属性有两种方式：

1、点操作符.

2、方括号操作符[]

同时在TypeScript中使用点操作符编译器会类型检查，方括号操作编译器无法检查



Ts里的变量有哪些类型？

基本类型：

boolean、number、String、null/undefined

复杂类型：

Array

Any：任意类型



当你遇到各种奇怪离谱的数据问题时：前端调试好方法：console.log然后在控制台看数据



在React框架里，key属性的作用是什么？

标识数组里的元素，当渲染列表时key帮助react识别哪些元素发生变化



JavaScript里返回对象和返回JSX元素语法：

这个吸顶的实现

继续看下 权限控制实现



然后再看看这个图标的实现，怎么调整成视觉稿那样



然后可以很明显的看到 表格里的趋势图数据不对 看下怎么接入 



还是再看一下吸顶的实现



day6

看下IM这边还剩哪些：

1、吸顶bug修复 -- 页面抖动

2、万比订单--改造 这两个变成四位小数点

3、交互调整&视觉调整

4、前端跑流水线



这个四位小数点其实 前端根本不要动 改前端即可



现在有些点很尴尬，就是不能改key，改了的话本地没数据

总结一下现在前后端要怎么修改：

1、后端指标计算逻辑改变

本质上后端数据加工本质就只有日维度的分子值--分母值，我们想一下处理过程

根据筛选条件



先修一下页面的bug

这里其实可以参照芹林筛选项吸顶代码的实现



```
<div id={'imScrollContainer'} style={{
  height: 'calc(100vh - 100px)',
  overflow: 'auto',
  scrollbarWidth: 'none',
}}>
  <Card
    ref={formInViewPortRef}
    bordered={false}
    style={{
      marginBottom: 16,
    }}
    styles={{
      header: {
        color: TextColor.Text2nd,
      },
      body: {
        padding: 16,
      },
    }}
  >
```



然后吸顶的div：

```
<div
  style={{
    opacity: isShowFormAffix ? 1 : 0,
    position: 'absolute',
    top: 0,
    left: 0,
    backgroundColor: '#232c3e',
    zIndex: isShowFormAffix ? 1000 : 0,
    transition: 'opacity 0.3s',
    maxHeight: 1000,
    height: '32px',
    // height: isShowFormAffix ? 'auto' : 0,
    padding: '8px 32px',
    width: '100%',
  }}
>
```



实现思路：给filter包一层card，card引用ref，然后外层包div





