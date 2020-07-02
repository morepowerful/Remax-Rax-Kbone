# Remax、Kbone、Rax小程序方案比较

## 背景
由于各大平台推出自己的小程序（微信、头条、支付宝、钉钉等）。再加上每个平台之间又有大大小小的差异，这对于有跨平台需求的小程序来说无异于是一场灾难。所以市面上推出各种小程序框架（Taro、Kbone、Remax、Rax等）。

## 先来了解下小程序架构
 小程序本质上是运行在 webview 上的一个 H5 应用，我们代码分别跑在render（视图）线程和worker（js逻辑）线程上，render线程由平台把控。我们的view 放在 render 线程，平台提供了一种特殊的语言*xml来写 view，并且不能写入 js 代码（能写被限制过的js），逻辑就放在 worker 线程，并且worker 不能操作 dom，这样做的目的是为了方便平台方管控，以及避免开发者操作dom影响性能。

![架构如下：](https://github.com/morepowerful/Remax-Rax-Kbone/blob/master/imgs/%E6%97%A0%E6%A0%87%E9%A2%981.png)

## 市面上的小程序框架底层实现类型
- 编译时: Rax、Taro、uni-app、..
- 运行时：Rax、Remax、Kbone..

## Kbone原理分析
 kbone 就是在 worker 线程适配了一套 js dom api，因为上层不管是哪种前端框架(react、vue)或原生 js 最终都需要调用 js dom api 操作 dom，适配的 js dom api 则接管了所有的 dom 操作，并在内存中维护了一棵 dom tree，所有上层最终调用的 dom 操作都会更新到这棵 dom tree 中，每次操作(有节流)后会把 dom tree 同步到 render 线程中，通过 wxml 自定义组件进行 render。
 ![架构如下：](https://github.com/morepowerful/Remax-Rax-Kbone/blob/master/imgs/%E6%97%A0%E6%A0%87%E9%A2%982.png)

 ### Worker线程
 worker运行上层框架转化成的小程序代码（需要一个webpack插件把web端编译的代码转化为小程序目录结构的代码），并适配啦js dom api和定义一套数据结构来描述dom tree（根据上层框架自身调用的dom api来生成一套Kbone定义的结构数据）；
   js dom api其实就是重新去实现dom api，这些函数用来操作自己存在内存中的dom tree；（自己模拟一个document全局对象）。

   **dom树结构如下：**
   ![架构如下：](https://github.com/morepowerful/Remax-Rax-Kbone/blob/master/imgs/%E6%97%A0%E6%A0%87%E9%A2%983.png)

   ### Render线程
  Kbone定义了一个【Element自定义组件】，用于渲染内存中的dom tree。Element做的事：首先把自己渲染出来，然后再把子节点渲染出来，同时子节点又通过Element渲染出来，这样一直递归下去；
  **简略代码：**
     ![架构如下：](https://github.com/morepowerful/Remax-Rax-Kbone/blob/master/imgs/%E6%97%A0%E6%A0%87%E9%A2%984.png)

  ### 总结下Kbone所做的事情
  1.适配拉一套jsDom；
  2.实现一个自定义组件渲染dom tree；
  3.把上层web端代码转成下程序结构代码；

  ## Remax
 remax 是通过 react 来写小程序，整个小程序是运行在 worker 线程，remax 实现了一套自定义的 renderer，会将react Vdom渲染成它定义的一套Vnode，然后把Vnode转成小程序的data，最后通过小程序提供的 setData 方法传到 render 线程，render 线程则把 vdom tree 递归的遍历出来。

所以也和 kbone 类似，都会在 worker 线程维护一棵 dom tree。

不同点：
    1、kbone是适配了js dom api，上层可以用任何框架，remax是自己写一套react renderer，上层只支持react；
    2、remax在domtree发生变化的时候，不是把整棵树传到render线程，而是计算差异，把差异传到render线程（remax这是利用了react提供的diff完后生成的差异）。
### Remax原理解析
Remax运行时本质是一个通过[react-reconciler](https://github.com/facebook/react/tree/master/packages/react-reconciler)实现的一个小程序端渲染器
![架构如下：](https://github.com/morepowerful/Remax-Rax-Kbone/blob/master/imgs/%E6%97%A0%E6%A0%87%E9%A2%985.png)

### 怎么自定义渲染器
1、实现宿主配置：这是react-reconciler要求提供一些配置项，这些配置定义啦如何创建宿主节点实例、更新、提交等操作。详见[HostConfig](https://github.com/remaxjs/remax/blob/cdc068ecd97d31f611713f3b69df03044de1d6d9/packages/remax/src/hostConfig.ts)。

2、实现渲染函数，类似于ReactDom.render();详见[render](https://github.com/remaxjs/remax/blob/cdc068ecd97d31f611713f3b69df03044de1d6d9/packages/remax/src/render.ts)


## Rax小程序方案
   Rax小程序方案融合了编译时和运行时两套方案，具体项目用哪一个方案在项目配置里加上buildType：runtime/compile；Rax编译时方案：基于AST转译的前提下，再通过词法、语法分析转译成小程序代码。（就是通过工具把RAX代码语法分析一遍，然后把其中JSX部分和逻辑部分抽取出来，分别生成小程序静态模版和小程序页面定义）。这样很明显会有一些问题因为jsx动态能力.例如：

   ![架构如下：](https://github.com/morepowerful/Remax-Rax-Kbone/blob/master/imgs/%E6%97%A0%E6%A0%87%E9%A2%986.png)

   所以Rax又加了个轻量的运行时垫片。但是仍然不能解决根本问题，所以Rax官方文档给出了一些语法约束。

## Rax运行时方案
直接复用啦Kbone的架构，然后再一定程度上做啦些改造；因为Kbone只支持渲染到微信小程序










