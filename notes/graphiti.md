# Graphiti

| 源仓库 | [getzep/graphiti](https://github.com/getzep/graphiti) |
|---|---|
| 协议 | Apache-2.0 |
| 主语言 | Python |
| 笔记基准 | commit `72b5c59`（2026-10-28） |

Graphiti 是 Zep（一个 agent 记忆产品）背后的开源引擎，定位"为 AI agent 构建时序上下文图谱"（`README.md:37-39`）。GitHub star 约 3.2 万。它把 agent 的对话和事件实时加工成一张知识图谱，但和传统知识图谱有个关键区别：每条事实带着时间属性，既能回答"现在什么是对的"，也能回答"什么时候开始对的、什么时候不再对"。用官方的说法："tracking what's true now and what was true before"（`README.md:48`）。

先澄清两个常见误解：第一，graphiti 不是 agent 框架——它自己明确说 "Neither is an agent memory product: Graphiti is a framework for building graphs"（`README.md:89-90`），循环在外部 agent 那里，它只管建图和查图；第二，它是纯 Python 项目，没有 Rust 组件（网上有些文章提的 Rust 服务不存在，`server/` 是 Python 的 FastAPI 服务）。

这份笔记按问题组织，不按 graphiti 的开发时间线：每个问题讲清 graphiti 现在是怎么解的、为什么这么解、代价是什么。项目演进过程放在文末修订记录里。笔记起于 commit `72b5c59`（2026-10-28），跟进到 commit `72b5c59`（2026-10-28）。

## 它要解决什么问题

给 agent 配长期记忆，主流做法有两类：向量记忆（mem0 那样存事实文本）和静态知识图谱（把实体关系抽出来存图）。两者都有一个共同盲区：**事实是有寿命的**。"用户住在北京"这个事实，在用户搬家那天起就不再为真，但文本还在库里、图的边也还连着。静态图谱要么删掉旧边（丢历史），要么留着（污染查询），没有中间态。

graphiti 的切入点是给图谱加上完整的时间维度：每条事实（图上的边）记录四个时间字段，分两条时间轴——事实在现实世界里何时成立（valid_at 到 invalid_at），以及这条数据何时进入系统、何时被替代（created_at 到 expired_at）。查询默认只取"现在仍成立"的事实，需要时也能把历史翻出来："用户 3 月到 8 月住北京、9 月起住上海"是两个都保留、各自带时间区间的边。这套双时间轴设计（bi-temporal，双时态，数据库领域的经典思想）是理解全仓库一切代码的钥匙。

## 仓库怎么分工

- `graphiti_core/`——核心库。`graphiti.py` 是编排入口（`Graphiti` 类）；`nodes.py`/`edges.py` 是数据模型；`prompts/` 是全部提示词模板；`utils/maintenance/` 是建图时的加工逻辑（节点/边抽取、消歧、失效判定、社区检测）；`search/` 是检索；`driver/` 是图数据库驱动抽象。
- `mcp_server/`——MCP 服务端，把建图和查图暴露成十几个 MCP 工具，供 Claude Code 这类编码工具直连。
- `server/`——REST 服务（FastAPI），提供消息摄取、检索、删除的 HTTP 接口。
- `tests/`——集成测试和端到端评测。

依赖方向简单：核心库不依赖两个服务；服务只是核心库的两种包装。

## 主轴：事实会过时，图谱要能回答"现在还成立吗"

整张图谱的时间语义落在实体边的四个字段上（`graphiti_core/edges.py:263-290`）：

| 字段 | 时间轴 | 含义 |
|---|---|---|
| `created_at` | 摄取轴 | 这条数据何时进入系统 |
| `expired_at` | 摄取轴 | 这条数据何时被更新的数据替代 |
| `valid_at` | 事件轴 | 事实在现实世界何时开始为真 |
| `invalid_at` | 事件轴 | 事实在现实世界何时不再为真 |

两条轴分开的原因：数据的生命周期和事实的生命周期不同步。用户昨天说"我上个月搬去了上海"，这条消息今天入库（created_at=今天），但事实从上个月开始成立（valid_at=上个月）。拆开两条轴，系统才能既回答"现在查库会查到什么"（摄取轴），又回答"当时世界是什么样"（事件轴）。

派生出三个贯穿全仓库的机制，后面各节展开：

1. **失效不删除**。被替代的旧事实写上 `expired_at` 和 `invalid_at`，数据留在图里，检索默认跳过（`README.md:123-125`）。
2. **LLM 提名、时间规则裁决**。发现矛盾（"用户住上海" vs 旧边"用户住北京"）时，大模型只负责指出"哪些旧事实矛盾"，是否真的失效由确定性的时间区间重叠规则裁决（`edge_operations.py:715-783`）——比"模型说了算"可预测得多。
3. **边是记忆的本体**。`Graphiti.search()` 默认返回的就是边（事实）列表而不是节点（`graphiti.py:1921`）——agent 要的是"现在还成立的结论"，不是实体名单。

## 数据模型

四种节点、一种核心边，全是 pydantic 模型：

- `EpisodicNode`（episode 节点，`nodes.py:318-332`）：一条原始输入（一条消息、一段文本、一个 JSON 记录）的原样存档，带内容、来源类型和时间。它是整张图的"证据层"：所有结论都能回溯到 episode。
- `EntityNode`（实体节点，`nodes.py:499`）：从 episode 里抽出的实体（人、组织、地点、概念），带自动演进的摘要。
- `CommunityNode`（社区节点，`nodes.py:727`）：实体聚类的摘要节点（可选功能）。
- `SagaNode`（`nodes.py:907`）：一串 episode 的滚动摘要，记录首尾 episode 的指针——长对话不必逐条回放。
- `EntityEdge`（实体边，`edges.py:263-290`）：一条事实陈述（`fact` 文本，如"张三任职于 A 公司"）+ 双时间轴四字段 + `episodes` 列表（哪些 episode 支持这条边，`:267-270`，即溯源记录）。

节点上没有时间语义——时间全在边上；节点的"变化"体现为摘要随新 episode 演进。这个决定把复杂度集中到了一处。

## 模型接入

三个模型角色各有抽象基类：`LLMClient`（抽取和判重）、`EmbedderClient`（向量化）、`CrossEncoderClient`（检索重排序，一个对小模型逐条给"查询-文档"相关度打分的组件）。实现类在 `graphiti_core/llm_client/`（openai、anthropic、gemini、groq 及 OpenAI 兼容端点等）、`embedder/`、`cross_encoder/`。

默认组合：LLM 用 openai 的 `gpt-6-luna`（`llm_client/openai_base_client.py:35`）、嵌入用 `text-embedding-3-small`（`embedder/openai.py:24`）、图库默认 Neo4j（`graphiti.py:335`）。两个"默认"容易搞混：核心库默认 Neo4j，但 MCP 服务端文档默认 FalkorDB（`mcp_server/README.md:126`）——两处默认不一致，部署时要显式指定。另外 Kuzu 驱动虽在代码里，官方已宣布废弃（上游 Kuzu 项目停止开发，`README.md:215`）。

## 建图

`add_episode()`（`graphiti_core/graphiti.py:1334`）是一次完整的加工：一段输入进来，图谱相应更新。全程八步：

1. **作用域解析**：确定这批数据写到哪个 group_id（租户标识），必要时切到对应数据库（`graphiti.py:1430`）。
2. **取前文**：检索最近 N 条 episode 作为上下文，让抽取模型知道前情（`graphiti.py:1435-1445`）。
3. **建 episode 节点**：原始输入先原样存档（`graphiti.py:1448-1461`）。
4. **实体抽取**：大模型按提示词从输入里抽实体（`utils/maintenance/node_operations.py:73-162`，提示词在 `prompts/extract_nodes.py`）。
5. **实体消歧**（这步是质量关键，见下文）。
6. **边抽取**：大模型抽"实体-关系-实体"三元组；提到但没定义的实体（悬空端点）会被物化成新节点（`edge_operations.py:156-158`）。
7. **边判重与失效判定**（主轴机制，见下文）。
8. **落库与社区更新**：写 episode、MENTIONS 边、实体与关系边；社区检测是可选项，默认关（`graphiti.py:1343`）。

**实体消歧是三级漏斗**（`node_operations.py:816-910`）："张三"和"Zhang San"和"老张"是不是同一个人？第一级用身份属性精确匹配（邮箱、手机号这类字段相同就直接合并）；第二级用 MinHash+LSH 做相似初筛（MinHash 是把文本变成简签名的算法，LSH 是让相似的签名大概率落进同一个桶，两者合起来能在大规模数据里快速找出候选相似项，`dedup_helpers.py:104-132`），筛出候选后再算精确相似度；第三级才用大模型判断"这两个名字指同一个实体吗"（提示词 `prompts/dedupe_nodes.py:54`）。便宜的先上，模型只处理前两级剩下的硬骨头。合并结果是一张映射表：抽取出的新节点指向已有的旧节点。

**边判重与失效是"LLM 提名、规则裁决"两段式**。新边进来，先用混合检索找同端点的相似旧边；逐字相同直接判重复；否则让大模型判定新边与哪些旧边是"重复"或"矛盾"（`edge_operations.py:1018-1039`）。到这里模型的工作结束。接下来是纯代码的时间规则（`_apply_temporal_invalidation`，`edge_operations.py:715-745`）：被判矛盾的旧边，只有当两条边的时间区间真的重叠才失效——"用户 3 月住北京"和"用户 9 月住上海"时间不重叠，不算矛盾，是两段并存的生活。失效即写 `invalid_at`（新边生效时刻）和 `expired_at`（now）。

时间从哪来？`add_episode()` 要求调用方传 `reference_time`（这条输入的参考时间，`graphiti.py:1339`），事实自身的起止时间由一次小模型调用从文本里抽（`edge_operations.py:786-831`）。提示词里有一条容易忽略的规则：未来计划（"Emma 将于 12 月搬去东京"）的 `valid_at` 设为输入的参考时间而不是 12 月——图谱记录"这个计划从现在开始为真"，12 月这个日期留在事实文本里（`prompts/extract_edges.py:183`）。同样，"截止到明年 3 月"这类软期限不触发 `invalid_at`（`edges.py:283` 明确写着 "soft deadlines are not invalid_at"）。

## 记忆检索

检索入口两个：`search()`（`graphiti.py:1913`）返回事实边列表，是给 agent 用的默认口；`search_()`（`graphiti.py:1989`）返回包含节点、episode、社区的完整 `SearchResults`。

底层（`graphiti_core/search/search.py:98-256`）对边、节点、episode、社区四路并发检索。每路可组合三种检索方法和一种重排器：检索方法有余弦相似度（向量）、BM25（关键词全文，走图数据库自带的全文索引）、BFS（从指定中心节点沿图扩散几跳）；重排器默认 RRF（Reciprocal Rank Fusion，把多路检索的排名按 1/(排名+k) 融合成一个分数，`search_utils.py:1786-1801`），可选交叉编码器精排（更准更慢，需显式选用预设配方）。常用配方在 `search/search_config_recipes.py`：如 `EDGE_HYBRID_SEARCH_RRF`（`:111`）。

一个体现设计复用性的细节：建图时的边判重、找矛盾候选，调用的就是同一套 `EDGE_HYBRID_SEARCH_RRF` 检索（`edge_operations.py:526-539`）——对内"找出和这条新边相似/矛盾的旧边"与对外"找出和查询相似的事实"是同一个问题，一套实现两处用。

过滤用 `SearchFilters`（`search/search_filters.py:55-73`）：按节点类型、边类型、来源/目标节点、UUID 列表过滤，四时间字段都支持区间和"是否为空"条件——"查 6 月时点有效的事实"这类时点查询就靠它。

## 更新与遗忘

核心原则：**失效不删除**。旧事实被新事实替代，写上失效时间即可，历史完整保留。

真正的删除有三个口：模型级 `Node.delete`/`Edge.delete`；`remove_episode(episode_uuid)`（`graphiti.py:2174-2213`）值得单独讲——删一条原始输入时，不是级联删掉它产生的一切，而是**按证据处理**：只删"没有任何其他 episode 支持"的边（`:2181-2190`）和"仅被该 episode 提及"的节点（`:2192-2212`），被多处引用的实体留着并重建摘要。这个设计的前提就是每条边的 `episodes` 溯源列表：证据没了，结论才撤。

遗忘时间语义上也有：查询默认只返回"现在仍成立"的边（expired 的跳过），但 `SearchFilters` 可以显式把失效边查回来——遗忘是查询层的默认行为，不是物理删除。

## 隔离与多租户

隔离主键是 `group_id`（`nodes.py:96` 的注释就是 "partition of the graph"）：所有节点、边、episode 都带它，全文检索强制拼上 group_id 过滤（`search_utils.py:85-96`）。更进一层，group_id 可以映射到独立的物理数据库：作用域解析发现 group_id 与当前数据库不同时，调用 `driver.clone(database=group_id)` 切库（`graphiti.py:1304-1332`）。这个机制的由来很诚实：旧实现并发时共享同一个 driver 对象，两个租户的数据会写错库（代码注释里引用了他们自己的 issue #1676）——修 bug 修出来的多租户特性。

## 服务化与扩展

- **MCP 服务端**：13 个工具，覆盖建图（add_memory、add_triplet——不调模型直接插三元组的口）、检索（search_memory_facts、search_nodes）、管理（删除、建社区、清空）（`mcp_server/src/graphiti_mcp_server.py` 内多处）。给编码 agent 当记忆后端是最轻的接入方式。
- **REST 服务**：消息摄取、检索、各类删除与清空（`server/graph_service/routers/`）。
- **驱动抽象**：`GraphDriver` 基类（`driver/driver.py:90`）带两个可选能力接口——驱动可以接管检索和图操作的原生实现（比如用数据库自带的全文索引），抽象不挡性能。

## 可观测性与评估

遥测用 PostHog 匿名事件，**默认开启**（环境变量 `GRAPHITI_TELEMETRY_ENABLED=false` 关闭，`telemetry/telemetry.py:36-38`——注意默认值是 true），初始化即上报。追踪按 OpenTelemetry 标准留了注入点：`Graphiti(tracer=...)` 传入后，add_episode 和 search 内部自动建 span（`graphiti.py:1432`）；LLM 侧另有 token 记账器。评测代码在 `tests/evals/`：多 session 对话回放建图后对比图谱质量。

## Agent Loop

没有。方法面只有建图、检索、社区、删除这些图谱操作，没有决策循环。README 原话见"它要解决什么问题"节。外部 agent（比如接了它 MCP 的编码工具）负责决定"什么时候存、什么时候查"。

## 设计原则

- **双时间轴是第一公民**。为了让"现在还成立吗"有确定答案，四个时间字段进了数据模型、进了检索过滤器、进了提示词规则。代价是每次写入要额外抽时间、每条边多存四个字段。
- **LLM 提名、规则裁决**。凡是能用确定性规则下判断的地方（时间区间重叠），不让模型拍板；模型只做它擅长的语义判断（这两句话矛盾吗）。失效判定因此可测试、可复现。
- **证据驱动**。episode 原样存档、边带溯源列表、删除按证据裁决。图上任何一条事实都能回答"你从哪知道的"。
- **串行写入**。同一 group 内官方建议 episode 逐条串行加入（`graphiti.py:1410-1413` 的文档注释）——边失效判定依赖先后顺序，乱序并发会判错。吞吐换正确性，写入密集场景要考虑批处理接口 `add_episode_bulk`。

## 可复用的做法

- **双时间轴四字段**（`edges.py:271-287`）：任何"事实会变化"的存储（配置、定价、权限、用户画像）都适用。两条轴分开是对的关键：数据进入系统的时刻和事实生效的时刻是两回事。
- **LLM 提名 + 确定性规则裁决**（`edge_operations.py:715-783`）：让模型参与判断但不当裁判——模型输出候选结论，规则代码给出最终决定。适合一切"模型判断 + 硬约束"混合的决策点。
- **三级消歧漏斗**（`node_operations.py:816-910`）：实体去重（以及一切"判断两个东西是不是同一个"的问题）先精确匹配、再便宜算法初筛、最后才上模型。控制模型调用量同时保住准确率。
- **失效不删除 + 溯源列表**（`edges.py:267-270`、`graphiti.py:2181-2190`）：删除原始数据时按"还有没有别的证据支持"决定级联范围，比"连带删除一切"安全得多。
- **检索能力复用给写入**（`edge_operations.py:526-539`）：写入时的"找相似旧数据"和查询时的"找相似结果"是同一个检索问题，一套实现两处用，避免两套检索行为不一致。
- **错误修复转化成隔离特性**（`graphiti.py:1304-1332`）：并发写错库的 bug，修成了"租户可映射独立数据库"的特性——事故复盘时值得想想能否这样转化。

## 修订记录

- 2026-10-30 初版：基于 commit `72b5c59` 完整成文。覆盖双时间轴数据模型、add_episode 建图八步、三级实体消歧漏斗、"LLM 提名 + 时间规则裁决"的失效机制、四路混合检索、证据驱动的删除与 group_id 隔离。
