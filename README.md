# RSI-RAG_Agent_Patent

面向航空电子装备知识服务的 **RSI-RAG Agent（Recursive Self-Improving Retrieval-Augmented Generation Agent）** 发明专利工作仓库。

## 当前工作标题

**一种面向航空电子装备知识服务的递归自优化检索增强智能体方法及系统**

## 当前阶段

已完成 Step 10 的 V0.1 技术交底包。当前查新门禁按 `patent-disclosure-skill` 的三态规则记为：

> **带疑点过**

这里不是把“RSI-RAG Agent”名称本身当作新颖性依据，而是把保护主线集中到：

> **已验证检索执行轨迹 → REU（Retrieval Experience Unit）→ 正负经验对比/聚合与冲突消解 → 检索编排策略 → Policy Overlay 修改未来检索任务图 → 双门控、版本化演化与回滚**

## 文件导航

| 文件 | 内容 |
|---|---|
| [docs/01_专利点分析与选定.md](docs/01_专利点分析与选定.md) | Step 3–4：候选专利点、融合方案、总发明构思和边界 |
| [docs/02_查新与区别定位_20260928.md](docs/02_查新与区别定位_20260928.md) | Step 5/5.5：D1/D2/D3、RSIAgent近邻、F1–F4区别特征和组合风险 |
| [docs/03_交底书生成前摘要预览.md](docs/03_交底书生成前摘要预览.md) | Step 6：案件名称、技术问题、核心技术路线、保护重点 |
| [outputs/RSI-RAG_Agent/一种面向航空电子装备知识服务的递归自优化检索增强智能体方法及系统_20260928124506.md](outputs/RSI-RAG_Agent/一种面向航空电子装备知识服务的递归自优化检索增强智能体方法及系统_20260928124506.md) | Step 7：V0.1 完整技术交底书正文 |
| [docs/04_交底书自检记录_20260928.md](docs/04_交底书自检记录_20260928.md) | Step 8：逻辑闭环、创造性主线、算法客体和充分公开自检 |
| [docs/05_现有代码到RSI层的实现映射.md](docs/05_现有代码到RSI层的实现映射.md) | 现有 Agentic RAG 与新增 RSI 外循环的代码实现映射 |

## V0.1 必要技术特征

### F1 已验证检索经验单元

从完整检索执行轨迹中形成带验证状态的 REU，至少描述任务条件、检索动作、执行结果、策略效果诊断和来源版本。

### F2 对比聚合与策略协调

针对条件相近的正向/负向 REU 对比检索动作，并对多条已验证经验进行聚合、适用范围限定和冲突消解，形成检索编排策略。

### F3 Policy Overlay 驱动未来 Agent 规划

保留基础 Planner，将匹配到的长期检索策略作为 Policy Overlay，改变后续检索任务图中的任务分解、查询扩展、知识源路由、工具调用、上下文扩展或停止条件中的至少一项。

### F4 双门控与版本化可逆演化

Outcome Gate 控制“轨迹→经验”，Policy Gate 控制“经验/候选策略→生效策略”；策略保留知识库版本、支持经验、反例和版本日志，可降权、失效或回滚。

## 技术架构

本案采用两个时间尺度的闭环：

- **Agentic RAG 内循环**：规划 → 检索 → 证据评价 → 补查/重规划 → 生成与验证；
- **RSI 跨任务外循环**：检索轨迹 → 独立验证 → REU → 策略聚合/冲突消解 → Policy Memory → Policy Overlay → 后续任务。

## 当前实现基础

现有仓库 `kgnpdzpbcg-collab/agentic-RAG` 可作为内循环实现基础，已有任务规划、混合检索、上下文扩展、证据池、context grader、answer verifier、chain verifier 和 trace。新增实现重点放在 RSI 外层的 trajectory exporter、experience distiller、policy consolidator/reconciler、policy selector/overlay 和 policy journal。

## V0.1 尚未写死的内容

- 不要求知识图谱；
- 不限定具体 LLM、Embedding、Reranker 型号；
- 不限定固定 Top-k、固定检索轮数和固定门槛数值；
- 不把基础模型在线微调作为 RSI 的必要条件；
- 暂未虚构准确率、召回率或效率提升数值，后续以实际实现验证补充。



## 2026-09-28 V0.2：按课题组已有正式专利格式重写

上一版 V0.1 属于“技术交底书”结构。根据课题组已有专利《一种仿测数据深度融合的滚动轴承不平衡开集故障诊断方法》的实际申请文件结构，V0.2 已改为正式初稿逻辑：

**说明书摘要 → 摘要附图 → 权利要求书 → 说明书（技术领域/背景技术/发明内容/附图说明/具体实施方式）→ 说明书附图。**

- [课题组已有专利撰写逻辑分析](docs/06_课题组已有专利撰写逻辑分析.md)
- [RSI-RAG Agent 专利初稿 V0.2](outputs/RSI-RAG_Agent/一种面向航空电子装备知识服务的递归自优化检索增强智能体方法及系统_专利初稿_V0.2.md)

V0.2 将主技术链收敛为三个步骤：

1. **S1 知识任务执行与检索轨迹形成**；
2. **S2 检索经验提炼与策略递归更新**；
3. **S3 策略增强的后续知识服务**。

权利要求1保护上述完整主链，从属权利要求进一步限定 REU、Outcome Gate、正负经验对比、冲突消解、Policy Gate、Policy Overlay 及版本回滚等机制。
