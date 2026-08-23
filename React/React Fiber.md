React Fiber 是 React 16 之后重写的核心渲染架构，它把原来“递归同步渲染”的过程，改造成了“可中断、可恢复、可调度、可分优先级”的异步渲染模型。

#### 一、为什么React需要Fiber？
在 Fiber 之前，React 的更新过程大致是：状态更新 -> 生成新的 Viritual DOM -> 递归 Diff -> 找出变化 -> 一次性同步更新 DOM。
早期 React 使用的是递归遍历 Virtual DOM 树。

问题是：**一旦开始渲染，就不能中断。** 假设一个页面组件树很大，某次更新要遍历几千个组件，浏览器主线程会一直被 React 占用。
结果就是：React 正在计算更新、浏览器无法响应点击、浏览器无法执行动画、浏览器无法处理输入、页面卡顿。

所以 React 需要一种机制：
大的渲染任务拆成很多小任务，每执行一小段就看看浏览器是否有更高优先级任务；如果有就暂停，等空闲时继续。
这就是 Fiber 的核心价值。

#### 二、Fiber 解决了什么问题？

Fiber 主要解决四类问题：
1. 任务可拆分
2. 渲染可中断
3. 更新有优先级
4. 渲染和提交分离

> Fiber 的本质是 React 对组件树的一种新的数据结构和调度机制。它把组件更新过程拆成一个个 Fiber 节点，使 React 可以暂停、恢复、丢弃或复用渲染任务，从而支持异步渲染、优先级调度和并发特性。

#### 三、Fiber 是什么？

Fiber 有两层含义：
1. Fiber 是一种架构，也就是 React16 之后的新协调架构
2. Fiber 是一种数据结构，每一个 React Element 对应一个 Fiber 节点。

例如：
``` javaScript
function App() {
  return (
    <div>
      <h1>Hello</h1>
      <Button />
    </div>
  );
}
```

React 内部会构建类似这样的 Fiber 树：
```
App Fiber
  ↓ child
div Fiber
  ↓ child
h1 Fiber
  ↓ sibling
Button Fiber
```

每个 Fiber 节点大致包含这些信息：

```TypeScript
type Fiber = {
  type: any;              // 组件类型，例如 div、App、Button
  key: null | string;     // key，用于 diff
  stateNode: any;         // 对应真实 DOM 或组件实例

  child: Fiber | null;    // 第一个子节点
  sibling: Fiber | null;  // 下一个兄弟节点
  return: Fiber | null;   // 父节点

  pendingProps: any;      // 新 props
  memoizedProps: any;     // 上一次 props
  memoizedState: any;     // 上一次 state

  alternate: Fiber | null;// 双缓存对应节点
  flags: number;          // 标记增删改等副作用
}
```

其中 child（子节点）、sibling（兄弟节点）、return（父节点）这三个指针非常重要。

因为 React 需要手动控制遍历过程，不用普通递归树结构，递归调用一旦进入就很难中断，而 Fiber 使用链表结构后，React 可以自己控制：**当前处理到哪个 Fiber，下一个 Fiber 是谁，能不能暂停，暂停后从哪里恢复。**

#### 四、Fiber 和 Virtual DOM 是什么关系？🌟

**React Element / Virtual DOM**

JSX：
```JavaScript
<div className="box">hello</div>
```
会变成 React Element：
```JavaScript
{
  type: 'div',
  props: {
    className: 'box',
    children: 'hello'
  }
}
```

React Element 是一种轻量描述对象，也就是我们常说的 Virtual DOM。

Fiber 是 React 基于 React Element 创建出来的工作单元。
> React Element：描述 UI 长什么样
> Fiber：描述这个 UI 节点如何被更新、调度、提交

Interview response：
>Virtual DOM 是 UI 的描述对象，而 Fiber 是 React 内部用于调度和更新的工作单元。
>React 会根据 React Element 创建 Fiber 节点，并通过 Fiber 树完成 diff、调度、更新和提交。

#### 五、React Fiber 更新流程 🌟🌟

**1.  Reconciliation/Render Phase（协调阶段）：可中断**

Render 阶段主要做这些事：
```
1. 根据更新生成新的 Fiber 树
2. 对比新旧 Fiber
3. 找出需要更新的地方
4. 给 Fiber 打上 flags 标记
```
这个阶段是可以被中断的。
例如：
```
React 正在渲染一个低优先级列表更新
用户突然输入内容
React 暂停列表更新
优先处理输入更新
之后再继续或重新开始列表更新
```

Render 阶段不会真正操作 DOM，它只是计算：哪里需要新增、哪里需要删除、哪里需要更新属性、哪里需要执行 effect。

**2. Commit Phase（提交阶段）：不可中断**

Commit 阶段负责真正把变化提交到宿主环境，比如浏览器 DOM。
它会做：
```
1. 执行 DOM 插入、更新、删除
2. 执行 useLayoutEffect
3. 调度 useEffect
4. 更新 ref
```

Commit 阶段必须同步完成，不可中断。原因是：
>DOM 一旦开始改，就必须保持一致性。如果中途暂停，用户可能看到一半新 UI、一半旧 UI，页面会处于不一致状态。

所以：
Render 阶段：可中断、可调度
Commit 阶段：不可中断、同步提交

#### 六、Fiber 的工作循环：beginWork 和 completeWork

Render 阶段内部又可以分为两个过程：`beginWork`、`completeWork`，可以理解为一次深度优先遍历。

**1. beginWork：向下遍历**
beginWork 负责处理当前 Fiber，并生成子 Fiber。
例如：
```
App
 ↓
div
 ↓
Header
```
它主要做：
```
1. 根据组件类型计算子节点
2. 对比新旧 children
3. 创建或复用子 Fiber
4. 标记更新
```
对于函数组件，beginWork 会执行函数组件：
```JavaScript
function App() {
  return <div>Hello</div>;
}
```
React 会在 beginWork 阶段调用 App，拿到新的 React Element。

**2. completeWork：向上归并**
当一个节点的子节点都处理完之后，会进入 completeWork。
completeWork 主要做：
```
1. 创建真实 DOM 节点
2. 收集子节点的副作用 flags
3. 构建 effect 链表或副作用标记
4. 完成当前 Fiber
```
整体流程类似：
```
beginWork(App)
  beginWork(div)
    beginWork(Header)
    completeWork(Header)
  completeWork(div)
completeWork(App)
```

Summary：
>Render 阶段是一个基于 Fiber 树的深度优先遍历过程。向下时执行 beginWork，用于计算子 Fiber；向上时执行 completeWork，用于创建 DOM、收集副作用并完成节点。

#### 七、Fiber 的“双缓存”机制

Fiber 架构中有两棵树：
```
current Fiber 树
workInProgress Fiber 树
```

**current Fiber 树** ，代表当前屏幕上已经显示的 UI。
current tree = 当前页面

**workInProgress Fiber 树**，代表正在内存中计算的新 UI。
workIn Progress tree =。正在准备的新页面

更新过程：
```
当前页面：current Fiber tree
        ↓
发生 setState
        ↓
基于 current 创建 workInProgress
        ↓
在内存中完成 diff 和计算
        ↓
commit 阶段提交 DOM
        ↓
workInProgress 变成新的 current
```
这个机制类似前端图形渲染里的双缓冲：
```
旧画面继续显示
新画面在内存中准备
准备好后一次性切换
```
好处是：
```
1. 避免 UI 中间态暴露给用户
2. 支持中断和恢复
3. 支持并发渲染
```

Summary：
>Fiber 使用 current 和 workInProgress 两颗树实现双缓存。current 对应当前屏幕内容，workInProgress 用于计算下一次更新。更新完成后，workInProgress 会切换为新的 current，从而保证渲染过程不会直接污染当前 UI。

#### 八、Fiber 如何实现可中断？

Fiber 的可中断依赖两个核心点：
```
1. 把递归调用改造成链表式 Fiber 遍历
2. 把渲染过程拆成一个个工作单元
```

每个 Fiber 节点都可以看作一个工作单元。
React 每处理一个 Fiber，就判断一下：
```
当前时间片是否用完？
有没有更高优先级任务？
浏览器是否需要响应用户输入？
```

如果需要让出主线程，React 就暂停。伪代码可以理解为：
```JavaScript
function workLoop() {
  while (nextUnitOfWork && shouldYield() === false) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }
  
  if (nextUnitOfWork) {
    // 还有任务没完成，下次继续
    scheduleCallback(workLoop);
  } else {
    // render 阶段完成，进入 commit
    commitRoot();
  }
}
```
这里的关键是 `nextUnitOfWork`，它记录当前执行到哪个 Fiber 节点。所以 React 可以暂停后继续执行。

#### 九、Fiber 与优先级调度

Fiber 不只是能中断，还能区分任务优先级。
比如：
```
用户输入        高优先级
点击按钮        高优先级
动画更新        较高优先级
接口返回渲染列表 低优先级
隐藏区域更新    更低优先级
```
在 React 18 中，这套优先级模型主要通过 Lane 模型实现。可以简单理解为：
```
Lane = 更新所在的车道
不同车道代表不同优先级
```

例如：
```
SyncLane              同步更新
InputContinuousLane   连续输入更新
DefaultLane           默认更新
TransitionLane        过渡更新
IdleLane              空闲更新
```

核心思想：
>React 会为不同更新分配不同优先级，高优先级任务可以打断低优先级任务。Fiber 架构提供了可中断的工作单元，Scheduler 和 Lane 机制负责决定先执行哪个任务。

#### 十、Fiber 与 React 18 并发渲染

React 18 的 Concurrent Rendering 依赖 Fiber。
常见 API：
```JavaScript
startTransition(() => {
  setList(keyword);
});
```
或者：
```JavaScript
const deferredValue = useDeferredValue(value);
```

这些能力背后都依赖 Fiber 的可中断渲染。
例如搜索框场景：
```JavaScript
function SearchPage() {
  const [keyword, setKeyword] = useState('');
  const [list, setList] = useState([]);

  function handleChange(e) {
    const value = e.target.value;

    setKeyword(value);

    startTransition(() => {
      setList(filterLargeList(value));
    });
  }

  return (
    <>
      <input value={keyword} onChange={handleChange} />
      <List data={list} />
    </>
  );
}
```
这里：
```
setKeyword：高优先级，保证输入框不卡
setList：低优先级，可以延后渲染
```

Fiber 允许 React 在渲染大列表时被输入打断。如果没有 Fiber，这种大列表渲染可能会阻塞输入。

#### 十一、Fiber 和 Diff 的关系

React Diff 仍然遵循几个基本策略：
```
1. 不同类型的元素，直接销毁重建
2. 同类型元素，复用节点并更新 props
3. 列表通过 key 判断节点是否可复用
```

Fiber 架构并没有推翻 Diff，而是改变了 Diff 的执行方式。
以前，递归同步 Diff；现在，基于 Fiber 节点的可中断 Diff。

Summary：
>Fiber 并不是替代 Virtual DOM 或 Diff，而是让 Diff 过程可以被拆分、暂停和调度。
>React 仍然会根据 type 和 key 判断节点是否复用，只是这个过程现在发生在 Fiber 树的构建和协调过程中。

#### 十二、Fiber 和 Hook 的关系

Hooks 的状态也是挂在 Fiber 节点上的。
例如：
```JavaScript
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Tom');

  return <div>{count} - {name}</div>;
}
```

对于函数组件 Fiber，React 内部会维护一个 Hooks 链表。大致类似：
```
Counter Fiber
  memoizedState
      ↓
    Hook(useState count)
      ↓
    Hook(useState name)
      ↓
    Hook(...)
```

这也是为什么 Hooks 不能写在条件语句里，错误写法：
```JavaScript
function Demo({ visible }) {
  if (visible) {
    const [count, setCount] = useState(0);
  }

  const [name, setName] = useState('Tom');
}
```

因为 React 是按调用顺序关联 Hook 状态的。如果某次渲染 Hook 顺序变了，React 就无法正确知道哪个 state 对应 哪个 Hook。
所以：
```
Hooks 状态依赖 Fiber
Hooks 顺序依赖组件渲染顺序
```

Summary：
> 函数组件的 Hooks 状态保存在对应 Fiber 节点的 `memoizedState` 上，多个 Hook 以链表形式连接。React 通过 Hook 的调用顺序来匹配状态，所以 Hook 不能放在条件、循环或嵌套函数中。

#### 十三、Fiber 和 useEffect / useLayoutEffect

Fiber 的 Commit 阶段会处理副作用。
React 中常见的副作用包括：
```
DOM 更新
ref 更新
useLayoutEffect
useEffect
```
它们的执行时机不同。

##### useLayoutEffect
```
DOM 更新完成之后
浏览器绘制之前
同步执行
```
适合：
```
读取 DOM 布局
同步修改 DOM
避免闪烁
```

##### useEffect
```
浏览器绘制之后
异步执行
```
适合：
```
请求数据
订阅事件
日志上报
非阻塞副作用
```

Fiber 会在 Render 阶段收集这些 effect，在 Commit 阶段统一执行。

Summary：
> useEffect 和 useLayoutEffect 都会在 Render 阶段被记录到 Fiber 的副作用链上，但执行发生在 Commit 阶段。useLayoutEffect 在 DOM 更新后、浏览器绘制前同步执行；useEffect 通常在绘制后异步执行。

#### 十四、一次 setState 后 Fiber 做了什么？

具体流程：
```
1. 调用 setState / dispatch
2. React 创建 Update 对象
3. Update 被加入对应 Fiber 的 updateQueue
4. React 根据更新类型分配优先级 Lane
5. 从当前 Fiber 向上找到 Root
6. 调度更新任务
7. 进入 Render 阶段，构建 workInProgress Fiber 树
8. 执行 beginWork / completeWork
9. 找出需要变更的 Fiber，并打 flags
10. Render 完成后进入 Commit 阶段
11. 执行 DOM 更新、ref、layout effect、passive effect
12. workInProgress 树切换为 current 树
```

简化版：
```
setState
  ↓
创建更新
  ↓
调度任务
  ↓
Render 阶段计算变化
  ↓
Commit 阶段更新 DOM
  ↓
页面完成更新
```

#### 十五、面试高频问题与回答思路

##### 问题 1：什么是 React Fiber？

>Fiber 是 React 16 引入的新协调架构。它既是一种架构，也是一种数据结构。每个 Fiber 节点对应一个组件或 DOM 节点，React 通过 Fiber 树把渲染工作拆成多个可调度的工作单元，从而支持任务中断、恢复、优先级调度和并发渲染。

##### 问题 2：Fiber 解决了什么问题？

> Fiber 主要解决旧版 React 递归同步渲染不可中断的问题。旧架构中，大组件树更新会长时间占用主线程，导致输入、动画和点击卡顿。Fiber 把渲染拆成多个小任务，使 React 可以在合适的时候暂停低优先级任务，优先响应用户交互。

##### 问题3：Fiber 为什么能中断？

> 因为 Fiber 把组件树从递归结构改造成了链表结构，每个 Fiber 节点都是一个工作单元，并通过 child、sibling、return 指针连接。React 可以记录当前执行到哪个 Fiber，通过 nextUnitOfWork 暂停和恢复任务，而不是依赖不可控的递归调用栈。

##### 问题4：Render 阶段和 Commit 阶段有什么区别？

> Render 阶段负责构建 workInProgress Fiber 树、执行 diff、计算变化并标记副作用，这个阶段可以被中断。Commit 阶段负责把变化真正提交到 DOM，并执行 ref、useLayoutEffect、useEffect 等副作用，这个阶段不可中断，必须同步完成，以保证 UI 一致性。

##### 问题 5：Fiber 和 Virtual DOM 是什么关系？

> Virtual DOM，也就是 React Element，是 UI 的描述对象；Fiber 是 React 内部基于 React Element 创建的工作单元。React Element 描述 “要渲染什么”，Fiber 描述“这个节点如何被更新、调度和提交”。

##### 问题 6：Fiber 的双缓存是什么？

>Fiber 中有 current 树和 workInProgress 树。current 树表示当前屏幕上的 UI，workInProgress 树表示正在计算的新 UI。React 在内存中完成 workInProgress 的构建和 diff，提交后再切换成新的 current，从而避免更新过程暴露中间状态。

##### 问题 7：Fiber 和 React 18 Concurrent Mode 有什么关系？

>React 18 的并发渲染依赖 Fiber 的可中断能力。比如 startTransition 可以把某些更新标记为低优先级，React 在渲染这些更新时，如果有用户输入更高优先级任务，可以暂停当前渲染，优先处理高优先级更新，从而提升交互流畅度。


##### 问题 8：为什么 Commit 阶段不能中断？

> 因为 Commit 阶段会真实修改 DOM。如果中断，页面可能处于一部分旧 UI、一部分新 UI 的不一致状态。为了保证用户看到的 UI 是完整一致的，Commit 阶段必须同步执行完成。

##### 问题 9：Fiber 和 Hooks 有什么关系？

> 函数组件的 Hooks 状态保存在对应 Fiber 节点的 memoizedState 上，多个 Hook 以链表形式连接。React 按 Hook 调用顺序来匹配状态，所以Hooks 不能写在条件、循环或嵌套函数中。
> 
> 核心联系与机制：
> 数据持久化（Fiber 存储）：每个函数组件对应一个 Fiber 节点。在该节点上的 memoizedState 属性，存储着该组件所有 Hooks 组成的单向链表。
> 
> 状态管理：当函数组件执行时，`useXXX` 会在该组件的 Fiber 上按顺序读取或更新链表中的 Hooks 节点。
> 
> 调用顺序约束：基于单向链表结构（`hook.next`)，React 要求 Hooks 必须在顶层按序调用，这决定了不能在条件语句或循环中调用它们。
> 
> 副作用管理：`useEffect` 或 `useLayoutEffect` 创建的副作用会被封装并记录在 Fiber 的 `updateQueue` 或 `memoizedState` 中，伴随 Fiber 调度执行。

##### 问题 10：React Fiber 中 key 的作用是什么？

>key 主要用于列表 diff，帮助 React 判断新旧 children 中哪些 Fiber 可以复用。稳定的 key 可以减少不必要的卸载和重建；不稳定的 key，比如数组 index，在列表插入、删除、排序时可能导致状态错乱和性能问题。

#### 十七、最重要的记忆点

```
Fiber = 新协调架构 + 工作单元数据结构

核心目的：解决同步递归渲染不可中断的问题

核心能力：可中断、可恢复、可丢弃、可复用、可调度、有优先级

核心流程：Render 阶段可中断；Commit 阶段不可中断

核心结构：child、sibling、return、alternate

核心机制：
current / workInProgress 双缓存
beginWork / completeWork
flags 副作用标记
Lane 优先级模型
```
