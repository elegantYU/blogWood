---
title: webapck
date: 2023-11-08 14:33:11
tags:
---

webpack是一个打包工具，他会把js，css，图片，html文件，分析整个文件的目录结构，确认文件之间的依赖，通过编译，转换，生成js格式的bundle文件。他把所有的文件当成模块。

## webpack打包过程
1. 根据entry入口文件，调用所有配置的loader对象进行编译
因为webpack只能处理js，css等其他格式需要loader辅助进行编成js后进行打包
2. 利用babel将js转为ast，在用babel-traverse对ast进行遍历
3. 遍历的目的是为了找到import引用节点，也就是依赖关系
4. 根据引用节点，引入模块，生成唯一标识的id，并将解析过的模块缓存起来。如果其他地方也引入了模块，则不需要重新解析，生成一个依赖图谱。
5. 递归遍历所有的模块，根据依赖图谱，生成一个个chunk（包含多个模块）
6. 将生成的文件输出到output中。

## Tree-shaking 原理

基于 ESM 模块规范，在运行过程中静态分析模块之间的导入导出，确认ESM模块中哪些导出值未被其他模块使用，并将其删除。

- 收集模块导出变量并记录到模块依赖图中
- 遍历模块图，标记模块导出变量有没有被使用
- 生成产物时，若变量未被引用则删除其对应的导出语句

[更多细节点击这里](https://juejin.cn/post/7002410645316436004)

## webpack 压缩代码的原理

[点击链接查看](https://q.shanyue.tech/fe/js/138)


## 热更新的原理

> 先介绍功能点

webpack-dev-server 主要包含了三个部分：
1. webpack: 负责编译代码
2. webpack-dev-middleware: 主要负责构建内存文件系统，把webpack的 OutputFileSystem 替换成 InMemoryFileSystem。同时作为Express的中间件拦截请求，从内存文件系统中把结果拿出来。
3. express：负责搭建请求路由服务。

> 介绍工作流程

对代码进行修改的时候，webpack会对代码进行重新打包;通过 webpack 创建的 complier 实例， 可以往`compiler.hooks.done`(webpack 编辑完成的钩子)注册事件;当监听到一次 webpack 编译结束，会通过 websocket 向客户端发送 hash 和ok 事件;客户端接收服务端的推送信息后，对比当前 modules tree ，再次发请求到 Server 端获取新的JS模块;获取到新的JS模块后，会更新 modules tree并替换掉现有的模块;最后调用 module.hot.accept() 完成热更新.

<%note primary%> 
热更新在webpack里主要是根据websocket，建立双方的通信。当代码发生变化时，通知浏览器请求新的模块，替换原来的。
<% endnote%>
#### webpack
HMR的核心就是客户端将新的文件从服务端拉取，也就是chunk diff
wds（webpack-dev-server）和服务端之间建立一个websockt连接，进行通信。
本地资源发生变化后，wds就向服务端发起更新，带上hash，让浏览器进行对比。
客户端对比出差异后，就会向wds发送ajax请求获取更新的内容。

当一个文件发生变化，webpack监听到这个变化对文件重新编译打包，生成hash值，

**是全量更新，即便更改一个文件，也会重新编译整个文件。**
#### vite
以原生esm形式服务源码，只要在浏览器请求源码的时候获取源码进行转换再返回转换的源码，所以比webpack热更新更迅速。
<% note primary %> vite是增量更新，只更新修改的**
<% endnote%>
1. 利用原生的esmodule模块，开发时可以跳过打包过程，提升编译效率
2. 当通过import加载资源，浏览器发起http请求时，vite会拦截http请求，返回对应的文件。

### webpack 和 vite 热更新的区别
1. wepack的热更新，是在某个依赖更改的情况下，会将该依赖所处的整个module都重新更新，也就是说如果依赖越来越多，编译的速度会慢。
2. vite：如果某个文件发生变化，只会更新这个文件。其他文件都无需更新。


webpack是一个静态资源打包文件，将项目文件组合编译在一起。

是一个打包工具。

他的本身功能比较少，只能处理js文件。

#### 五大核心概念

- entry  入口
- output 编译好的文件放置位置
- loader 除了js json等资源的 其他输入
- plugin 扩展功能
- mode 开发模式 生产模式  等



#### 开发模式作用

1.编译代码

2.代码质量检查，代码规范



#### 处理样式文件

webpack不能识别样式资源，借助loader实现

如果不用css-loader style-loader等loader处理器 会编译失败 需要一个loader解析

---

处理图片资源

wepback4: file-loader url-loader

将10kb以下的图片转为base64格式 可以减少请求数量 但是相应体积变大  转成base64的图片 已经压入了js中

更改配置 需要删除dist内容  自动清空上次打包结果：output→clean:true

#### 处理字体资源

```JavaScript
{
  test：/\.(ttf|woff2?)$/
  type:"asset/resource",
  generator:{
    filename:"static/media/[hash:8][ext][query]
  }
}
```

asset/resource:  相当于file-loader， 将文件转成能识别的资源

asset：相当于url-loader， 转成识别资源，同时小于某个大小的资源会处理成data-url

#### 处理js资源 plugin

代码格式 ： eslint

js兼容：babel

---

先完成eslint检测 再用babel

#### Eslint

- parseOptions  解析选项

  sourceType es模块化

  ecmaFeatures :{jsx:true} 其他特性

  ecmaVersion: es版本

- rules 具体规则

  0 —off 

  1—warn

  2—error

- extend  继承现有规则

在webpack5中eslint作为一个插件  需要用new调用

#### Babel 

将es6语法编写的代码向后兼容，其他浏览器

```JavaScript
{
  preset:[]
}
```

babel-loader

#### 处理html资源

html-webpack-plugin

自动引入打包生产的js文件 内容和源文件一致

#### 自动化& 开发服务器  

```JavaScript
{
  devServer:{
    host:
    port:
    open:// 自动打开浏览器
  }
} // 平级 entry
```

指令→npx webpack serve

#### css提取

mini-css-extract-plugin

css打包到js文件中，会创建一个style标签，造成闪屏 

所以提取单独的css文件， 通过link标签加载性能

- loader：style-loader去掉 改为 mini的loader
- plugin：加载 插件

#### css压缩 
css-minimizer-webpack-plugin

## 高级webpack优化

#### 提升开发体验  —-sourceMap：源代码映射

#### 提升打包速度

1.HotModuleReplacement HDR 热模块替换

webpack默认将所有模块都进行打包 

```JavaScript
devServer:{
  hot:true
}
```

避免整个页面刷新

2.OneOf 只匹配上一个loader

3.exclude include

4.cache 缓存

5.thead 

多进程打包

每个进程启动都有时间 所以要在特别耗时的操作中进行 启动进程的核数就是cpu的核数

#### 减少代码体积

1.Tree-shaking 移除js中没用上的代码 依赖es module

2.babel  plugin-transform-runtime 禁用babel自动对每个文件注入runtime

3.Image minimizer   

#### 优化代码运行性能

1.Code split 代码分割 生成多个js文件 按需加载

- 多入口

- 提取公共代码

- 按需加载

- 单入口  pulgin:['import']

- 给动态导入取名字

2.preload prefetch

按需加载带来的卡顿  在浏览器空闲时间加载后续使用资源

preload： 立即加载 优先级高

prefetch： 空闲加载 优先级低

共同点：加载不执行 缓存

不同： preload只加载当前页面

插件 preload-webpack-plugin 

3.Network Cache 更新文件名 读取缓存

4.Core-js

async promise babel无法解决

所以需要core-js 解决兼容性问题 做es6以上的api的polyfill

5.PWA 提供离线体验 通过service workers

# 总结

- 提升开发体验

  SourceMap  准确的错误提示

- 提升打包速度

  hdr 热更新

  oneof

  include exclude

  cache

  thead

- 减少代码体积

  treeshaking

  禁用babel自动导入runtime

  本地图片压缩

- 优化代码

  code split 代码分割 多入口 按需加载

  core-js 提升兼容性问题 

  preload prefetch 提前加载

  network cache 缓存

  pwa 离线访问

# loader :帮助webpack识别不能识别的模块

执行顺序： pre， normal， inline， post 从右到左， 从下到上

### 开发一个loader

```JavaScript
module.exports= function loader1(content){
  return content
}
```

#### loader接受的参数 

需要返回一个webpack要输出的东西，loader本质是一个函数

content：源文件内容 

map：sourceMap 

meta：数据 

默认normal-loader

#### loader的分类

同步 异步 raw pitch

raw：buffer数据流 处理图片文件

pitch:提取终止

pitch顺序:从左到右 中断返回上一个



#### 手写babel-loader





## plugin：插件

plugin找到webpack的钩子事件，介入控制编译



Tapable暴露的三个给插件的方法

- tap: 可以注册同步钩子和异步钩子
- tapAsync 回调方式注册
- tapPromise: Promise方式



Plugin 构建对象

1.Compiler  保存完整的webpack环境配置

2.Compilation 一次资源的构建 资源处理 对构建依赖图中所有模块进行编译

![](https://secure2.wostatic.cn/static/h6nhe4sbHRRSpVuZPD8xep/image.png?auth_key=1700037699-sDvdM4jTGfxTqto6uqXnWc-0-451a9aa841a3d84b3b32a1b4ee3848fd)



#### 生命周期执行顺序

- webpack加载webpack.config.js所有配置，此时就会new一个实例插件， 就会执行constructor
- webpack创建compiler对象
- 遍历所有plugins的apply方法，调用
- 执行剩余编译 触发各个hooks



钩子：make  afterCompiler emit

#### 注册hooks

```JavaScript

class TestPlugins{
  constructor(){
    console.log('constructor')
  }
  
  apply(compiler){
    compiler.hooks.emit.tap('testPlugin',(compilation)=>{})
  }
}
```

## loader和plugin的区别

loader：文件加载器，加载资源文件，进行编译 压缩

plugin： 插件，解决loader无法解决的事情

不同：

loader：在打包文件前处理

plugin：贯穿了webpack的编译周期

#### 编写loader的本质：

loader本质是一个函数 他不能用箭头函数写 因为他的this作为上下文被webpack填充了

函数有三个参数 content map meta

是作为源文件内容的 

他有四个类型  同步 异步  raw pitch

异步的话 ： const callback = this.async()  需要return

#### 编写plugins的本质

plugin在创建过程中会产生两个核心的对象

1.compiler  包含了所有配置信息

2.compilation  作为回调的参数 资源的构建

插件 是一个函数 或者是 包含apply方法的对象










