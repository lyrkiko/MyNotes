#### 一、React 为什么需要 Diff？

React 的更新流程可以简化为：
```
state / props 变化
  ↓
组件重新执行 render
  ↓
生成新的 React Element 树
  ↓
和旧树进行 Diff
  ↓
找出变化
  ↓
更新真实 DOM
```

例如：
```JavaScript
function App() {
  const [count, setCount] = useState(0);

  return <div>{count}</div>;
}
```

当 `count` 从 `0` 变成 `1` 时，React 不会直接全量销毁整个页面再重建，而是会比较：
```
旧 UI：<div>0</div>
新 UI：<div>1</div>
```

然后发现：
```
div 类型没变
只需要把文本从 0 改成 1
```
这就是 Diff 的价值。

#### 二、传统树 Diff 为什么不能直接用？

如果严格比较两棵树的最小差异，复杂度通常很高。对于一棵有 n 个节点的树，传统树 Diff 的复杂度可能达到：O(n^3)，这对前端页面来说太慢了。

React 采用了一套基于经验的策略，把复杂度降低到接近：O(n)。
核心前提是 React 做了几个假设：
```
1. 不同类型的元素，通常会生成不同的树
2. 同一层级的子节点，可以通过 key 判断是否可复用
3. 跨层级移动的节点情况较少，不做复杂跨层级比较
```
这几个假设就是 React Diff 的核心思想。

#### 三、React Diff 的三个核心策略 🌟🌟

1. 不同类型直接替换
2. 同类型复用并更新属性
3. 同层列表通过 key 复用节点

#### 四、策略一：不同类型元素，直接销毁重建

如果新旧节点类型不同，React 会认为两棵子树完全不同。
例如：
```JavaScript
// old
<div>
  <Counter />
</div>

// new
<section>
  <Counter />
</section>
```

旧节点是：`div`  新节点是：`section` ，类型不同，React 会直接：
```
卸载旧 div 子树
创建新 section 子树
```
即使里面都有 `<Counter />`，React 也不会尝试跨类型深度复用。
这意味着：旧 Counter 会被卸载，新 Counter 会重新挂载，组件状态会丢失。

>React Diff 中，如果新旧交替节点 type 不同，React 会直接卸载旧节点及其子树，重新创建新节点及其子树，因为 React 认为不同类型通常代表不同结构。

#### 五、策略二：同类型 DOM 节点，复用 DOM，只更新属性

如果新旧节点类型相同，例如都是 `div`：
```JavaScript
// old
<div className="old" title="hello" />

// new
<div className="new" title="hello" />
```

React 会复用原来的 DOM 节点，只更新变化的属性：
```
className: old → new  
title: 不变，不处理
```

真实 DOM 不会被销毁重建。

再比如：
```JavaScript
// old
<button disabled={false}>提交</button>

// new
<button disabled={true}>提交</button>
```
React 会复用 `button`，只更新 `disabled` 属性。

> 如果新旧元素 type 相同，React 会复用对应的 Fiber 和真实 DOM 节点，只比较 props 的变化，然后在 commit 阶段更新变化的属性。

#### 六、策略三：同类型组件，复用组件实例或 Fiber
``` JavaScript
// old
<UserCard name="Tom" />

// new
<UserCard name="Jerry" />
```

对于组件也是一样。因为组件类型都是 `UserCard`，React 会复用这个组件对应的 Fiber。

变化是：
```
props.name: Tom -> Jerry
```

所以 React 会让 `UserCard` 重新渲染，但不会把它当成一个全新的组件。

这意味着：
```
组件 state 会保留
Hooks 状态会保留
组件不会重新 mount
```

如果组件类型变了：
```JavaScript
// old
<UserCard />

// new
<ProductCard />
```

React 会认为这是两个完全不同的组件：
```
卸载 UserCard
挂载 ProductCard
旧组件状态丢失
```

> 对于组件节点，React 根据组件 type 判断是否复用。如果 type 相同，则复用 Fiber，更新 props，并继续比较子节点；如果 type 不同，则卸载旧组件，挂载新组件。

#### 七、React 只做同层比较

这是 Diff 的另一个核心点。React 不会做复杂的跨层级移动比较。

例如：
```JavaScript
// old
<div>
  <A />
</div>

// new
<section>
  <div>
    <A />
  </div>
</section>
```

虽然 `<A />` 还存在，但它的层级位置变了。
React 通常不会跨层级去寻找这个旧的 `<A />` 并复用，而是按层级比较：
```
old 根节点：div
new 根节点：section
类型不同
直接替换整棵子树
```
所以：
```
A 会被卸载后重新挂载
A 的状态会丢失
```

> React Diff 是同层比较，不做跨层级节点复用。这样可以降低算法复杂度，但代价是某些跨层级移动会被视为删除和新增。

#### 八、列表 Diff：key 是核心 🌟🌟🌟
```JavaScript
const list = ['A', 'B', 'C'];

return list.map(item => <div>{item}</div>);
```
如果没有 key，React 会按位置比较。
旧列表：A B C；新列表： A C B

React 只能按照 index 比较：
```
第 0 个：A vs A，复用
第 1 个：B vs C，更新
第 2 个：C vs B，更新
```

它不知道 `B` 和 `C` 只是交换了位置。所以 React 需要 key。

#### 九、有 key 时，React 如何复用？

```JavaScript
const list = [
  { id: 1, name: 'A' },
  { id: 2, name: 'B' },
  { id: 3, name: 'C' },
];

return list.map(item => (
  <div key={item.id}>{item.name}</div>
));
```

旧列表：
```
key=1 A
key=2 B
key=3 C
```

新列表：
```
key=1 A
key=3 C
key=2 B
```

React 通过 key 可以知道：
```
key=1 还在
key=2 还在，只是位置变了
key=3 还在，只是位置变了
```
因此 React 可以复用对应 Fiber 和 DOM，而不是错误地按位置更新。

> key 的作用是帮助 React 在同层 children 中识别哪些节点可以复用。React 会通过 key 和 type 一起判断节点身份，key 相同且 type 相同，才可以复用。

#### 十、为什么不推荐用 index 作为 key？🌟🌟🌟

如果列表永远不排序、不插入、不删除，用 index 问题不大。但是一旦列表会发生变化，index 作为 key 会导致问题。

例如：
``` JavaScript
// 旧列表
index 0: A
index 1: B
index 2: C

// 现在在头部插入一个 X：
index 0: X
index 1: A
index 2: B
index 3: C

list.map((item, index) => (
  <Item key={index} value={item} />
));
```

React 会认为：
```
key=0 的节点还在，只是内容从 A 变成 X
key=1 的节点还在，只是内容从 B 变成 A
key=2 的节点还在，只是内容从 C 变成 B
key=3 是新增
```

这会导致：
```
组件状态错位
输入框内容错乱
动画异常
性能变差
```

尤其是这种场景：
```JavaScript
function TodoItem({ text }) {
  const [checked, setChecked] = useState(false);

  return (
    <label>
      <input
        type="checkbox"
        checked={checked}
        onChange={() => setChecked(!checked)}
      />
      {text}
    </label>
  );
}
```

如果使用 index 作为 key，列表头部插入新数据后，原来某个 Todo 的 `checked` 状态可能会跑到另一个 Todo 上。

> 不推荐用 index 作为 key，因为当列表发生插入、删除或排序时，index 会变化，React 会错误复用节点，导致组件状态错位。key 应该使用稳定且唯一的业务 id。

#### 十一、key 是否只影响性能？

key 不只影响性能，还影响组件状态是否正确。

```JavaScript
<UserForm key={userId} userId={userId} />
```

当 `userId` 变化时，如果你希望表单状态完全重置，可以故意改变 key。
```
key 变化
  ↓
React 认为这是新组件
  ↓
卸载旧组件
  ↓
挂载新组件
  ↓
state 重置
```
所以，key 有两个作用
```
1. 帮助列表 Diff，提高复用准确性
2. 控制组件身份，决定状态是否保留
```

>  key 本质上用于表示同层节点的身份。它不仅影响 Diff 性能，也影响组件状态的保留或重置。key 改变时，React 会把它当成一个新节点，旧状态会丢失。

#### 十二、React Diff 的大致流程

一次更新中，React Diff 发生在 Render 阶段。可以简化为：
```
1. state / props 更新
2. React 重新执行函数组件
3. 生成新的 React Element
4. 基于 current Fiber 创建 workInProgress Fiber
5. 对比 old Fiber 和 new React Element
6. 判断 type 和 key 是否相同
7. 能复用则复用 Fiber，并更新 props
8. 不能复用则标记删除旧节点，创建新 Fiber
9. 对 children 继续 Diff
10. 给需要变更的 Fiber 打 flags
11. Render 阶段结束
12. Commit 阶段根据 flags 更新真实 DOM
```

> React Diff = 对比新旧 Fiber 和 新 React Element，生成新的 workInProgress Fiber，并标记需要提交的变更。

#### 十三、Fiber 架构下 Diff 发生在哪里？🌟🌟🌟

React 更新时会有两颗 Fiber 树：
```
current Fiber 树：当前页面对应的 Fiber 树
workInProgress Fiber 树：正在计算的新 Fiber 树
```

Diff 发生在构建 `workInProgress Fiber` 的过程中。更具体地说：
```
beginWork 阶段会根据新的 React Element 和旧 Fiber 进行 reconcileChildren 
```
也就是：
```
old Fiber children
       +
new React Element children
       ↓
reconcile
       ↓
new workInProgress children
```

如果可以复用：
```
旧 Fiber 通过 alternate 关联到新 Fiber
```

如果不能复用：
```
创建新 Fiber
旧 Fiber 标记 Deletion
新 Fiber 标记 Placement
```

> 在 Fiber 架构中，Diff 主要发生在 Render 阶段的 beginWork 过程中。React 会用当前 Fiber 的子节点和新的 React Element 子节点进行 reconcile，生成 workInProgress Fiber，并通过 flags 标记新增、删除和更新，最后在 Commit 阶段统一执行 DOM 操作。

#### 十四、React Diff 不是直接比较真实 DOM 🌟🌟🌟

React Diff 比较的不是浏览器真实 DOM，而是：旧 Fiber、新 React Element。或者更通俗地说：React 内部的虚拟结构。

流程是：
```
新 JSX
  ↓
新 React Element
  ↓
和旧 Fiber 比较
  ↓
生成 workInProgress Fiber
  ↓
commit 阶段更新真实 DOM
```

> React Diff 不会直接拿真实 DOM 做比较。它比较的是 React 内部的数据结构，也就是旧 Fiber 和新的 React Element。真实 DOM 只会在 Commit 阶段根据 Diff 结果被更新。

#### 十五、单节点 Diff 流程

```
同 key + 同type = 复用
同 key + 不同type = 删除旧节点，创建新节点
不同 key = 删除旧节点，创建新节点
```

> key 和 type 通常要同时匹配，React 才能认为是同一个节点。

#### 十六、多节点 Diff 流程，也就是列表 Diff

React 对列表 children 的 Diff 大致分两轮。

假设旧列表：
```
A B C D
```
新列表：
```
A B E C D
```

**第一轮：从左到右顺序比较**

React 会先按顺序比较：
```
A vs A，复用
B vs B，复用
C vs E，不匹配，停止第一轮
```
前面连续相同的部分可以快速处理。

**第二轮：把剩余旧节点放进 Map**

剩余旧节点：
```
C D
```

React 会建立一个 Map：
```
C -> old C  
D -> old D
```

然后遍历新列表剩余部分：
```
E：旧 Map 中没有，新增
C：旧 Map 中有，复用
D：旧 Map 中有，复用
```

最终：
```
E 标记 Placement
C 复用
D 复用
```

这就是为什么稳定 key 很重要。React 需要用 key 快速在旧 children 中找到可复用节点。

#### 十七、React 如何判断节点是否需要移动？

列表场景中，React 会维护一个类似 `lastPlacedIndex` 的变量。它的作用是：判断旧节点的位置是否小于已经处理过的最大位置。

如果一个节点的旧位置比 `lastPlacedIndex` 小，说明它需要移动。

例如旧列表：
```
A B C
```
新列表：
```
B A C
```
旧位置：
```
A: 0
B: 1
C: 2
```
新列表从左到右处理：
```
B：旧位置 1，lastPlacedIndex = 0，不需要移动，更新 lastPlacedIndex = 1
A：旧位置 0，小于 lastPlacedIndex 1，需要移动
C：旧位置 2，大于 lastPlacedIndex 1，不需要移动
```

> React 列表 Diff 会根据旧节点的 index 和遍历过程中记录的 lastPlacedIndex 判断节点是否需要移动。如果旧位置小于已经处理过的最大旧位置，说明这个节点在新列表中发生了**前移**，需要标记移动。

#### 十八、React Diff 的结果是什么？

Diff 的结果不是直接更新 DOM，而是给 Fiber 打标记。
常见标记包括：
```
Placement：新增或移动
Update：属性或文本更新
Deletion：删除
```

例如：
```JavaScript
// old
<div className="a">hello</div>

// new
<div className="b">hello</div>
```
结果可能是：
```
复用 div Fiber
标记 Update
commit 阶段更新 className
```

再比如：
``` JavaScript
// old
<div>A</div>

// new
<span>A</span>
```
结果是：
```
旧 div 标记 Deletion
新 span 标记 Placement
commit 阶段删除 div，插入 span
```

> Diff 的结果是生成新的 workInProgress Fiber 树，并在 Fiber 上打上 flags，比如 Placement、Update、Deletion。真正的 DOM 操作会在 Commit 阶段根据这些 flags 执行。

#### 十九、React Diff 和 shouldComponentUpdate / React.memo 的关系

Diff 不是每次都会深入到所有子节点。
React 可以通过一些方式跳过不必要的渲染：
```
class 组件：shouldComponentUpdate / PureComponent
函数组件：React.memo
Hooks：useMemo / useCallback 辅助稳定引用
```
例如：
```JavaScript
const Child = React.memo(function Child({ name }) {
  console.log('Child render');
  return <div>{name}</div>;
});
```
如果父组件更新，但传给 `Child` 的 props 没变，`React.memo` 可以让 `Child` 跳过重新渲染。

但注意：
```JavaScript
<Child user={{ name: 'Tom' }} />
```

每次父组件 render 都会创建新对象：
```
旧 user !== 新 user
```
所以 `React.memo` 可能失效。

> React.memo、PureComponent 和 shouldComponentUpdate 可以在一定条件下让 React 跳过子树的 render 和 Diff。它们不是替代 Diff，而是减少进入 Diff 的范围。

#### 二十、React Diff 和不可变数据的关系

不可变数据是指：不要直接修改原来的对象，而是创建一个新的对象来表示变化后的状态。

React 不会深度递归比较所有对象属性，因为成本太高。如果 React 每次都深比较，大型对象、列表、嵌套结构会非常慢。所以 React 判断 props 是否变化，很多时候依赖引用地址比较。

例如：
``` JavaScript
const oldUser = { name: 'Tom' };
const newUser = { name: 'Tom' };

oldUser === newUser; // false
```

即使内容一样，只要引用不同，React 就可能认为 props 变了。所以在 React 中通常推荐：

```
不要直接修改原对象
而是创建新对象
```

> React Diff 本身比较的是前后两次 render 生成的元素树，但 React 以及 React 生态中的很多性能优化，例如 React.memo、PureComponent、shouldComponentUpdate、useMemo，都依赖浅比较。浅比较只比较对象引用，不会深度比较对象内部属性。如果直接修改旧对象的属性，对象引用没有变化，React 很难判断这个数据发生了变化，可能导致组件不更新或 memo 失效。
> 
>  所以 React 更推荐使用不可变数据更新方式，也就是创建新对象或新数组。这样变化路径上的引用会发生变化，React 可以通过引用比较快速判断哪些数据变了，同时未变化的部分仍然可以复用旧引用，从而兼顾正确性和性能。

#### 二十一、React Diff 和状态保留

React 是否保留组件状态，主要看组件在树中的身份是否保持一致。
判断身份主要看：**同一层级、相同 type、相同 key**

> React 通过组件在树中的位置、type 和 key 判断组件身份。身份相同则状态保留，身份不同则卸载旧组件并创建新组件，状态会重置。

#### 二十二、常见面试题与回答思路

##### 1. React Diff 的核心思想是什么？

> React Diff 的核心思想是基于 UI 的特点做启发式比较。它默认不同类型的元素产生不同子树，只做同层级比较，并通过 key 在同层 children 中识别可复用节点，从而把复杂树 Diff 从高复杂度降低到接近 O(n)。

##### 2. React Diff 的复杂度为什么是 O(n)？

> React 放弃了通用树 Diff 的跨层级最小编辑比较，而采用三个假设：不同类型直接替换；只比较同层级节点；列表通过 key 识别复用。因此每个节点大致只需要遍历一次，所以复杂度接近 O(n)。

##### 3. key 的作用是什么？

> key 用于标识同层级 children 中节点的身份。React 会结合 key 和 type 判断旧 Fiber 是否可以复用。稳定 key 可以减少不必要的销毁和重建，也可以避免组件状态错位。

##### 4. 为什么不建议用 index 作为 key？

> 因为 index 不是稳定身份。当列表发生插入、删除或排序时，index 会变化，React 可能错误复用节点，导致组件状态错位，比如输入框内容、checkbox 状态或动画状态跑到其他列表项上。

##### 5. React Diff 会直接操作真实 DOM 吗？

> 不会。React Diff 发生在 Render 阶段，比较的是旧 Fiber 和新的 React Element，生成 workInProgress Fiber，并标记 flags。真实 DOM 操作发生在 Commit 阶段。

##### 6. type 相同和不同分别会怎样？

> type 相同，React 会尝试复用 Fiber 和 DOM，只更新 props，并继续 Diff 子节点。
> type 不相同，React 会卸载旧节点及其子树，创建新节点及其子树，旧组件状态会丢失。

##### 7. React 如何处理列表节点移动？

> React 会先顺序比较前面可复用的节点，遇到不匹配后把剩余旧节点放入 Map，然后用新列表的 key 去查找可复用的节点。它会根据旧 index 和 lastPlacedIndex 判断节点是否需要移动，并打上 Placement 标记。

##### 8. key改变会发生什么？

> key 改变后，React 会认为这是一个新的节点。旧节点会被卸载，新节点会重新挂载，因此组件内部 state 和 Hooks 状态都会重置。

##### 9. React.memo 和 Diff 是什么关系？

> React.memo 可以在 props 浅比较相等时跳过函数组件的重新渲染，从而减少后续子树 Diff。它是一种减少 Diff 范围的优化手段，不是替代 Diff。

##### 10. Diff 和 Fiber 的关系是什么？

> Diff 是协调过程的一部分。Fiber 是 React 内部的工作单元和树结构。React 在 Render 阶段通过 Fiber 架构执行 Diff，生成 workInProgress 树，并把变化记录为 flags，最后在 Commit 阶段提交到 DOM。


#### 二十三、一套完整面试回答模板

> React Diff 是 React 在更新时对比新旧 UI 树，找出需要变更部分的过程。React 并没有使用通用树 Diff 算法，因为复杂度太高，而是基于前端 UI 的特点做了启发式优化，使复杂度接近 O(n)。
> 
> 它的核心策略有三个：第一，不同类型的元素会直接销毁旧子树并创建新子树；第二，相同类型的 DOM 或组件会复用已有节点，只更新变化的 props，并继续比较子节点；第三，对于同层列表，React 通过 key 判断哪些节点可以复用。
> 
> 在 Fiber 架构中，Diff 发生在 Render 阶段，主要是在 beginWork 中通过 reconcileChildren 对比旧 Fiber 和新的 React Element，生成 workInProgress Fiber。Diff 的结果不是直接操作 DOM，而是在 Fiber 上打 flags，比如 Placement、Update、Deletion。等 Render 阶段完成后，React 会进入 Commit 阶段，根据这些 flags 真实更新 DOM。
> 
> key 在列表 Diff 中非常重要，它代表同层节点的稳定身份。使用稳定 key 可以帮助 React 正确复用节点，避免不必要的重建，也避免组件状态错位。不推荐用 index 作为 key，因为列表插入、删除或排序时，index 会变化，可能导致错误复用。
> 
> 总结来说，React Diff 的本质是用同层比较、type 判断和 key 复用这几个规则，在性能和准确性之间做平衡。


```
React Diff 核心关键词：

1. 启发式 Diff
2. O(n)
3. 同层比较
4. type 不同直接替换
5. type 相同复用节点
6. key 标识同层节点身份
7. 不推荐 index 作为 key
8. Fiber Render 阶段执行 Diff
9. Diff 结果是 flags
10. Commit 阶段更新真实 DOM
11. key 改变会导致组件重新挂载
12. React.memo 可以减少 Diff 范围
```



