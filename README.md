[toc]

# 项目背景

小组项目作业，我们小组做的项目是校园跑腿系统，系统分为前台（面向用户）和后台（面向管理员）两个部分。前台系统是所有用户的系统入口（包括管理员），主要提供给普通用户发布、浏览和接受跑腿订单；后台系统则是面向管理员，用于帮助其掌握系统运营信息的。

由于我们组采用的是前后端分离的开发模式，且我负责的是系统前台部分的前端开发工作，所以本项目只涵盖**校园跑腿系统前台部分（面向用户）的前端代码**。

这是我第一次用Vue3做项目，可以说这个项目有很多不完美的地方需要改进，但是Vue3和Vue2相比少了很多可参考的经验

**复盘从这个项目中学到了什么，自己总结出一些方法论，再从不足中确定自己需要学习的方向**，这才是我认为重要的地方，也是写下这篇文档的初衷。

本文档作为复盘，只会进行一些总结，具体的项目细节在源码中含有大量注释，请结合食用。前端新手捡破烂，希望有大佬指点迷津~

## 技术栈

- 编程语言：JavaScript
- 前端框架：Vue3.x
- 路由工具：Vue Router 4.x
- 状态管理：Vuex 4.x
- UI组件库：Element Plus
- HTTP库：axios

## 项目展示

由于我们组的部署工作只完成了前端部分，所以不能线上预览系统

在此简要概述一下前台各页面功能：

- 首页：展示随机六条订单（提供换一组功能）、搜索订单、个人中心入口、发布订单
- 个人中心页：修改个人信息、修改密码、提交投诉反馈、查看我发布的/我接受的/历史浏览订单
- 订单详情页：接单、修改订单状态、修改订单信息（根据订单和用户的状态展示不同信息）

未实现：我的投诉、线上支付。

# 如何规划项目结构

```
     |- /assets 资源文件夹
       |- /css
       |- /fonts 字体图标
       |- /images 存放网页结构相关图片
     |- /components 可复用组件文件夹
       |- /common 存放可被其他项目复用的组件
       |- /content 存放本项目多处复用的组件
     |- /router 路由配置
       |- index.js
     |- /plugins 插件
     |- /utils 工具文件夹
     |- /store vuex配置，项目简单所以没有分模块配置
       |- index.js
       |- getter.js
       |- mutations.js
       |- actions.js
     |- /views 页面组件文件夹，内部二级文件夹以页面来划分，再存放子组件
     |- /network 封装网络请求
       |- request.js 通用网络请求配置，目录下子文件都要导入
       |- home.js/cart.js ... 按照模块划分网络请求配置文件
```

# 准备工作

## 基础样式

1. 引入[normalize.css](https://github.com/necolas/normalize.css)，让样式兼容各个不同的浏览器
2. 完成`base.css`，定义一些基础共用样式并引入进`main.js`或`App.vue`
3. 如果有的话，引入组件库的主题样式

## 相关配置

### Webpack配置

1. 目前只接触了别名配置：可以在Webpack配置中指定文件夹的别名，方便修改路径。

### axios配置

1. 通过`axios.create({...})`来新建一个axios实例并配置`baseURL`等属性，通过实例来封装网络请求
2. 通过`instance.interceptors.request/response.use(config =>{...}`来配置请求/响应拦截器
   - 请求拦截器可以判断用户的登录权限并进行处理，有登陆权限就让请求头携带token
   - 响应拦截器可以先判断响应的`status`（200则调用接口成功），再通过`code`去判断一层业务的逻辑是否合法，在此处可以统一对后端返回的错误信息进行提示

### 路由配置

1. 定义全局前置守卫，可以判断用户是否有跳转路由的权限（没登陆就跳去登录页）

   > 注意区分此处的判断用户登录权限和axios中的，一个是页面跳转的权限一个是发送网络请求的权限

2. 定义全局后置守卫，可以利用后置守卫修改document的title，注意后置守卫没有`then`方法

### 定义Vuex全局状态管理数据

1. 存放一些全局公用数据，比如订单种类、订单状态对象等等
2. 注意存放用户信息时，刷新会导致Vuex里的信息丢失

### 跨域代理

1. 在`vue.config.js`中利用代理对象，把请求交给后端代理来解决跨域问题

# 如何划分页面逻辑

> 刚开始做项目时，打开一个页面总会有不知道从哪里开始下手的感觉，做完以后就有了一点自己的小心得，在此把心得进行总结。

1. **第一步，在`<template>`中规划好页面块，用注释标清楚页面的每一部分。**

   这一步很重要，在前期能快速厘清页面结构，弄清楚**“这个页面是做什么的**，由哪几块组成”；在测试以及后期复盘时期能帮助自己快速定位到某一部分。

2. **第二步，明确页面需要的数据。**

   了解了页面是做什么的以后，就要明确——为了达到该页面的目的，**需要有哪些数据**，再进一步确定**如何获得这些数据**。

   比如订单组件（Order），这个组件的目的是简单展示订单信息，那么它的页面结构就有两块：发单人信息和订单信息。我们可以由此确定这个组件需要的数据是一个`User`对象和一个`Order`对象。

   那么这两个数据的来源是什么呢？数据的来源通常**有父组件向子组件传递**、**通过网络请求获取**、**从Vuex中获取**等等。由于Order组件是作为子组件被多处复用的，所以我选择了通过`props`接收父组件传来的`Order`对象，再利用`Order`对象里的`publisherId`属性值，通过发送网络请求来获取`publisher`的数据（即user对象）。

3. **第三步，搞清楚用户和数据之间有哪些交互。**

   很多情况下用户是期望能操作数据的，比如修改订单信息、修改个人信息等等。有交互就会导致数据的变动，因此明确交互实际上是在**明确如何操作数据**，包括监听、修改等等。

   明确了如何操作以后，对于所写的方法一定要再梳理一遍逻辑。

4. 第四步，**仔细检查页面展示和使用的数据**。

# 注意事项

1. **一定一定要记得写注释**！否则后期再测试修改什么的非常火葬场，也不利于团队协作。
2. 如果说Vuex是全局的状态管理模式，那么`setup`里定义的`state`对象就是利用了单页面状态管理的思想，`state`对象里的属性可以是状态需要被管理的某个变量或对象。
3. 获取数据时要注意组件的生命周期顺序，还要注意网络请求是异步发送的，**避免数据在获取之前被使用**的报错。
4. 使用开源组件库时，要仔细阅读API。

# 总结

## 我学到了什么

Vue3的组合式API相比于Vue2的选项式API，我认为更贴近我以往编程的习惯，在写项目的过程中进一步领会其思想，对我来说是一个不小的收获。在这个项目之前我一直在学习Vue的理论知识，这个小小的新手村实战项目帮助我对理论进行实践，使我对Vue的使用更加得心应手，踩过的很多坑也让我有了不少经验。

最重要的是，我了解了做一个项目是什么流程，总结了自己应该怎么做的更好。

## 我的不足之处

在此针对不足总结一下接下来的学习方向：

- Webpack再学习
- JS异步
- ES6再学习
- Vue异步组件（在没有进一步扎实JS基础之前，不打算深入学习Vue）
- axios深入学习
- http协议
- 利用git掌握项目开发进度
