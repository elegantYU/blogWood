---
title: react 常见问题
date: 2023-11-02 18:56:00
tags:
---

## react16做了哪些更新

react16之前，虚拟dom的diff算法 理解是一个递归的过程，树如果复杂的话，会堵塞后面的渲染，交互响应迟钝，出现卡顿。因此引入了fiber架构。他引入了任务优先级制度和requestIdleCallback，循环调度。将之前一根筋的渲染方式拆成了两个部分。
1. render阶段 也就是reconciliation 插入高优先级任务，被拆分成一个个小的fiber，在浏览器的**空闲时间**做diff
2. commit阶段 不可打断， 直接执行最高优先级任务 

**由于render阶段可以打断 + 任务优先级问题**，使得很多生命周期被多次执行。因此react17之后componentWillMount,componentWillUpdate, componentWillReceivieProps被取缔。

**引入了workProgressHook和currentHook**，构建了两个workInProgress和current树，两个链表互相引用，如果workInProgress树生成失败，则更新失败，但是页面不会崩溃。

## 为什么 react要用hooks链表呢
如果有多个同个hook调用，还需要有调用的先后顺序，使用next串联所有的hook，同时有一个workProcessHook指针记录当前调用的是哪个hook，方便下次更新时获取上一次更新的对象。在下一次更新时，再次执行hook，就会去找到当前运行节点的链表。

链表的好处在于，即便暂停了，也可以用next记录下一次执行的节点，一旦浏览器有了空余时间，可以直接从这个next开始恢复。
## 为什么只能在函数最外层调用HOOK，不能在循环，条件里调用
因为hook每次调用都会生成一个hook链表挂在fiber.memoizedstate上，按顺序挂的。挂载和更新必须保证是队列是一致的，不然会引起异常报错。

## react hooks的引入解决了什么
react的设计思想： 数据驱动视图。
`view = fn(state)
`
### hooks出现前的逻辑复用
主要是通过hoc和render props 解决，面对复杂业务，会引发**嵌套地狱**和**引发diff算法性能问题**

> 在react16出现之前，函数组件是一个无状态的组件，为什么呢？ 因为16以前只有在类组件更新时会生成一个实例。16之后引入fiber，每个节点都有对应的案例。

### hooks的本质
 **闭包** + **两个链表**

 > 闭包
 > > 有权访问另一个函数里的变量和方法的函数。通过闭包可以突破作用域的特点，将函数内部的方法和变量传到外部。
 
两个链表分别是：
1. hooks链表
2. update链表 是一个环形链表，这样呢  尾部遍历完可以直接找到第一个头，在尾部插入也会快速定位，不需要遍历。

## setState为什么默认是异步，什么时候是同步？

**这里的异步同步不同于promise的那种，指的是setState之后数据会不会立马变化。**
在setState方法中，有一个字段isBatchingUpdates可以判断是否直接更新还是批量。
isBatchingUpdates为true，代表批量更新，也就是异步。**默认为false**
react有一个函数batchedUpdates会把这个值isBatchingUpdates变为true，也就是变为异步。因此只要绕过react事件机制的方法都是同步的。因为react在调用事件处理机制时都会调用这个batchedUpdates这个方法。
**本身也可以通过setstate或者usestate第二个参数返回一个callback，也可以立马获取。**
react17中暴露了这个方法<code>batchedUpdates</code>，包一下也可以批量更新。
react18中直接处理了 全都是批量更新。
具体来说
- react引发的事件处理 （onclick等） 异步
- 绕过react通过绑定addeventlistner，或者settimeout等等 同步

## useContext为什么不会被挂到hook链表上
因为在初始化和更新时会有两套不同的函数执行，mount 和 update，但是useContext只有一套代码，都是readContext.所以不需要挂载链表上。
原理类似于观察者模式。Provider上的值发生变化，通知给context和consumer

## useContext 和 redux 的区别
- context
  - 适合做全局管理
  - 避免props传递的繁琐
  - 如果组件依赖了context， 只要局部更新，组件都会一起更新，加上memo也没用，因为memo依赖的是props
  - 一般情况下可以替代redux
- redux
  - 去中心化思想，适合一些编辑器的场景，可以多状态
  - 多个组件，或者多个页面共享状态
  - 维护起来麻烦，新增的属性需要一一去添加

## useLayoutEffect 和 useEffect区别
react的一次更新分为2个阶段：
- render阶段： 构建一个fiber树，也就是workInProgress树；构建过程中，从root开始深度优先遍历再回溯到root
  - beginWorker ：组件状态的计算，diff，render函数
  - completeWorker：收集effect依赖链表，收集被跳过的update对象
- commit阶段 ： 异步执行useEffect，同步执行useLayoutEffect。
**useEffect要在页面渲染完之后才执行，不会堵塞页面，后者则是在dom更新完成，还没有开始渲染前执行。**
整体流程上都是先在render阶段，生成effect，并将它们拼接成链表，存到fiber.updateQueue上，最终带到commit阶段被处理。他们彼此的区别只是最终的执行时机不同，一个异步一个同步，这使得useEffect不会阻塞渲染，而useLayoutEffect会阻塞渲染。

## useRef 和 useState区别
|      |                 useRef                  |               useState               |
| :--: | :-------------------------------------: | :----------------------------------: |
| use  | 用于对dom的引用，保存值，不会被重新渲染 | 保存和更新组件的状态，值会被重新渲染 |
| 更新 |                同步更新                 |               异步更新               |
| 返回 |     一个全局可以访问和修改的ref对象     |      当前状态和一个状态更新函数      |

### useRef
1. 在整个生命周期中保持不变
2. 更改ref.current 不会引起组件的渲染， 不会触发re-render
3. useref 只在组件更新的时候渲染一次
4. usestate更新，可以被useeffect监听到， useref不可以
5. useref更新的是副作用

参考：[useRef的值可以被useEffect监听吗](https://blog.csdn.net/sinat_28071063/article/details/129302701)

### [ref callback](https://juejin.cn/post/7187429820928622629)

ref 可以接受一个ref对象，也可以接受一个callback函数
> 这个callback函数只有一个 dom元素 创建dom元素 立即执行，销毁也会执行一次 传参dom为null 
> 仅在挂载和卸载时调用
可能引起一些不必要的调用，可以用useCallback包装

用途场景：
- 操作 DOM，比如在组件挂载的时候滚动或聚焦
- 在 React 获取 DOM 属性，比如宽度或滚动位置
- 在 React 控制的 DOM 元素上使用 Portal
- 将 DOM 元素提供给多个消费者

## react hook 的一些闭包问题 和陷阱
<https://juejin.cn/post/6844903982037467143>
<https://juejin.cn/post/6844904193044512782>

## 父组件和子组件的useEffect哪个先更新 
dom render 和 主线程 先执行
useEffect 在之后执行
**dom render** 永远在useEffect之前执行

- 父子组件 ：
  - dom render 和 主线程： 父 -> 子
  - useEffect 在之后执行 ： 子 -> 父
- 兄弟组件：
  - 按前后顺序，递归
  - 如果组件含有子组件，则先执行完子组件 ，在走兄弟
[react中函数组件 - 父子组件的执行顺序](https://juejin.cn/post/7169526604199100447)

1. 渲染顺序：React 的渲染过程是从父组件开始的，这是因为父组件通常包含子组件的引用。因此，父组件需要首先渲染以确定子组件应该如何渲染。
2. 副作用和生命周期方法的执行：在所有组件都渲染完成后，React 会开始执行副作用和生命周期方法。这个过程是从最底层的子组件开始的，然后逐级向上。这样做的原因是，子组件通常是父组件逻辑的一部分，父组件的副作用可能依赖于子组件的状态或 DOM 元素。

> 父子组件的return（destroy）事件呢
[demo](https://codesandbox.io/s/uselayouteffect-zhixingshunxu-h5gg2?file=/src/App.js)
1. 在删除dom时，从父->子 删除  destroy
2. 在更新dom时，或者更新和删除都有时， 子->父 destroy
**无论什么情况**都是先执行所有的destroy 在执行create方法
## useEffect return的意义
每次重新渲染，都会让原组件（包括子组件）销毁，在重新诞生。
- 首次渲染，不会执行return
- 再次渲染，会先执行return ，在执行外面的
- return的回调，可以用来清除一些定时器

## react类组件的生命周期 现在有哪些
前文提到的引入了fiber+优先级制度，导致原先一些生命周期会混乱
- componentWillMounte/Update/RecieveProps 在16,17被取消
- 引入getDerivedStateFromProps(nextProps, nextState) 替代了上三者，render前调用，挂载更新后也会调用，**如果state的值在任何时候都依赖于props时才使用此方法**。
  - 让组件在props更新后更新state；
  - 返回一个对象，为null不更新。
- shouldComponentUpdate： 如果返回false，则不执行componentDidUpdate，也不会执行render；返回值默认是true。
- render
- getSnapshotBeforeUpdate在render之后，componentDidUpdate之前，
- componentDidUpdate
更新阶段的生命周期：
> static getDerivedStateFromProps()
shouldComponentUpdate()
render()
getSnapshotBeforeUpdate()
componentDidUpdate()

## React的错误边界
过去，组件内的代码异常会导致整个react内部状态破坏，一个组件在渲染期间发生错误，会卸载整个组件。因此引入了错误边界。

> 只能在类组件中使用，不能在hook中使用。因为需要用this.setState的回调传递callback，useState无法传入回调。
> 以下四种场景无法捕获错误边界
> > - 事件处理函数，需要用原生的trycatch
> > - setTimeout等异步代码
> > - 服务端渲染
> > - 他自己本身的错误
{% note primary %}
错误边界是一种组件， 他可以记录下渲染的错误，然后展示降级的ui。
{% endnote%}
实现错误边界的方法 依据两个生命周期
{% note primary %}**getDerivedStateFromError**和**componentDidCatch**{% endnote %}
- getDerivedStateFromError: 静态方法，给个机会去渲染降级ui
- componetDidCatch： 组件实例方法，当错误发生后，记录错误
当一个类组件定义了以上二者其一的时候，就会被定性为错误边界。
**一般来说，会包在最顶层的路由组件**、具体的粒度有自己决定
需要注意的是，从react16开始，没有被错误边界捕获的错误会导致整个react组件树被卸载。
因为 有的时候，留下一个错误的ui 比 完全移除他 更糟糕。

#### react的fiber vue为什么不需要fiber
react的fiber机制：是由于react内状态不可修改，所以需要自顶向下的去渲染树。本身会在内部生成一颗巨大的虚拟dom树，给第二步的diff带来了很大的性能消耗。而js占据主线程，渲染线程就无法工作。所以出现了react的fiber。
fiber 是一种纤程。他通过requestIdleCallback 去控制组件渲染。
vue的渲染机制：vue2使用的Object.defineProperty或者vue3中的proxy对数据做一个劫持。vue能准确的知道视图模版中哪一块需要更新。他本身可以实现精准更新，精确到当前组件的最小粒度。一方面：是给每个组件配置了{% label primary @监听器 %}，管理视图的依赖收集和数据更新。一方面：他的**模版语法，可以实现静态编译**，react的jsx语法是做不到的。这两者对性能也是有消耗的。但是他就不需要fiber去控制组件渲染，让出浏览器线程这类操作了。

## diff算法
Diff算法通过key和tag来对节点进行取舍，可直接将复杂的比对拦截掉，然后降级成节点的移动和增删这样比较简单的操作。对oldFiber和新的ReactElement节点的比对，将会生成新的fiber节点，同时标记上effectTag，这些fiber会被连到workInProgress树中，作为新的WIP节点。树的结构因此被一点点地确定，而新的workInProgress节点也基本定型。这意味着，在diff过后，workInProgress节点的beginWork节点就完成了。接下来会进入completeWork阶段
