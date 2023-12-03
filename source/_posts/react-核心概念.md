---
title: react 核心概念
date: 2023-11-02 17:16:48
tags:
---

# React的核心概念
----
基于两点介绍
1. Fiber架构和schedule调度
2. 优先级机制

## Fiber是什么
Fiber是什么？它是React的最小工作单元，在React的世界中，一切都可以是组件。在普通的HTML页面上，人为地将多个DOM元素整合在一起可以组成一个组件，HTML标签可以是组件（HostComponent），普通的文本节点也可以是组件（HostText）。
**每一个组件都可以对应一个fiber节点**，互相嵌套，组成fiber树。

```js
var fiber = {
  alternate,
  child,
  elementType: () => {},
  memoizedProps: null,
  memoizedState: null, // 在函数组件中，memoizedState用于保存hook链表
  pendingProps: {},
  return,
  sibling,
  stateNode,
  tag, // fiber的节点类型，初次渲染时，函数组件对应的tag为2，后续更新过程中对应的tag为0
  type: () => {}
  updateQueue: null,
}
```
### Fiber架构下的react是如何更新的
react的一次更新分为两个阶段
- render阶段
- commit阶段

##### render阶段 
> render阶段是 scheduler调度+ reconcile 就是把虚拟dom变成fiber
render阶段是在内部构建一颗新的fiber树，一般成为workInProgress树，构建过程依据现有的fiber树（current树），从root开始深度遍历再回溯到root。创建effect链表

每个fiber会经历两个阶段：
- beginWorker: 组件状态的计算，diff的操作，render函数的执行
- completeWorker: 被跳过的优先级update收集， effect链表的收集

**构建work树过程会有一个指针记录构建到哪个fiber节点，以便更新任务可恢复**

##### beginWorker
它的职能主要是**节点更新的入口，不会直接更新**，就是拦截无需更新的节点。

> 处理当前遍历的fiber，返回它的子fiber，构建workInProgress树。
核心更新的过程在于 **计算状态**  和 **diff算法**

##### 如何区分初始化和更新
**判断是否存在current**
初始化不会有current，更新时已经存在current树
根据节点是否是首次渲染 生成fiber 或者diff fiber

##### commit阶段
> 把fiber变成dom
- 不可中断
- 根据收集到的变化节点更新dom，异步执行useeffect 同步执行uselayouteffect

> 这两个任务都是独立的React任务，都会被schedule调度。
> render阶段的优先级是根据本次的更新的优先级决定的，高优先级可以打断低优先级
> commit阶段的优先级不可打断，按最高优先级来执行

#### schedule调度
根据优先级进行调度，保证最高优先级执行
根据时间片对任务进行中止和恢复。

#### 优先级机制

**如何将优先级机制融入 react 运行时**

**1. 获取本次更新优先级**

调用 `requestUpdateLane` 函数获取本次更新任务的优先级

优先级选取判断顺序如下
```js
SyncLane  >>  TransitionLane  >>  UpdateLane  >>  EventLane
```

**2. 创建 Update 对象**

setState 生成 Update 对象，每个对象里有个 lanes 属性

```js
function createUpdate(eventTime, lane) {
  var update = {
    eventTime: eventTime,
    lane: lane,
    tag: UpdateState,
    payload: null,
    callback: null,
    next: null
  };
  return update;
}
```

**3. 关联优先级**

- 将优先级合并到当前 fiber 节点的 lanes 属性中
- 将优先级合并到父级节点的 childLanes 中 (告诉父节点他的子节点有多少条赛道要跑)

- 将调度任务优先级合并到当前 react 应用根节点上
- 计算当前任务优先级赛道占用的开始时间

commit 阶段结束之后会释放占用的优先级

**4. 发起调度**

- 高优先级任务打断低优先级任务
  
  每一次调度都会对正在进行任务和当前任务最高优先级做比较，如果不相等，就代表有高优先级任务进来，需要打断当前正在的任务。

- 低优先级任务重启
  
  协调 (reconcile) 的下一个阶段是渲染 (renderer)，也就是我们说的 commit 阶段，在此阶段末尾，会调用 ensureRootIsScheduled 发起一次新的调度，执行尚未完成的低优先级任务

- 饥饿任务问题
  
  高优先级任务执行完毕则重启低优先级任务，如果持续有高优先级任务插入，如何处理？

  每次调度的开始，都会先检查下有没有过期任务，如果有的话，下一次就会以同步优先级进行 render 任务(reconcile)，同步优先级就是最高的优先级，不会被打断


交互就会生成更新，不同的更新，优先级不一样。
react中人为的进行了划分，最终目的是调度轻重缓急，因此产生了一套事件调度的优先级机制。
- 事件优先级： 按照用户的交互产生的
- 更新优先级： 事件导致的react产生的update对象的优先级（update.lane)
- 任务优先级： 产生update对象后，react去执行的优先级
- 调度优先级： scheduler根据react的任务生的调度任务的优先级

---

## 事件优先级
1. 离散事件 click，keydown等不连续的 优先级0
2. 用户阻塞事件 scroll drag mouseover 连续，阻塞渲染 优先级1
3. 连续事件 canplay error audio标签的canplay等等，优先2 最高

### 派发事件优先级
事件优先级在注册阶段确定，在root上注册时，会根据不同事件发不同的监听，最后绑定到root
然后借助scheduler中的runWithPriorty函数实现执行事件处理函数。

## 更新优先级
以setState为例，事件的执行会导致setState执行，而setState本质上是调用enqueueSetState，生成一个update对象，这时候会计算它的更新优先级，即update.lane

```js
const classComponentUpdater = {
  enqueueSetState(inst, payload, callback) {
    ...

    // 依据事件优先级创建update的优先级
    const lane = requestUpdateLane(fiber, suspenseConfig);

    const update = createUpdate(eventTime, lane, suspenseConfig);
    update.payload = payload;
    enqueueUpdate(fiber, update);

    // 开始调度
    scheduleUpdateOnFiber(fiber, lane, eventTime);
    ...
  },
};
```
**requestUpdateLane**:找出scheduler中记录的优先级，计算更新：lane.

## 任务优先级
假设产生一前一后两个update，它们持有各自的更新优先级，也会被各自的更新任务执行。经过优先级计算，如果后者的任务优先级高于前者的任务优先级，那么会让Scheduler取消前者的任务调度；如果后者的任务优先级等于前者的任务优先级，后者不会导致前者被取消，而是会复用前者的更新任务，将两个同等优先级的更新收敛到一次任务中；如果后者的任务优先级低于前者的任务优先级，同样不会导致前者的任务被取消，而是在前者更新完成后，再次用Scheduler对后者发起一次任务调度。

**保证高优先级任务及时响应，收敛同等优先级的任务调度。**

> 如果已经存在一个更新任务， 获得新任务的更新优先级之后，会进行比较，判断是否需要重新发起调度，如果需要，则计算调度优先级

## 调度优先级
调度优先级由任务优先级计算得出，在ensureRootIsScheduled更新真正让Scheduler发起调度的时候，会去计算调度优先级。

在Scheduler中，分别用过期任务队列和未过期任务的队列去管理它内部的task，过期任务的队列中的task根据过期时间去排序，最早过期的排在前面，便于被最先处理。而过期时间是由调度优先级计算的出的，不同的调度优先级对应的过期时间不同。

#### 小总结
这四种优先级 是**递进**的关系
事件优先级由事件本身决定，更新优先级由事件计算得出，然后放到root.pendingLanes，任务优先级来自root.pendingLanes中最紧急的那些lanes对应的优先级，调度优先级根据任务优先级获取。几种优先级环环相扣，保证了高优任务的优先执行。

#### 参考
- [react优先级机制](https://segmentfault.com/a/1190000038947307)
- [Concurrent模式下React的更新行为- 优先级模型](https://segmentfault.com/a/1190000039131960)
- [react的调度机制原理](https://segmentfault.com/a/1190000039101758)
---
#### 双缓冲机制

开始更新后，会有两棵树，一颗是workInProgress树，一棵是现有的current树，是当前页面显示的树。在更新未完成的时候，所有更新会在workInProgress树上，页面始终展示current，更新结束之后，两者替换。