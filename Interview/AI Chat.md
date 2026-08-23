我们做的是携程机票的 AI 对话助手，用户问"帮我查上海到北京的便宜航班"，AI 会先展示一个思考过程卡片（"正在理解需求…正在查询航班…思考完成"），然后展示航班推荐卡片。这些卡片不是前端硬编码的，而是用 SDUI 方案——服务端下发 DSL 模板 + 数据，前端按模板动态渲染。 

难点 
传统 SDUI 是一次性下发完整数据渲染一次就结束了。但 AI 对话场景下，一张卡片的内容是逐步生成的。思考卡片要先显示第一步"正在理解需求"，过 500ms 追加第二步"正在查询航班"，最后把状态从"思考中"变成"思考完成"。这三次更新要作用在同一张卡片上，而不是创建三张新卡片。

解决方案 
我设计了一套三层方案：协议层、前端双通道、依赖收集。 

第一层：协议设计 
服务端定义了一组 SDUIAction 指令：render 表示创建新卡片，append 表示往数组追加数据，replace 表示替换某个字段。每条 SSE 消息都带一个 msgId（来自 LLM 的 message_id）和 dataKey（指定更新哪个数据路径）。同一张卡片的所有操作共享同一个 msgId，这是前端找到对应渲染实例的唯一标识。

服务端的 Processor 根据消息类型决定用哪个指令。比如思考开始时，Processor 从模板服务拿到思考卡片的 DSL 模板，把初始数据填进去，设置 SDUIAction: 'render'。后续每产出一步思考内容，设置 SDUIAction: 'append'，dataKey: 'list'。思考结束时，设置 SDUIAction: 'replace'，dataKey: 'status'，值从 'start' 改成 'end'。 

第二层：前端双通道 
前端收到 SSE 消息后，根据 SDUIAction 走两条完全不同的路径。

render 走 React 状态通道：MessageItem 渲染出 FigRender 组件。FigRender 内部解析出 msgId、template、data，注册到一个全局的渲染管理器 PuppyStreamRenderManager，以 msgId 为 key 存入 dataStore 和 templateStore，然后做模板处理、递归渲染。 

append 和 replace 走事件通道：完全不调用 setMessages，直接调用 FigElement.update()。这个方法找到 dataStore 里对应 msgId 的数据，执行内存操作（比如 dataStore.list.push(newItem) 或 dataStore.status = 'end'），然后重新处理模板生成新的节点描述，最后通过事件系统通知受影响的节点更新。 

第三层：依赖收集
即使绕过了 React，如果每次 append 都重新渲染整张卡片的 30-50 个节点，还是浪费。所以我们在模板处理阶段做了依赖收集。

processTemplate 遍历模板树时，每个节点访问了哪些数据字段，就记录下来。比如 Loading 节点的条件表达式是 {status !== 'end'}，它依赖 status 字段；列表容器节点绑定了 {list}，它依赖 list 字段。这些依赖关系存在一个 DataDependencyCollector 里，本质是一个双向映射表：nodeId → [依赖的字段] 和 字段 → [依赖它的 nodeId]。 

更新时，比如 append 到 list，查表得到"只有列表容器节点依赖了 list"，就只通知这一个节点重新渲染。Loading 节点、标题节点、图标节点完全不动。replace status 为 'end' 时，查表得到"Loading 节点和完成图标节点依赖了 status"，只通知这两个节点——Loading 隐藏，图标显示，列表内容不动。

这个机制类似 Vue 的响应式系统：在"读"的时候收集依赖，在"写"的时候精确通知。区别是 React 的 Virtual DOM diff 是"渲染完再比较"，O(n) 遍历整棵树；我们的依赖收集是"更新前就知道谁需要变"，O(1) 查表。

还有一个时序问题 
实际中还遇到过一个边界情况：LLM 输出很快时，render 和第一条 append 可能间隔不到 50ms。render 路径要走 React setState → re-render → FigRender useEffect → register，这个过程需要 1-2 帧。如果 append 在 register 之前到达，渲染管理器里还没有这个 msgId 的实例，更新就会丢失。 

我们在两端做了保护。服务端的流缓冲策略保证 render 和紧跟的 append 之间至少间隔 100ms。前端的 FigElement.update 也有兜底——如果 msgId 还没注册，把更新暂存到 pending 队列，等 register 完成后自动 flush。

效果 
优化前，一次思考过程（5-10 条 SSE 消息）触发 5-10 次消息列表 re-render，每次都导致 SDUI 卡片 30-50 个节点全量重渲染。优化后，只有第一条 render 消息触发一次 React re-render 创建卡片，后续所有 append/replace 都走事件通道，每次只更新 1-2 个节点。React Native 上从掉帧卡顿恢复到稳定 60fps。这套机制后来也被火车票团队复用了。