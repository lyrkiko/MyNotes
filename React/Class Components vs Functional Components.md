
##### React 类组件与函数组件的定义与区别

###### **1. 类组件（Class Components）**

- **定义**：通过 ES6 类继承 `React.Component`，使用生命周期方法和状态管理。

- **特点**：
	-  使用 `this.state` 和 `this.setState()` 管理状态。
	-  通过生命周期方法（如 `componentDidMount`、`componentDidUpdate`）处理副作用。
	-  需要绑定事件处理函数的 `this` 上下文。
###### **2. 函数组件（Functional Components）**

- **定义**：普通 JavaScript 函数，通过返回 JSX 定义 UI。

- **特点**（借助 Hooks）：
    - 使用 `useState` 管理状态，`useEffect` 处理副作用。
    - 无 `this` 绑定问题，代码更简洁。
    - 逻辑复用通过自定义 Hooks 实现（替代 HOC 或 Render Props）。

### **核心区别**

| **特性**    | **类组件**                       | **函数组件**          |
| --------- | ----------------------------- | ----------------- |
| **状态管理**  | `this.state` + `setState`     | `useState` Hook   |
| **副作用处理** | 生命周期方法（`componentDidMount` 等） | `useEffect` Hook  |
| **代码复杂度** | 高（需绑定 `this`，样板代码多）           | 低（无 `this`，逻辑更集中） |
| **逻辑复用**  | HOC 或 Render Props            | 自定义 Hooks         |
| **未来兼容性** | 部分生命周期方法已弃用                   | 全面支持新特性（如并发模式）    |

###### 为什么推荐使用函数式组件？
React 团队自 **React 16.8（引入 Hooks）** 起推动函数组件成为主流，并在 React 19 中进一步强化。原因如下：

**1. 更简洁的代码**

- 函数组件避免类组件的样板代码（如 `constructor`、`this` 绑定）。
    
- **示例对比**：
    
    - 类组件需显式绑定 `this`：`this.handleClick = this.handleClick.bind(this)`
        
    - 函数组件直接声明函数：`const handleClick = () => {...}`

 **2. 更好的逻辑复用**

- **自定义 Hooks** 允许抽取状态逻辑（如 `useFetch`、`useAuth`），替代复杂的 HOC 或 Render Props。
    
- 类组件的复用模式易导致“嵌套地狱”（Wrapper Hell）。

 **3. 更优的性能优化**

- 函数组件支持 `React.memo` 避免无效渲染，配合 `useMemo`/`useCallback` 精细控制更新。
    
- 类组件的 `PureComponent` 和 `shouldComponentUpdate` 使用更繁琐。

**4. 对齐未来架构（并发模式）**

- React 18+ 的 **并发特性**（如 Suspense、流式渲染）在函数组件中更易实现。
    
- 类组件的生命周期方法与并发模式存在兼容性问题（如 `componentWillMount` 已弃用）。

 **5. 更友好的学习曲线**

- 函数组件只需掌握 JavaScript 函数和 Hooks，无需理解类、`this` 等 OOP 概念。
    
- Hooks 将相关逻辑聚合（如将 `useEffect` 替代 `componentDidMount` + `componentDidUpdate`）。