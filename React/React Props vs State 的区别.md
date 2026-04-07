Props 是从父组件传入的只读数据，组件本身不能修改它，就像函数的参数一样。
State 是组件自己拥有并管理的可变数据，调用 setState / useState 会触发重渲染。

|       | Props      | State         |
| ----- | ---------- | ------------- |
| 来源    | 父组件传入      | 组件自己初始化       |
| 可变性   | 只读（对本组件而言） | 可变（通过 setter） |
| 控制权   | 父组件        | 本组件           |
| 触发重渲染 | 父组件更新时     | 调用 setter 时   |

>复杂项目中要区分”本地状态“、”共享状态“、”服务端状态“，不要将所有数据都塞进state

1. 本地状态（Local State）
   只有当前组件关心，不需要共享。适合放在 `useState/useReducer`中：
```
   // ✅ 正确：弹窗开关、输入框内容、hover 状态 —— 只属于这个组件
	function FilterPanel() {
		const [isOpen, setIsOpen] = useState(false);
		const [inputValue, setInputValue] = useState('');
		// ...
	}
```
    如果把这些放进全局 store，会带来： 每次输入框变化 → 全局 store 更新 → 所有订阅组件重渲染; store 里堆满了和业务无关的 UI 临时状态，难以维护.

2. 共享状态（Shared State)
   多个组件需要读写同一份数据。适合放在状态管理库或提升到共同父组件。
   `src/pages/middle/store/slices/cabin/` 里的 cabin 数据就是典型的共享状态 —— 舱位数据在 CabinList、HighGradePartitionHeader、MixedPartitionMorePrice 等多个容器里都会消费，放进 Zustand slice 是正确选择。
   常见错误： 把共享状态放在某一个子组件的本地 state 里，然后通过层层 props drilling 或者不稳定的事件总线传递 → 数据流不清晰，难以 debug。

3. 服务端状态（Server State）
   本质上数据来自服务器，本地只是缓存的副本。这类状态有完全不同的生命周期需求：加载、缓存、过期、重新请求、错误重试。

   如果用普通 useState 手动管理：
   ```
// ❌ 反模式：手动管理 loading/error/data，代码冗余且缓存逻辑缺失
const [flights, setFlights] = useState([]);
const [loading, setLoading] = useState(false);
const [error, setError] = useState(null);

useEffect(() => {
  setLoading(true);
  fetchFlights().then(setFlights).catch(setError).finally(() => setLoading(false));
}, [dep]);
   ```
   而用 React Query / SWR:
   ```
   // ✅ 服务端状态交给专门工具：自动缓存、去重请求、后台刷新、失效重取
const { data: flights, isLoading, error } = useQuery({
  queryKey: ['flights', dep, arr, date],
  queryFn: () => fetchFlights(dep, arr, date),
  staleTime: 30_000, // 30s 内不重新请求
});
   ```
   把服务端状态塞进 Redux 的代价：
   * 需要手写大量 loading/error/success action；
   * 缓存失效逻辑要自己实现；
   * 多个组件同时请求同一接口时无法自动去重；
   * 数据变化后手动 invalidate，容易遗漏导致展示旧数据

**核心原则：最小化状态的作用域，让每类数据待在最合适的地方。**
