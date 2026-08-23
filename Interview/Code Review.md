背景 
我们团队搭建了一个 CI 自动化工具 autocoder-cli，集成在 GitLab Pipeline 中。其中 code review 功能基于 Claude Agent SDK，在 MR 提交时自动触发，按照团队预定义的 10 条 review 规则（平台规范、feature flag 使用、时区处理、命名规范等）对变更代码逐文件检查，发现违规后自动修复并提交。

二、问题（遇到了什么挑战） 
现象 
通过 4 轮递进测试发现：单文件场景下，即使 diff 达到 615 行，违规检出率始终是 100%。但当 MR 涉及 6 个文件、总 diff 1755 行时，检出率骤降到 17%——6 个文件中只有 1 个被真正 review，其余 5 个的违规全部漏掉。而且 Agent 的日志显示所有文件都标记为"已完成"，它认为自己做完了。 

根因 
原来的实现是把一个 prompt 丢给一个 Claude Code session，让 Agent 自己去发现文件、逐个 review。问题在于 LLM 有 200K token 的 context window 限制，而 context 是累积的——Agent 每调用一次工具（读文件、读规则、grep 检查、编辑修复），输入输出都会留在 context 里。一个文件的完整 review 流程需要 5-8 轮工具调用，消耗约 20-30K tokens。处理完 2-3 个大文件后，context 就累积到 80-100K，后面的文件要么被跳过，要么只做了浅层检查。

本质上是一个有限资源的串行累积问题——单个 session 的 context 空间被前面任务的历史记录逐步蚕食，导致后续任务质量退化。 

三、方案（如何解决）
核心思路 
把串行累积变成并行隔离——每个文件分配一个独立的 Agent session，各自拥有干净的 200K context，互不干扰。 
具体实现，三步改造：
文件发现前置化：新增 getDiffFiles，在代码层面用 git diff --name-only 获取变更文件列表，不再依赖 Agent 自己去发现 

Prompt 聚焦化：为每个子 Agent 生成独立的 prompt，明确指定只 review 哪个文件 

任务分发并发化：复用项目已有的 runFixWorkflow 并发框架，每个文件一个独立 session，最多 5 个并发执行