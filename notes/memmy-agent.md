# Memmy

| 源仓库 | [MemTensor/memmy-agent](https://github.com/MemTensor/memmy-agent) |
|---|---|
| 协议 | MIT |
| 主语言 | TypeScript（GUI 操控与本机历史采集另有 Swift/Go/PowerShell 原生辅助程序） |
| 笔记基准 | commit `dfb43b613`（2026-11-19，v1.2.7） |

Memmy 是 MemTensor（MemOS 的开发团队，本笔记另有 MemOS 专篇）出品的"个人记忆基础设施"，GitHub star 约 2200。它由三块组成：一个跑在本机的记忆服务（默认 `127.0.0.1:18960`）、一套从各家 AI 编码 agent 本地数据里读取工作历史的适配器、以及一个自带的 agent 运行时。三种使用形态：桌面端应用（Electron，用网页技术打包桌面应用的框架）、`memmy` 命令行加终端界面（TUI，基于 Ink，一个用 React 写终端界面的库）、独立编译的 `memmy-memory` 记忆服务命令行。端口规划：记忆服务 18960、网关健康检查 18970、WebUI 18980、OpenAI 兼容 API 18990（`docs/cn/reference/ports.mdx:8-15`）。

这个项目和本笔记系列其他记忆篇的根本差异在接入方式上。mem0、cognee、graphiti 是给应用调用的库，supermemory 靠 SDK 和中间件接入，MemOS 靠插件接入——它们都要求"应用一侧发生点什么"。Memmy 的做法是绕开应用：Claude Code、Codex、Cursor 这些工具本来就把完整会话写在本地磁盘（JSONL 文件或 SQLite 数据库），Memmy 直接读这些文件收割历史，不需要任何一个工具做改造；它自带的适配器覆盖 deepseek-harness、openclaw、hermes、opencode、pi、codex、claude code 等——其中五个在本系列已有专篇。读懂"怎么从别人的本地数据里可靠地收割记忆"是读这份笔记的主线。

这份笔记按问题组织，不按 Memmy 的开发时间线：每个问题讲清 Memmy 现在是怎么解的、为什么这么解、代价是什么。项目演进过程放在文末修订记录里。笔记起于 commit `265f6d7`（2026-11-02），跟进到 commit `dfb43b613`（2026-11-19）。

## 它要解决什么问题

同一个用户会在多个 AI 编码 agent 之间切换：今天用 Claude Code 写代码，明天用 Codex 重构，后天让 OpenClaw 跑自动化任务。每个 agent 都只记得自己会话里的事——你的偏好、项目背景、昨天做到哪，换一个 agent 就归零。常见的补救有两条路：给每个 agent 接同一个记忆后端（mem0 路线，每个接入点都要写适配）；或干脆只用一个自带记忆的 agent（letta-code 路线，被单一 agent 锁住）。

Memmy 的观察是：这些工具各自都会把会话历史完整持久化在本机（Claude Code 写 `~/.claude/projects/` 下的 JSONL，Codex 写 `~/.codex/sessions/` 的 rollout 文件，Cursor 和 OpenCode 各有自己的 SQLite），这些数据本身就是记忆的原材料，只是散落在各自的格式里。Memmy 把收割和共享这两件事做成一个本机服务：定期扫描各工具的本地数据、加上 hook 实时补采，统一整理进一个 SQLite 记忆库，再把检索结果注入回每个工具的提示词。另外它把"整理记忆"本身也交给一个自带 agent——历史摘要、策略归纳、世界模型抽象都是后台任务，不需要用户动手。

## 仓库怎么分工

- `Memory/AgentSourceCore/`——采集层的纯解析库（npm 包名 `@memmy/agent-source-core` 未变，v1.2.6 开发窗口里从仓库顶层挪进了 `Memory/` 目录下）：七个工具的轮次解析器（claude-code、codex、cursor、opencode、openclaw、hermes、deepseek）、统一的 `SourceTurn` 轮次结构（见主轴节）、脱敏器（`secret-redactor.ts`）、token 用量上限计算。不依赖运行时，hook 和离线扫描共用同一套轮次组装代码。旁边的 `Memory/AgentSourceReaders/` 是给桌面端用的只读共享包——把 `Memory/src/agent-source/adapters` 下的读取器和 agent 路径函数编译成 `@memmy/agent-source-readers`，App/backend 只用它做来源发现和类型引用，扫描任务一律交给 Memory 服务执行（包内 README 写明了这条纪律）。
- `Memory/`——记忆服务本体：`storage/`（SQLite 存储与 schema）、`algorithm/`（检索算法）、`service/`（写入、检索、演化服务）、`server/`（HTTP 服务与鉴权）、`cli/`（`memmy-memory` 命令行）、`model/`（嵌入模型）。
- `Knowledge/`——与 Memory 平行的云端文档知识库客户端，带独立的召回注入器（RAG，检索增强生成——先检索资料再拼进提示词的通用做法）；与个人记忆层零耦合，注入用 `<memmy_knowledge_context>` 标签（`Knowledge/src/recall.ts:15` 注释 "Independent recall integration. No Memory imports"）。
- `App/`——桌面端：`frontend/`（React 19 + Vite）、`backend/`（内嵌的本地 Fastify 服务——Fastify 是 Node.js 的一个 Web 服务框架——管桌面应用状态、权限、云账号）、`shell/`（Electron 壳，编排 backend 和 agent 网关子进程）、`native-computer-history/`（本机使用历史采集，Swift/Go/PowerShell）、`native-computer-use/`（GUI 操控，以 MCP 服务形式暴露）。
- `App/memmy-agent/`——agent 运行时与 `memmy` CLI：provider 注册表、ReAct 循环、52 个内置工具类、TUI、OpenAI 兼容 API。
- `Migrations/`——桌面端运行时配置、工作区、应用数据库的版本化迁移；注意记忆库自己的表结构迁移不在这里，内嵌在 `Memory/src/storage/schema.ts`。
- `plugins/`——独立打包的命令插件目录，目前只有 office 一项：DOCX/PDF/PPTX/XLSX 四类文档的解析与格式指南插件，自带打包好的库和四个技能资源，应用安装包不含它、由用户手动安装（`plugins/office/README.md:1-11`）。

## 主轴：记忆从工具本地数据里收割，同一轮对话只收一次

Memmy 的写入设计要同时回答两个问题：从哪收，以及怎么保证不收重。

**从哪收：读原生数据 + hook 补采。** 每个适配器读该 agent 自己的本地持久层，把"用户按一次回车到拿到完整回复"切成一个轮次（turn）：

| 工具 | 数据源 | 轮次标识 |
|---|---|---|
| claude code | `~/.claude/projects/**/*.jsonl` | promptId（`Memory/AgentSourceCore/src/claude-code-source-turn.ts:58-82`） |
| codex | `~/.codex/sessions/**/rollout-*.jsonl` | turn_id（`Memory/AgentSourceCore/src/codex-source-turn.ts:111-134`） |
| cursor | Cursor 的 `state.vscdb` SQLite | user 消息的 bubbleId（`Memory/AgentSourceCore/src/cursor-source-turn.ts:102-108`） |
| opencode | `~/.local/share/opencode/opencode.db`（兼容 v1/v2 两代表结构） | user 消息 id（`Memory/AgentSourceCore/src/opencode-source-turn.ts:151-160`） |
| openclaw | `~/.openclaw` 的 transcript SQLite | runId（`Memory/AgentSourceCore/src/openclaw-source-turn.ts:83-90`） |
| hermes | `~/.hermes/state.db` | `{sessionId}:{行id}`（`Memory/AgentSourceCore/src/hermes-source-turn.ts:122-130`） |
| deepseek harness | `~/.dsh/sessions/**`（zstd 压缩 JSONL） | `{sessionId}:{turn}`（`Memory/AgentSourceCore/src/deepseek-source-turn.ts:169-171`） |

所有适配器产出同一个 `SourceTurn` 结构（`Memory/AgentSourceCore/src/source-turn.ts:20-36`）：来源、会话 id、轮次 id、起止时间、query/answer、工具调用列表，外加一个关键字段 `completionEvidence`（完成证据）——每个适配器必须回答"原生记录里哪个字段代表这轮结束了"（Claude 的 `turn_duration` 系统行、Codex 的 `task_complete`、Hermes 的 `finish_reason=stop` 等）。拿不出完成证据的轮次保持待定状态，不猜测、不用文件时间戳凑数（`Memory/AgentSourceCore/README.md:24-26`）。这个约束保证了"半个回复"不会被当成完整历史收进记忆。

**怎么不收重：确定性身份 + 账本裁决。** 同一轮对话有两条采集通道：工具的 hook 在会话结束时实时提交，记忆服务的定时扫描（启动后 5 分钟首扫、之后每小时一轮，`Memory/src/agent-source/runtime.ts:79-80`）再读一遍磁盘。两条通道跑的是同一份解析代码、从原生字段重建同一个 turnId，提交到同一个入口 `completeSourceTurn`——它在事务里查 `source_turn_captures` 账本表：同轮次没见过就写入（`stored`），见过且内容哈希一致返回 `existing`，不一致报 `conflict`（`Memory/src/service/session/session-turn-service.ts:1537,1582-1634`）。先到先得，不做时间窗或相似度判断。配合四层辅助去重（确定性 memory_key 的 upsert、脱敏后问答文本的 sha256 claim 表、`content_hash` 内容变更检测、user_memory 的归一化文本哈希），多个工具重复接入也不会产生双份记忆。

收割进来的每轮对话存成一条 L1 记忆（ID 由 `trace_` 加确定性哈希生成，`session-turn-service.ts:2328`），摘要和嵌入由后台任务异步补——写入路径本身不调模型，对话不被记忆加工阻塞。episode 归属同样延迟：入库时只在原始轮次上记一个 pending 状态（连同按内容算出的 generation 号一起写进轮次的元数据字段），再排两个后台任务 `episode_assignment`/`episode_finalize` 做归属裁决（`session-turn-service.ts:551-562`）——归属要用模型挑 episode、可能重试多次，放进后台就不会阻塞收割；账本对重复提交的应答里因此多了 `assignmentStatus`（已归属/待归属）字段（`:1597`）。每轮结束后还有一次"回合评判"（turn.judge.v1）：一个模型调用同时判定这轮是新任务还是延续、反馈情绪、该轮 L1 与用户偏好的取舍（`Memory/src/config/index.ts:244-247`，默认开启 `:511`）；关闭时退回"回合开始时判定任务关系、采集只做保守决策"的旧路径。这个"先收原文、后台精炼"的形态与 supermemory 的两阶段写入、MemOS 的 fast/fine 双模是同一个思路，只是 Memmy 的原材料来自收割而不是应用提交。

## 模型接入

agent 侧的 provider 注册表（`App/memmy-agent/src/providers/registry.ts:138-470`）覆盖四类：OpenAI 兼容端点数十家（DeepSeek、Gemini、智谱、DashScope、Kimi、MiniMax、OpenRouter、vLLM、Ollama、LM Studio 等）、原生 Anthropic/Azure/Bedrock、OAuth（开放授权协议）类（OpenAI Codex、GitHub Copilot）、官方托管 `memmy_account`（注册赠送的体验 Token 走这个）。配置在 `~/.memmy/config.yaml`，API key 支持 `${ENV_NAME}` 引用，明文不落配置文件。无密钥时用托管模式，额度用尽切 BYOK（Bring Your Own Key，自带 API 密钥）。

记忆服务侧有三个独立的 LLM（Large Language Model，大语言模型）角色（`Memory/src/config/index.ts:294-304`）：summary（记忆摘要，默认 512 token 上限）、evolution（记忆演化，见"更新与遗忘"节）、embedding。嵌入默认跑本地模型 `Xenova/all-MiniLM-L6-v2`（`Memory/src/model/embedder.ts:56`），用 transformers.js 在 CPU 上以 q8 量化执行（`Memory/src/model/embedder.ts:388`）——开箱不联网、不花钱，代价是中文效果弱于商用嵌入模型；也可配 OpenAI 兼容/Gemini/Cohere/Voyage/Mistral 任一云端嵌入。

BYOK 记忆加工的模型用量有配额上限：按百万 token 计，默认日限 10M、总限 500M（`Memory/src/config/index.ts:380-383`），触发后停掉记忆演化类后台任务、检索不受影响（`Memory/AgentSourceCore/src/memory-token-budget.ts:164-171`）。

## 记忆写入

三条入口最终汇到同一个写入层：

1. **hook 实时提交**：给各工具装的 hook（见"扩展机制"节）在轮次结束时用与扫描相同的解析代码重读磁盘，经 `POST /api/v1/source-turns/complete` 提交（`Memory/src/agent-source/integration/workspace-bridge/runtime.ts:277`）。
2. **离线扫描**：`POST /api/v1/agent-sources/scan` 触发，流水线分三相——stage（流式解析，100 条或 8 MiB 一批写进 per-job 的 SQLite 暂存库，可断点续跑）、prepare（按轮次切分、按上次处理位置增量选轮）、ingest（逐轮提交）。摘要和嵌入不在这里做，是延迟的后台任务（`Memory/src/agent-source/runtime.ts:417-441`）。初次全量扫描限制 1000 轮、每源 200 轮（`runtime.ts:81,433`）。
3. **手动/工具添加**：`POST /api/v1/memory/add` 或 `memmy-memory add`。

入库顺序是先存原始轮次（`raw_turns` 表），再逐条生成 L1 trace 记忆；之后走处理状态机：摘要待定 → 摘要中 → 嵌入待定 → 就绪/失败，失败可重试（`Memory/src/storage/schema.ts:679-684`）。用户偏好类记忆（user_memory）在无 LLM 时用中英文正则规则抽取（`Memory/src/service/user-memory/user-memory.ts:5-17`），配置好 LLM 后交给后台观察提取。

## 记忆组织与存储

单机 SQLite 一个文件（默认 `~/.memmy/memory-service/memory.sqlite`，`Memory/src/storage/db.ts:185-190`），better-sqlite3 驱动加 sqlite-vec 扩展（SQLite 的向量检索扩展，让 SQLite 表能按向量相似度查询），WAL 模式（写前日志，SQLite 支持读写并发的日志方案），schema 版本 15、三十四张表（`Memory/src/storage/schema.ts:8`）。

一条记忆是 `memories` 表的一行（`schema.ts:29-52`）：类型（LongTermMemory 等）、状态（activated/resolving/archived/deleted）、层级 `memory_layer`（L1/L2/L3/Skill 四层，`Memory/src/contracts/memory-runtime.ts:29`）、`memory_key`/`memory_value`、tags/info/properties 三个 JSON 字段、`content_hash`、版本号、软删除时间。用户偏好单独有 `user_memories` 表带 FTS5 全文索引（SQLite 内置的全文检索扩展，`schema.ts:110-150`）。向量不内联在记忆行里，而是按维度分表：`memory_vec_{维度}` 的 vec0 虚表（`Memory/src/storage/sqlite-vec-store.ts:379-403`）——不同嵌入模型维度不同，换模型不用迁移旧表。

四层各有分工：L1 是收割来的原始轮次 trace；L2 是从 L1 归纳出的策略候选（有 30 天 TTL（time-to-live，到期作废）的候选池表）；L3 是跨会话的"世界模型"（对用户和项目的长期理解，独立成组表）；Skill 层存从各工具 home 目录顺带收割的技能文档（扫描时顺带读各 agent 的 `skills/**/SKILL.md`，按内容哈希幂等入库，`Memory/src/agent-source/runtime.ts:1482-1495`）。这条"原始层 → 归纳层 → 抽象层"的演化链由后台任务驱动（episode 合并、奖励回传、策略归纳、世界模型抽象），是整个服务里最重的一套机制。

## 记忆检索与注入

检索是一条可降级的多通道管道。入口 `POST /api/v1/memory/search`（`Memory/src/server/http.ts:87`），候选生成四路并行：向量（sqlite-vec 的 KNN——K 近邻，取向量最近的候选，候选窗按该层配额的 4 倍圈定、窗口内滚动补齐）、FTS5（SQLite 的全文检索扩展）全文、LIKE 模式、结构化片段（`Memory/src/service/retrieval/indexed-candidate-pool.ts:232-259`）。合并用 RRF（Reciprocal Rank Fusion，倒数排名融合——多路检索结果按"排名的倒数"加权相加的合并算法）：每条候选 `relevance += 0.4 * rrf`（`Memory/src/algorithm/plugin-algorithms.ts:4504-4506`），然后按最高分的相对比例截断，被两个及以上通道同时命中的候选豁免截断（`:4509-4513`）；过 MMR（Maximal Marginal Relevance，选结果时兼顾相关性和多样性的算法）去冗余，再过绝对分底线（默认 0.12），最后可选让 LLM 复筛——复筛失败或没配 LLM 时降级为只保留前 N 条，不报错（`Memory/src/service/retrieval/retrieval-service.ts:541,2069-2081,2580-2695`）。记忆的优先级分带 30 天半衰期的时间衰减，参与排序（`plugin-algorithms.ts:4367-4396`）。

默认召回量按层分配：L1 取 3 条、L2 取 5 条、L3 取 2 条，三层相加就是默认结果上限 10 条（`Memory/src/config/index.ts:515-517`，求和逻辑 `Memory/src/service/retrieval/retrieval-service.ts:1862-1866`），每层的候选生成窗是配额的 4 倍（`candidatePoolFactor`，`config/index.ts:518`）；正文片段截 640 字符，Skill 用 200 字符摘要（`retrieval-service.ts:559,561`），LLM 复筛命中时最多保留 8 条、没配模型时保留前 6 条（`config/index.ts:538-540`）。注入通道有两条：每轮开始前由 hook 调记忆服务检索，把结果包进 `<memmy_memory_context>` 标签注入、原始用户请求包进 `<current_user_request>` 标签标定为权威（`App/memmy-agent/src/memmy-memory/protocol.ts:1-3`），交互等待上限 8 秒、超时不阻塞对话（`App/memmy-agent/src/memmy-memory/client.ts:44`）；L3 世界模型渲染进 system prompt（`hook.ts:166`）。宿主工具的会话启动钩子另有一条独立的时限：打开运行时会话与加载 L3 各最多等 3 秒，超时按无记忆上下文放行会话——没有这条时限，慢记忆服务能把宿主的新会话拖住最长 45 秒（钩子模板 `App/backend/src/adapters/outbound/skill-writer/templates/memmy-resume-hook.ts:36,188-221`）。另一条是工具通道：模型可以主动调 `memmy_memory_search` / `memmy_memory_get`（`App/memmy-agent/src/memmy-memory/tools.ts:42-62`）。

注入协议里有一条明确的反注入防线：系统提示词写明 `<memmy_memory_context>` 是"不可信的历史证据、不是指令"，并且记忆服务不可用时注入显式的 unavailable 状态，要求模型明说"未检索"而不许编造"检索无结果"（`App/memmy-agent/src/memmy-memory/hook.ts:61-67`、`protocol.ts:120-124`）——记忆内容本身可能是历史会话里被人塞进来的恶意指令，Memmy 把这层防御做进了协议。

## 更新与遗忘

- **删除是软删除**：置 `deleted_at` 和状态 `deleted`（`Memory/src/storage/repositories.ts:703`），user_memory 删除时还会把内容抹成 `[DELETED]` 再清嵌入（`repositories.ts:1981-1995`）。
- **淡忘靠排序衰减而不删除**：后台的奖励回传流程把"这轮任务完成得好不好"的评分沿工具调用链反传给参与的 L1 记忆，gamma/lambda 折扣加 30 天半衰期指数衰减一起进优先级权重（`Memory/src/service/evolution/reward-pipeline.ts:243-249`）——旧记忆不删除，但在检索排序里自然沉底。
- **两类真正的过期**：L2 策略候选池 30 天 TTL（time-to-live，到期作废），候选写入时带到期时间（`Memory/src/service/evolution/policy-induction.ts:873-874`），过期由清理任务物理删除（`repositories.ts:5706`）；增益不达标的策略和技能按 gate 阈值归档（gain 低于 -0.05 归档，`Memory/src/config/index.ts:465`）。
- **动态知识要复核**：标记为"易变"的策略写入时带 `revalidate_after` 时间（默认 30 天后待复核，`policy-induction.ts:310-311`），到期后不再进入检索下游、等重新确认（`plugin-algorithms.ts:3378-3389`）。
- **更新带版本号**：修改记忆 `version` 自增，变更写 `memory_change_log` 与审计表（`repositories.ts:680,3889`）。

没有定时全库清理，正式记忆一旦激活就长期保留——"遗忘"主要由检索排序完成，这是与 mem0 相同的取舍。

## 隔离与多租户

单机场景的隔离是四个维度的组合：`user_id`（默认 `local-user`）、`source`（调用方工具，默认 `unknown`）、`profileId`（工具内的档案）、`projectId`（由 `sha256(userId + 主机 + 工作区路径)` 从工作区派生，`Memory/src/service/namespace/workspace-identity.ts:38-47`）。L1 记忆按项目过滤——在项目 A 会话里检索不会带回项目 B 的轮次（`indexed-candidate-pool.ts:100-107`）。跨用户访问直接拒绝：user_id 不匹配抛 forbidden（`Memory/src/service/memory-service.ts:4014`）。HTTP 层可用 `x-memmy-user-id` 等请求头把请求绑定到固定命名空间（`Memory/src/server/http.ts:1553`）。

## 持久化与恢复

全部状态在本机文件：记忆库 `~/.memmy/memory-service/memory.sqlite`（WAL），桌面应用状态 `app.sqlite`，扫描暂存放在 `agent-source-scans/{jobId}.sqlite`——任务成功后删除，失败或暂停时保留，下次可复用未完成的 jobId 续跑（`Memory/src/agent-source/runtime.ts:178,552-554`）。记忆处理链路的恢复靠三张表：`memory_processing_state`（每条记忆的摘要/嵌入进度，失败可重试）、`embedding_retry_queue`（嵌入失败重试队列）、`token_usage_outbox`（token 用量先记录后上报的发件箱，`schema.ts:651-741`）。服务单实例运行，靠锁文件防止双开（`Memory/src/server/index.ts:325-336`）。Linux 下以 systemd 用户服务常驻（`memmy-memory.service` + `memmy-gateway.service`），登录自启、终端退出不停（`README.zh-CN.md:97-106`）。

## 安全与隐私

Memmy 存的是所有工具的会话原文，安全设计围绕"本地明文 SQLite 但外泄面尽量小"：

- **收割入口脱敏**：每个适配器在解析瞬间就跑 `redactSecrets`——PEM 私钥块、Bearer 头、Anthropic/OpenAI/Google 的 API key、password 赋值六条精确正则，加一条保底规则：任意 32 字符以上带边界的 base64 连串一律打码（`Memory/AgentSourceCore/src/secret-redactor.ts:11-38`）。去重用的哈希也基于脱敏后文本，保证判定一致（`Memory/src/utils/memory-capture-claim.ts:11-16`）。设计原则写在采集层 README 里："Redact at the reader boundary"（`Memory/AgentSourceCore/README.md:126`）。
- **日志二次抹除**：日志层把 prompt、messages、content、api_key 这类键名全部视为敏感，值直接替换为 `[redacted]`（`Memory/src/logging/logger.ts:23-43`）。
- **网络边界收紧**：记忆服务硬性拒绝非 loopback 绑定地址（`Memory/src/server/index.ts:453-454`）；鉴权三态——配置了本地 token 则必须携带（Bearer/`x-api-key`/`?token=` 都收，`http.ts:1544-1553`），无 token 且未显式允许时拒绝，云 token 和范围 API key 可绑定固定命名空间；内置 Web 面板额外校验同源和 `sec-fetch-site`（`viewer-api.ts:103-106`）。
- **遥测默认静默**：分析事件是自建的 GA4 Measurement Protocol 上报，目标地址由环境变量 `MEMMY_CLOUD_SERVICE` 决定——不设置就整体不发（`Memory/src/cli/analytics.ts:68,527`）；也可 `MEMMY_CLI_ANALYTICS_SKIP=1` 显式关闭。事件里的会话/轮次 id 只上报 sha256 前 16 位哈希（`Memory/src/server/memory-add-analytics.ts:51-54`）。六个记忆项目里，这是第二个默认不外传的（另一个是 MemOS）。
- **给 agent 的 skill 明令**：`memmy-memory` 的 SKILL.md 写明不得存储密钥、token、私钥、密码（`Memory/src/cli/skills/memmy-memory/SKILL.md:44-48`）。

云侧（账号、赠送 Token、托管模型 `memmy_account`）与本地记忆是分开的：扫描和入库全程在本机，云端知识库（Knowledge）是独立可选项。

## 扩展机制

- **HTTP API**：记忆服务的完整 REST 面——会话/轮次、检索、增删、管理（导出、重载配置）、面板（`Memory/src/server/http.ts:73-106`）。
- **memmy-memory CLI**：`init/install/health/session/turn/search/add/get/delete/raw` 十个数据与生命周期命令，全局可带 `--url/--token/--user-id/--source`；支持的 agent 清单：codex、cursor、claude、opencode、openclaw、hermes、dsh、workbuddy、pi、qwenwork（`Memory/src/cli/commands.ts:661`）。
- **OpenAI 兼容 API**：`memmy serve` 在 18990 端口起一个标准 Chat Completions 服务（`App/memmy-agent/src/entrypoints/openai-like-api/server.ts:579`，端口默认值 `App/memmy-agent/src/config/schema.ts:1057`）——任何能调 OpenAI 接口的 agent 都能把"带 Memmy 记忆的 agent"当成一个模型用，这是比 hook 更普适的接入面。
- **MCP**：Model Context Protocol（让模型调用外部工具的标准协议）一等公民，支持 stdio（标准输入输出）/SSE（服务器单向推送的事件流）/streamableHttp 三种传输（`App/memmy-agent/src/core/agent-runtime/tools/mcp.ts:8-10`）；原生的 GUI 操控能力就是以 `open_computer_use` 这个 stdio MCP 服务接入的（`Migrations/src/migrations/v1.1.6/0001-add-open-computer-use-mcp.ts:23-24`）。
- **工具插件**：扫描 workspace 附近 node_modules 里带 `memmyAgent.tools` 字段的包自动发现第三方工具（`App/memmy-agent/src/core/agent-runtime/tools/loader.ts:130-152`）。
- **接入安装器**：`memmy-memory init` 往各工具的 home 写 skill 说明和标记块（claude 的 `CLAUDE.md`、codex 的 `AGENTS.md`、hermes 的 `SOUL.md` 等，`Memory/src/cli/skill-writer/index.ts:48-91`）；`install` 再装真正的 hook/plugin——Claude Code 与 Codex 挂 `UserPromptSubmit`/`Stop`/`SessionStart`/`PostCompact`/`SessionEnd` 五个事件，OpenClaw 装的是插件（注册记忆工具加监听会话事件），hermes 装 Python 插件（`Memory/src/agent-source/integration/` 下各 target）。hook 失败只写 stderr、绝不打断宿主工具的会话（`integration/templates/memmy-resume-hook.ts:392-398`）。

## 可观测性与评估

库内观测有三层：结构化日志（级别由 `MEMMY_LOG_LEVEL` 控制，敏感值抹除）、库内事件表（`recall_events` 记每次召回、`api_logs` 记 API 调用、`memory_change_log` 和 `audit_logs` 记增删改，`Memory/src/storage/schema.ts:479-734`）、内置 Web 面板（`/api/v1/panel/overview|analysis` 提供记忆库总览与分析，`http.ts:104-106`）。评估方面仓库没有基准测试套件，同门的 HaluMem（agent 记忆幻觉评测基准）是 MemTensor 另一个仓库。

## Agent Loop

Memmy 自带完整的 agent 循环，这是它与其他六个记忆项目的最大不同——其他六个都是被 agent 调用的库，Memmy 的循环是产品本体（README 的第三张卡片 "Act"）。实现是标准 ReAct（模型交替"思考-行动-观察"的多轮模式）：`AgentRunner.run()` 逐轮执行"调模型 → 有工具调用就执行并把结果追加回消息 → 继续，没有就定稿"，默认上限 200 轮（`App/memmy-agent/src/core/agent-runtime/runner.ts:249`）；循环耗尽后用专门的定稿提示词再请求一次收尾（`runner.ts:2173-2177`）。配套机制：空回复重试、超长恢复、微压缩（保留最近 10 条，`runner.ts:66-84,1461-1465`）、注入消息排空；单轮走八态状态机（RESTORE→COMPACT→COMMAND→BUILD→RUN→SAVE→RESPOND→DONE，`loop.ts:260-269`）；跨会话的"继续未完成任务"由 GoalRuntime 承载，目标带自己的 token 上限（`App/memmy-agent/src/core/session/goal-state.ts:23-24`）。工具约 52 个类：文件读写补丁、shell、搜索、网页、浏览器控制（21 个）、屏幕观察、目标管理、定时任务、子 agent 派生、图像生成等（`App/memmy-agent/src/core/agent-runtime/tools/loader.ts:52-86`，浏览器类在 `tools/browser.ts:2113-2135`）。README 致谢里明确说 agent 循环与 MCP 集成的工程实践参考了 nanobot（`README.zh-CN.md` 致谢节）。

记忆与循环的耦合是松的：记忆注入发生在循环开始前（hook 的 beforeRun 召回，服务不可用时只注入"未检索"提示，`hook.ts:354-358`），回写发生在循环结束后（afterRun 的 `turn.complete` 异步执行，带 5xx/429 重试，`hook.ts:441-455`）——循环体内没有记忆操作，模型只在需要时主动调记忆工具。

## 设计原则

- **不改应用，读原生数据**。接入成本被转移到"适配器解析原生格式"这一处，接入的 agent 数量可以无限扩展而应用侧零改动。代价是脆弱：agent 一改本地格式，适配器就要跟着改（OpenCode 已经要同时兼容 v1/v2 两代表结构）。
- **完成证据驱动切分，不猜测**。轮次边界、完成状态都从原生字段取证，取不到证就挂起等下一轮扫描，最多等 24 小时——宁可迟到，不收半成品（`Memory/AgentSourceCore/src/source-turn.ts:82`）。
- **同一事实只收一次**。确定性身份加账本裁决的 exactly-once（恰好一次）语义，让“hook 实时 + 扫描保底”的双通道可以放心并行。
- **本地优先，外泄面显式限定**。全库在本机 SQLite，服务强制 loopback，遥测默认静默，脱敏在数据进入系统的第一个瞬间生效。
- **记忆分层进化**。L1 原始轮次 → L2 归纳策略 → L3 世界模型 → Skill 技能，每层由后台任务驱动晋升，检索时分层配额——与 MemOS 的类型化记忆同思路，但 Memmy 的晋升是数据驱动（奖励回传、增益 gate），MemOS 是调度器驱动。
- **把注入防御做进协议**。`<current_user_request>` 权威、`<memmy_memory_context>` 不可信、服务不可用时明说未检索——对抗提示词注入的防线放在记忆通道本身。

## 可复用的做法

- **确定性 ID + 账本裁决实现 exactly-once**（`session-turn-service.ts:1537,1582-1634`）：任何"同一数据可能从多条通道到达"的系统（多端同步、事件重复投递）都适用——身份从不可变的原生字段重建，裁决只看"见过没有、内容一样吗"，不用时间窗和相似度这类模糊判据。
- **完成证据字段**（`Memory/AgentSourceCore/README.md:24-26`）：处理外部系统的流式记录时，显式建模"什么算一条完整记录"，取不到证据就挂起——比按时间戳猜测可靠得多。
- **reader 边界脱敏加熵保底**（`secret-redactor.ts:40-61`）：精确正则覆盖已知格式、任意长 base64 连串接住未知格式，而且去重哈希基于脱敏后文本——数据一进入系统就干净，之后所有环节不必再担心。
- **可降级的检索管道**（`plugin-algorithms.ts:4504-4513`）：四通道召回、RRF 融合、多通道豁免截断、MMR 去冗余、LLM 复筛失败降级为数量上限——每一步都给失败留了默认行为，没有 LLM 也能走完全链路。
- **把服务不可用做成显式协议**（`protocol.ts:39-48`）：依赖服务失败时，给模型的不是空结果而是"没检索过"的状态标注——防止模型把"依赖不可用"叙述成"查过了没有"。
- **per-job 暂存库断点续跑**（`runtime.ts:83,178,552-554`）：长时间批量任务把中间状态落到独立 SQLite，成功即删、失败保留，配合事件循环时间片上限避免批处理饿死在线服务。

## 修订记录

- 2026-11-03 初版：基于 commit `265f6d7` 完整成文。覆盖七工具采集层与 SourceTurn 完成证据机制、双通道恰好一次写入、L1/L2/L3/Skill 四层存储与演化链、四通道 RRF 检索与反注入协议、奖励衰减与两类过期、loopback 加脱敏加默认静默遥测的安全边界、自带 ReAct 循环与 OpenAI 兼容 API 扩展面。
- 2026-11-16 基于 commit `e640129a`（2026-11-12，v1.2.6）补充：episode 归属延迟到后台任务（原始轮次先写入、`episode_assignment`/`episode_finalize` 两个任务带 generation 号去重排队，账本应答新增 `assignmentStatus`）与回合评判 turn.judge.v1 默认开启（每轮一次模型调用判任务关系、反馈情绪与 L1/用户偏好取舍），补入主轴节；采集层从仓库顶层挪进 `Memory/`（`AgentSourceCore` 包名不变，新增桌面端只读共享包 `AgentSourceReaders`），新增顶层 `plugins/office` 文档插件目录；schema 版本 9→15（仍是三十四张表）；向量候选窗从固定 2000 改为按层配额的 4 倍、最终上限改为三层配额求和 10 条；LLM 复筛上限显式化为保留 8 条、无模型时 6 条。全篇行号引用按新代码核对更新。
- 2026-11-20 基于 commit `dfb43b613`（2026-11-19，v1.2.7）补充：宿主工具的会话启动钩子新增 3 秒生命周期时限——打开运行时会话与加载 L3 各最多等 3 秒、超时按无记忆上下文放行会话，防止慢记忆服务拖住宿主会话启动（提交 8964e2b91），补入"记忆检索与注入"节。v1.2.7 窗口的桌面端协作功能（共享项目、成员协助、Office 插件）与记忆系统无关，不写入。
