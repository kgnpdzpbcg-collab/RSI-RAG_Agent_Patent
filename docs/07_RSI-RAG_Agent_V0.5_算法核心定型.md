# RSI-RAG Agent V0.5 算法核心定型

> 状态：核心方案冻结稿，用于后续 Feature Chart、权利要求收敛与 V0.5 专利正文重写。
>
> 目标：固定当前已经完成的领域化算法主线，避免后续因上下文过长导致方案漂移。
>
> 本稿不是正式专利申请文本，而是算法与专利核心机制说明。

---

# 1. V0.5 的核心判断

V0.4 的主要问题在于：航空电子装备只存在于标题、文档类型和示例中，Agentic RAG、检索轨迹、REU、Policy Overlay 等核心算法本身仍高度通用，容易被理解为“通用 RAG/Agent/Memory 方案在航空电子领域中的应用”。

V0.5 的重构目标不是继续增加通用 Agent 模块，而是让航空电子装备的知识结构和维修逻辑直接决定算法内部的状态、门控、检索动作和长期学习对象。

V0.5 的核心技术链冻结为：

航电证据需求建模
→ 知识适用性约束
→ 证据状态形成
→ Evidence Gap 驱动检索
→ 验证状态迁移
→ 状态迁移经验递归更新检索动作

最终要保护的不是某个名词本身，而是上述各环节之间的因果处理关系和闭环耦合关系。

---

# 2. 总体算法结构

~~~text
航空电子知识任务 q
        ↓
任务解析与领域上下文提取
        ↓
AERT：生成当前任务的领域证据需求模板 Rq
        ↓
形成初始证据状态 Σ0
        ↓
执行初始检索
        ↓
候选文档/片段 → KEO
        ↓
设备/模块/版本/工况适用性校验
        ↓
仅允许有效证据更新证据状态 Σt
        ↓
根据 AERT 与 Σt 计算 Evidence Gap Gt
        ↓
基于 Gt 限定领域检索动作空间 A(Gt)
        ↓
结合历史状态迁移经验选择动作 At
        ↓
执行检索 → 获得新 KEO
        ↓
重新做适用性与支撑性验证
        ↓
得到新证据状态 Σt+1
        ↓
检查：
强制证据槽位是否满足？
是否存在未解决冲突？
        ↓
否：继续检索
是：输出知识服务结果
        ↓
记录已验证状态迁移
(Ct, Σt, At, Σt+1, Vt)
        ↓
更新 Gap/State-Action Policy
        ↓
未来相似状态下改变 Agent 的检索动作先验
~~~

---

# 3. 数据结构一：AERT —— 航空电子证据需求模板

## 3.1 定义

AERT（Avionics Evidence Requirement Template）用于把用户自然语言问题转换成一个工程证据要求，而不是直接进入普通语义检索。

定义：

Rq = <Cq, Eq, Mq, Θq, Dq>

其中：

- Cq：当前航空电子任务上下文；
- Eq：本任务所需证据槽位集合；
- Mq：必须满足的强制证据槽位集合；
- Θq：每个槽位从 EMPTY/PARTIAL 转为 SATISFIED 的判定条件；
- Dq：证据槽位之间的依赖关系。

AERT 的技术意义不是“列一个信息清单”，而是定义什么证据可以回答当前航空电子任务、证据何时算充分、哪些证据必须先验证、哪些证据具有前置依赖。

## 3.2 航空电子任务上下文 Cq

建议固定字段包括：

~~~text
equipment_model        装备/型号
board_or_module        板卡、功能模块、通道或接口
task_type              任务类型
operating_condition    当前工况/供电状态/任务阶段
maintenance_state      维护、BIT、自检或测试状态
hardware_revision      硬件版本
software_revision      软件版本（若存在）
document_baseline      当前有效文档基线
fault_code             BIT/BITE/告警代码（若存在）
known_symptoms         已知异常现象
~~~

字段允许为 UNKNOWN。UNKNOWN 本身可以进入 Evidence Gap，触发适用性补全动作。

---

# 4. AERT 任务模板库

AERT 不是一个万能模板，而是根据航空电子知识任务类型选择不同证据槽位。

## 4.1 故障原因分析模板

E_fault = {
effectivity,
symptom,
event,
principle,
mechanism,
test,
criterion,
action,
postcheck
}

对应：

- effectivity：当前设备/模块/版本/工况适用性；
- symptom：故障现象或异常表现；
- event：BIT/BITE、告警、复位或事件信息；
- principle：相关功能模块工作原理；
- mechanism：候选故障机理或可能原因；
- test：区分性测试点和测试步骤；
- criterion：正常范围、阈值或合格判据；
- action：后续检查或维修动作；
- postcheck：维修/操作后的验证步骤。

## 4.2 测试规程查询模板

E_test = {
effectivity,
precondition,
location,
instrument,
procedure,
criterion,
safety,
postcheck
}

## 4.3 BIT/BITE 故障码模板

E_BIT = {
effectivity,
code,
meaning,
trigger,
cause,
check,
limitation,
action
}

## 4.4 参数查询模板

E_parameter = {
effectivity,
parameter,
value,
unit,
condition,
source
}

## 4.5 工作原理查询模板

E_principle = {
effectivity,
input,
output,
functional_chain,
key_component,
control_relation
}

---

# 5. 证据依赖关系 Dq

V0.5 不把证据槽位视为平面并列项。

例如故障分析任务可以定义部分依赖关系：

effectivity ≺ test

表示测试步骤只有在设备/模块/版本适用性被确认后才能转为 SATISFIED。

类似地：

test AND condition ≺ criterion

表示“正常范围/合格判据”必须绑定明确测试条件后才可作为最终判据。

还可以有：

mechanism AND test AND criterion ≺ action

表示维修处置建议应建立在已确认的机理/检查证据与判据基础上。

这使算法区别于普通的 missing information list。

---

# 6. 证据状态机

每个证据槽位定义四种状态：

state(ej) ∈ {EMPTY, PARTIAL, SATISFIED, CONFLICT}

## 6.1 EMPTY

没有获得对应类型的证据。

## 6.2 PARTIAL

存在相关信息，但至少满足以下一种情况：

- 证据内容不完整；
- 装备/模块适用性未知；
- 文档版本适用性未知；
- 测试条件缺失；
- 只有单一非权威来源；
- 前置依赖槽位尚未满足。

## 6.3 SATISFIED

证据内容满足当前槽位要求，且：

- 适用性验证通过；
- 来源可追溯；
- 前置依赖满足；
- 没有未解决的同级冲突。

## 6.4 CONFLICT

存在两个或多个对当前任务均可能适用但相互矛盾的证据，例如：

- 不同版本手册给出不同测试步骤；
- 技术通告与旧版维修手册冲突；
- 故障代码解释与测试规程结果不一致；
- 历史案例与当前有效维修规程存在不一致。

---

# 7. 证据充分性与终止条件

辅助覆盖度：

C_t = sum(w_j c_j) / sum(w_j)

其中：

c_j = 0      when EMPTY
c_j = 0.5    when PARTIAL
c_j = 1      when SATISFIED
c_j = 0      when CONFLICT

但最终停止不能只依赖覆盖度。

终止条件需要同时满足：

1. C_t >= τ_C；
2. 对所有强制证据槽位 e_j ∈ M_q，state(e_j) = SATISFIED；
3. N_conflict = 0。

因此，即使系统已经获得大量高相关文本，只要缺失关键测试判据或存在未解决的版本冲突，也不得视为任务完成。

---

# 8. 数据结构二：KEO —— Knowledge Evidence Object

普通 RAG 的基本单元是 chunk，本方案中 chunk 不能直接作为最终证据。

候选文档片段经结构化后形成 KEO：

K_i = <Content_i, Type_i, App_i, Prov_i, Claim_i>

建议字段：

~~~text
KEO
├─ content
│  └─ 原始证据文本/表格/条目
├─ evidence_type
│  └─ principle / mechanism / BIT /
│     test / criterion / action / case ...
├─ applicability
│  ├─ equipment
│  ├─ board_or_module
│  ├─ hardware_revision
│  ├─ software_revision
│  ├─ operating_condition
│  ├─ maintenance_state
│  └─ validity
├─ provenance
│  ├─ document_name
│  ├─ document_type
│  ├─ chapter/page/item
│  ├─ revision
│  ├─ effective_date
│  └─ superseded_by
└─ claim
   ├─ supported_slot
   ├─ supported_statement
   └─ limitation
~~~

---

# 9. 航空电子知识适用性门控

本方案不是“相关性越高越可信”，而是“先判断能不能用于当前装备，再判断与当前问题有多相关”。

对 KEO 的关键适用维度：

App = {equipment, module, revision, condition, validity}

每一维可以取：

a_k ∈ {0, u, 1}

其中：

- 0：明确不适用；
- u：未知；
- 1：确认适用。

如果任一关键维度存在明确冲突：

存在 a_k = 0 => H(K_i,q) = 0

该证据不能填充最终强制证据槽位。

若不存在冲突但存在未知：

H(K_i,q) = u

可以作为探索性证据，但不能独立使强制槽位变为 SATISFIED。

仅当关键适用条件均确认时：

H(K_i,q) = 1

才允许作为最终工程证据进入证据状态更新。

---

# 10. 缺口感知检索排序

通过适用性门控后，检索结果进一步按当前证据缺口排序：

Score(K_i | q, G_t)
=
H(K_i,q)
[
λ1 S_sem
+ λ2 S_lex
+ λ3 S_source
+ λ4 S_gap
]

其中：

S_gap(K_i,G_t)
=
sum over e_j in G_t of w_j * cover(K_i,e_j)

与普通 RAG 的区别：

- 高语义相关但只重复已有内容的文档不会持续占据前列；
- 能补充“测试点、正常范围、版本适用性、维修后验证”等当前关键缺口的文档获得更高优先级。

---

# 11. Evidence Gap 的正式定义

当前证据状态：

Σ_t = [state(e1), state(e2), ..., state(em)]

证据缺口：

G_t = {e_j | state(e_j) != SATISFIED}

进一步分为：

G_t = G_miss ∪ G_uncertain ∪ G_conflict

分别表示：

- G_miss：完全缺失；
- G_uncertain：内容存在但完整性/适用性不足；
- G_conflict：存在未解决冲突。

不同类型 Gap 对应不同检索动作。

---

# 12. 航空电子领域检索动作空间

Agent 不直接在任意 Tool 上自由搜索，而是先由 Gap 类型限定领域动作。

典型映射：

| Evidence Gap | 领域检索动作 |
|---|---|
| 缺设备/版本适用性 | 当前型号/构型/Revision/有效性信息检索 |
| 缺工作原理 | 原理/设计资料限定 + 模块功能术语扩展 |
| 缺 BIT/BITE 含义 | 当前型号故障代码表限定检索 |
| 缺故障机理 | 工作原理 + 故障案例 + 部件/功能术语扩展 |
| 缺测试位置 | 测试规程/维修手册限定检索 |
| 缺测试步骤 | 命中测试条目后读取页级/相邻上下文 |
| 缺正常范围/判据 | 参数表、规程表格、测试条件联合检索 |
| 缺历史验证案例 | 同型号 + 同工况 + 同现象案例检索 |
| 缺维修动作 | 有效维修手册/排故流程限定检索 |
| 证据冲突 | 最新技术通告 + 最新有效版本 + 上位规程检索 |
| 适用性未知 | Effectivity verification action |

实际底层仍可调用 hybrid search、document-specific search、page context、context expansion 等，但专利层保护的是“领域知识获取动作”，而非具体软件工具名称。

---

# 13. 结构化 Retrieval Action

一次动作定义为：

A_t = <target, source, filter, query, tool, context>

例如：

~~~text
target_gap:
    acceptance_criterion

source:
    test_procedure

filter:
    equipment = current_model
    module = 5V_channel
    revision = effective_revision

query:
    5V output / test point / normal range

tool:
    document-specific search

context:
    matched page + adjacent page
~~~

---

# 14. V0.5 RSI 的核心：验证状态迁移

V0.5 不再把“新增了多少证据”作为最终长期学习对象。

真正记录的是：

Σ_t --A_t--> Σ_t+1

即：

> 在某种航空电子任务上下文中，当证据处于特定状态时，执行某一领域知识获取动作后，哪些证据槽位发生了何种经过验证的状态变化。

这是 V0.5 最核心的 RSI 对象。

---

# 15. 数据结构三：GAEU / Verified State Transition Experience

原 Gap-Action Experience Unit 进一步收敛成状态迁移经验：

X_t = <C_t, Σ_t, G_t, A_t, Σ_t+1, V_t, Cost_t, Prov_t>

其中：

- C_t：航空电子任务上下文；
- Σ_t：动作前证据状态；
- G_t：动作前主要证据缺口；
- A_t：执行的领域检索动作；
- Σ_t+1：动作后证据状态；
- V_t：适用性、来源和答案支撑验证结果；
- Cost_t：检索调用、Token、上下文规模等成本；
- Prov_t：知识库、文档和策略版本。

示例：

~~~text
Context
  task       = fault_analysis
  equipment  = power_module_A
  module     = 5V_channel
  revision   = Rev.C

Before State
  mechanism  = SATISFIED
  test_point = EMPTY
  criterion  = EMPTY

Gap
  test_point
  criterion

Action
  source        = test_procedure
  model_filter  = power_module_A
  revision      = Rev.C
  tool          = document_specific_search

After State
  test_point = SATISFIED
  criterion  = PARTIAL

Validation
  equipment_match = true
  revision_match  = true
  source_valid    = true
~~~

---

# 16. 状态迁移动作效用

总体覆盖变化：

ΔC_t = C_t+1 - C_t

强制槽位新增满足数量：

ΔM_t
=
sum over e_j in M_q of
I[state_t(e_j) != SATISFIED and state_t+1(e_j) = SATISFIED]

动作效用可定义为：

U_t
=
α ΔC_t
+ β ΔM_t
+ γ V_t
- δ Cost_t
- μ Conflict_t

其中：

- V_t：新增证据通过适用性与支撑验证的质量；
- Cost_t：检索开销；
- Conflict_t：动作是否引入新的未解决冲突。

该公式用于一种可实施方式，不需要在独立权利要求中写死具体权重。

---

# 17. Gap/State-Action Policy

长期策略不学习完整自然语言问题，而学习：

Z_t = (C_t, Σ_t, G_t)

即“任务上下文 + 当前证据状态 + 当前主要 Gap”。

对于候选动作：

Q(A|Z)
=
[
sum_i sim(Z,Z_i) * rho_i * U_i
]
/
[
sum_i sim(Z,Z_i) * rho_i + epsilon
]

其中：

- sim(Z,Z_i)：上下文与状态相似度；
- rho_i：历史状态迁移经验的验证可靠度；
- U_i：该动作在历史任务中的证据状态迁移效用。

当前动作：

A_t*
=
argmax over A in A(G_t)
[
Q(A|Z_t) - kappa * Cost(A)
]

当没有足够历史经验时使用基础 Agent Planner；

当历史经验足够时，Gap/State-Action Policy 仅作为动作先验，不完全替代 Planner。

---

# 18. 递归更新

每一个新任务继续产生验证后的状态迁移。

可采用简单递推：

Q_t+1(A|Z)
=
(1 - eta * rho_t) Q_t(A|Z)
+
eta * rho_t * U_t

因此：

- 动作稳定补齐关键证据：动作先验升高；
- 动作只返回重复信息：先验降低；
- 动作引用错误版本：明显降低；
- 动作引入冲突：降低；
- 新版本环境下旧动作失效：根据 Scope 进行分裂或降级。

RSI 只优化“知识获取动作策略”，不声明自主进化全部 Agent 能力。

---

# 19. Policy Scope 与版本约束

每一条长期策略都需要 Scope：

~~~text
Scope
├─ equipment_family
├─ board_or_module
├─ task_type
├─ evidence_state_pattern
├─ gap_type
├─ operating_condition
└─ document_revision_range
~~~

例如：

~~~text
P17
适用：
Power_Module_A
5V channel
fault_analysis
missing(test_point, criterion)
manual <= Rev.C
~~~

当知识基线从 Rev.C 变成 Rev.D 时：

- 旧策略不直接删除；
- 可将旧策略限制为 <= Rev.C；
- 新经验形成 >= Rev.D 的新策略；
- 当前 Revision 未知时，先生成 revision applicability gap。

---

# 20. 典型 DC/DC 实施流程

用户问题：

“某航空电子 DC/DC 模块 5V 输出持续偏低，应如何排查？”

## Step A：AERT

生成故障分析模板。

强制槽位：

~~~text
Effectivity         REQUIRED
Fault symptom       REQUIRED
Working principle   REQUIRED
Candidate mechanism REQUIRED
Test point          REQUIRED
Test procedure      REQUIRED
Criterion           REQUIRED
~~~

当前问题已经提供 Fault symptom。

## Step B：第一轮

初始 Gap：

~~~text
Effectivity
Working principle
Candidate mechanism
Test point
Test procedure
Criterion
~~~

Agent 首先获取当前型号及当前有效维修/技术资料。

通过 KEO 适用性验证后：

~~~text
Effectivity       → SATISFIED
Working principle → SATISFIED
Mechanism         → SATISFIED
Test point        → EMPTY
Procedure         → EMPTY
Criterion         → EMPTY
~~~

## Step C：第二轮

主要 Gap：

~~~text
Test point
Test procedure
Criterion
~~~

历史状态迁移经验提示优先检索当前型号 + 当前 Revision + 测试规程。

执行限定规程检索后：

~~~text
Test point → SATISFIED
Procedure  → SATISFIED
Criterion  → PARTIAL
~~~

记录一次验证状态迁移。

## Step D：第三轮

主要 Gap：

~~~text
Criterion
~~~

历史策略提示：

~~~text
规程条目命中但判据不完整
→ page context + adjacent page
~~~

执行后：

~~~text
Criterion → SATISFIED
~~~

所有强制槽位满足，且无未解决冲突，任务结束。

两轮动作均沉淀成状态迁移经验，供后续相同或相似证据状态复用。

---

# 21. 与 V0.4 的实质区别

| V0.4 | V0.5 |
|---|---|
| 航空电子主要体现在知识源和示例 | 航空电子知识结构进入算法状态定义 |
| REU记录任务轨迹 | 记录领域证据状态迁移 |
| Gap较泛化 | Gap来源于AERT强制证据槽位与依赖 |
| 文档相关性 + metadata | 适用性 Gate 决定证据能否进入状态 |
| Policy学习检索策略 | Policy学习“特定证据状态下的领域动作” |
| Evidence gain偏通用 | 学习 Σ_t → Σ_t+1 的验证状态转换 |
| Action以检索工具为中心 | Action以航空电子知识获取动作语义为中心 |

---

# 22. 当前方案必须保持的边界

1. 不做知识图谱：设备、模块、版本、工况、文档类型和有效性采用结构化 metadata 与证据模板表达。
2. 不把具体 Retriever 作为创新核心：BM25、Dense Retrieval、Reranker、Hybrid Search 都是底层实现。
3. 不把 LLM 自动生成所有 AERT 作为必要技术特征：第一版采用预定义领域任务模板 + LLM/规则选择和实例化。
4. 不把模型微调或强化学习作为必要条件：RSI 可以由经验检索、规则更新和增量统计完成。
5. RSI 仅优化检索知识获取策略：避免扩展为 Agent 全能力自进化。

---

# 23. 当前冻结的核心发明构思

当前建议独立权利要求围绕以下完整处理关系展开：

> 针对航空电子装备知识任务，根据任务类型、装备对象、功能模块、版本和工况建立具有强制证据槽位及槽位依赖关系的领域证据需求；对检索所得知识执行装备、模块、版本及工况适用性校验，并依据通过校验的知识更新当前证据状态；根据当前证据状态中未满足、部分满足或冲突的证据槽位确定证据缺口，并根据所述证据缺口选择对应的领域知识获取动作；在动作执行后重新验证并形成证据状态迁移；记录经过验证的任务上下文、动作前证据状态、检索动作和动作后证据状态，并依据历史证据状态迁移经验调整后续相同或相似证据状态下的检索动作选择。

---

# 24. 后续查新 Stop Rule

后续 Feature Chart 的目的不是“发现任何局部相似点就继续重构”。

仅在以下情况出现时考虑改核心方案：

1. 发现单篇现有技术覆盖上述完整主链；
2. 最接近现有技术只需加入一个明显的行业常规手段，即可直接得到上述完整主链。

如果只是分别存在 missing evidence、version filtering、Agent memory、evidence gain、feedback policy 等局部机制，但没有现有技术给出本方案所述的完整领域证据状态及验证状态迁移闭环，则不因局部相似而继续无限重构。

---

# 25. 当前算法核心的五句话摘要

1. AERT：把航空电子知识任务转换为具有强制槽位和证据依赖关系的领域证据需求。
2. KEO + Effectivity Gate：只有与当前装备、模块、版本和工况适用的知识对象才能正式更新证据状态。
3. Evidence State / Gap：由领域证据状态而非原始 Query 确定下一轮缺什么知识。
4. Domain Retrieval Action：根据 Gap 限定并选择航空电子领域知识获取动作。
5. Verified State Transition RSI：记录并学习 Σ_t --A_t--> Σ_t+1 的验证状态迁移，使历史经验改变未来类似状态下的检索动作先验。

---

**以上内容作为 RSI-RAG Agent V0.5 算法核心冻结稿。**
