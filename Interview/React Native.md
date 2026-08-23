
RN 新架构：完整启动 + 渲染全流程（面试口述版，一气呵成）

我从启动到渲染，把 JSI → Hermes → Metro → TurboModule → Fabric 整条链路串一遍，这是面试里最能体现深度的回答。

一、打包阶段（Metro + Hermes）

1. 开发/打包时，Metro 作为构建工具，从入口 index.js 开始做依赖解析、Babel 转译。

2. 开启 Hermes 后，Metro 会把所有 JS 代码通过 AOT 预编译 生成 Hermes 字节码 bundle，体积更小、不再需要运行时解析 JS 文本。

3. 同时 Codegen 工作完成：

◦ 根据 TS 接口定义，生成 TurboModule 的 C++/Native 绑定代码

◦ 生成 Fabric 组件的 ComponentDescriptor、Props、ShadowNode 绑定

这一步打好了：字节码 bundle + 跨语言绑定代码。

二、App 启动阶段（Native → JS 初始化）

1. Native 侧启动：

◦ iOS：RCTFabricSurface

◦ Android：FabricSurface

2. 创建 JS Runtime，并初始化 Hermes 引擎。

3. 在 Hermes 内部初始化 JSI（JavaScript Interface）：

◦ JSI 是一套轻量 API，让 JS 能直接持有并调用 C++ 对象/方法

◦ 不再需要 Bridge、JSON 序列化、消息队列

4. 加载并执行 Hermes 字节码 bundle，JS 环境启动。

三、JS 与 Native 双向安装（JSI 打通）

1. C++ 侧将全局关键对象通过 JSI 注入到 JS 全局：

◦ FabricUIManager：渲染核心

◦ TurboModuleManager：模块管理

2. JS 侧可以直接：

◦ 调用 C++ 函数

◦ 访问 C++ 对象

◦ 传递函数、回调，而不是传字符串

3. 至此：JS ↔ C++ 完全打通，同步互通。

四、TurboModule 模块初始化（懒加载）

1. JS 业务代码需要调用 Native 能力（如网络、定位、存储、自定义模块）时，从 TurboModuleRegistry 获取模块。

2. TurboModule 懒加载：

◦ 第一次调用时才通过 C++ 找到对应 Native 模块

◦ 实例化并通过 JSI 暴露给 JS

◦ 之后调用都是直接同步/异步调用，无桥接开销

3. 相比旧架构：启动时不用注册所有模块，启动更快、内存更低。

五、React 渲染开始（JS Fiber → Fabric ShadowTree）

1. JS 执行 AppRegistry.runApplication()，进入 React 渲染流程。

2. React Fiber 构建虚拟 DOM，React Fabric Renderer 接管渲染：

◦ 不再发指令给 Bridge

◦ 直接通过 JSI 同步调用 C++ 的 FabricUIManager.createNode

3. C++ 侧创建对应 ShadowNode，并逐步构建整棵 ShadowTree：

◦ ShadowTree 是不可变数据结构

◦ 每个节点自带 YogaNode

六、布局计算（C++ 后台线程）

1. ShadowTree 构建完成后，在后台 C++ 线程做：

◦ Yoga Flex 布局计算（完全下沉到 C++）

◦ 得出所有节点的 x/y/width/height

2. 对 新旧两棵 ShadowTree 做 Diff，生成最小变更集合：

◦ Create

◦ Update

◦ Delete

◦ Move

3. 把这些操作封装成 MountItem 队列，交给 MountingCoordinator。

这一步不阻塞 UI 线程。

七、Mount 阶段（UI 主线程渲染真实视图）

1. MountingManager 在 Native 主线程执行：

◦ 批量 apply MountItem

◦ 创建/更新/删除对应的 iOS/Android 真实 View

2. 只渲染变化部分，做到增量更新。

3. 同时绑定事件（点击、触摸等），打通 Native → JS 的事件回调。

八、交互更新流程（用户点击 → 界面刷新）

1. 用户点击 Native View，事件通过 JSI 同步派发到 JS。

2. JS 触发 setState，React 开始新一轮 Concurrent 渲染。

3. Fiber 重新构建 → JSI 同步更新 C++ ShadowTree。

4. 后台线程 Yoga 布局 → Diff → MountItem。

5. 主线程增量刷新 UI。

整个流程：同步、低延迟、无批量延迟、无序列化开销。

最终一句话总结（面试收尾）

RN 新架构的整体流程就是：

Metro 打包生成 Hermes 字节码，启动后通过 JSI 打通 JS 与 C++；TurboModule 实现模块懒加载与高效调用；Fabric 负责在 C++ 构建不可变 ShadowTree，后台做 Yoga 布局与增量 Diff，最终在主线程轻量挂载渲染；配合 Hermes AOT 预编译，实现启动更快、渲染更流畅、内存更低的跨端渲染体验。

JS 侧渲染阶段（React Render Phase）

这是纯 JavaScript/React 侧的逻辑。

1. 触发更新： 应用启动（runApplication）或用户交互触发状态变更（setState）。
    

2. 构建 Fiber 树： React 核心算法（Reconciler）开始执行，对比状态，生成/更新 React Fiber 树。
    

3. 支持并发： 得益于新架构，这里的渲染是支持 React 18 Concurrent Mode（并发模式）的，高优先级的任务（如动画、用户输入）可以打断低优先级的渲染任务。
    

第二阶段：C++ 侧提交阶段（Commit Phase & Shadow Tree）

在这个阶段，JS 的数据流向 C++，构建底层的渲染树。

4. JSI 同步调用： React 的 Fabric Renderer 接管 Fiber 树的输出，不再像旧架构那样把 UI 指令转成 JSON 发给 Bridge，而是通过 JSI（JavaScript Interface）直接同步调用 C++ 层的 FabricUIManager。
    

5. 构建 Shadow Tree： C++ 接收到指令后，在 C++ 内存中构建一棵对应的 Shadow Tree（影子树）。
    

- 树上的每个节点叫 ShadowNode，它包含了这个组件的所有属性（Props）、事件回调指针以及布局样式。
    

- Shadow Tree 是不可变数据结构（Immutable），每次更新都会生成一棵克隆的新树，保证线程安全。
    

第三阶段：C++ 后台布局计算（Layout Phase & Yoga）

这个阶段解决的是“UI 元素到底在屏幕什么位置、有多大”的问题。

6. Yoga 引擎接管： 当新的 Shadow Tree 提交后，C++ 层会调用内置的跨平台布局引擎 Yoga。
    

7. Flexbox 转换： Yoga 引擎解析前端熟悉的 Flexbox 样式（如 flexDirection: 'row', justifyContent: 'center'）。
    

8. 后台计算： 计算出每个节点在屏幕上的绝对坐标和尺寸（x, y, width, height）。
    

- 核心优势： 整个布局计算过程完全在 C++ 的后台线程中进行，不阻塞 JS 线程，也不阻塞原生 UI 主线程。
    

第四阶段：原生侧对比与挂载（Diff & Mount Phase）

这是最终在手机屏幕上画出视图的阶段。

9. C++ 增量 Diff： 布局计算完成后，C++ 层会对“旧的 Shadow Tree”和“刚算好布局的新的 Shadow Tree”进行 Diff（差异对比）。
    

10. 生成指令（MountItem）： Diff 出的结果被封装成一个个极简的变更指令，称为 MountItem（例如：创建节点、更新布局、删除节点）。
    

11. UI 线程挂载： 这一批 MountItem 队列会被交给原生的 MountingManager。原生操作系统（iOS/Android）在 UI 主线程中“消费”这些指令。
    

- iOS 端： 按照指令创建或更新 UIView，并调用 iOS 的 CoreAnimation 渲染上屏。
    

- Android 端： 按照指令创建或更新 android.view.View，并通过 Android 的视图渲染机制上屏。
    

面试/业务一句话总结：

RN 到原生的渲染全流程是：JS 侧根据状态生成 Fiber 树 $\rightarrow$ 通过 JSI 同步映射为 C++ 侧的 Shadow Tree $\rightarrow$ 在 C++ 后台线程由 Yoga 引擎完成布局计算并 Diff 生成变更指令 $\rightarrow$ 最后交由 Native UI 主线程增量挂载真实的 iOS/Android 原生视图。整个过程无桥接序列化，流畅度极高。