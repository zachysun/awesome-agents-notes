# Pi

| 源仓库 | [earendil-works/pi](https://github.com/earendil-works/pi) |
|---|---|
| 协议 | MIT |
| 主语言 | TypeScript |
| 笔记基准 | commit `73f8de80bc`（2026-10-20） |

Pi 是 earendil-works 开源的 AI agent 工具包，用 TypeScript 写成，以 monorepo（一个仓库里放多个软件包）的形式组织在 `packages/` 下的 11 个包里（2026-10-13 起；此前一度到过 14 个，实验性的 server/protocol/client 三包已整体删除），GitHub 上 star 数很高（旧笔记记录为 10.8 万）。它最出名的产品是 `pi` 这个终端编码 agent（在命令行里跟你对话、帮你读写代码的 AI 助手，同类产品有 codex、opencode、Claude Code），但它的野心不止于此：官方定位是 "agent harness"（套件/外壳，意思是"给你一套搭 agent 的现成骨架"），从统一大模型接口、agent 运行时、终端界面库，到实验性的持久化运行时和远程协议，一层层都能单独拿出来用。它还有一个标志性的产品理念：文档读不懂？你可以直接问 Pi 让它解释自己。这份笔记基于 commit `c7cdb46`（2026-09-21 克隆），把 12 个包逐个拆开看一遍。

## 它解决什么问题

做一个编码 agent，绕不开四件事：接模型、跑工具循环、做终端界面、存会话。市面上每个产品都自己造一遍轮子，而且造法互相不兼容。Pi 的解法是把这四件事拆成独立的库，每一层都可以单独使用：

- 只想要"一套代码调所有大模型"？用 `pi-ai`。
- 想要"能调工具、能出流式事件的 agent 引擎"？用 `pi-agent-core`。
- 想要"终端里的 AI 编程助手"？装 `pi-coding-agent`，也就是 `pi` 命令。
- 想自己写终端界面？用 `pi-tui`。

同时它对"agent 该长什么样"有一套明确的态度：核心保持极简，别人家内置的功能（子 agent、计划模式、权限弹窗、MCP 支持）它一概不做，改用扩展机制让你自己搭，或者"让 pi 自己给自己写扩展"。这份取舍是否合理见仁见智，但它的好处很具体：核心代码少、行为可预期、想改哪里都有明确的挂载点。

## 整体架构：12 个包怎么分工

`packages/` 下的 11 个包可以分成四组：

| 包 | 一句话职责 | 成熟度 |
|---|---|---|
| `ai` (pi-ai) | 统一多厂商大模型 API | 核心，已发布 |
| `agent` (pi-agent-core) | agent 循环与工具调用（2026-10 瘦身后只剩 Agent、agent 循环、代理流和类型，会话与存储已移入 durable） | 核心，已发布 |
| `coding-agent` (pi-coding-agent) | `pi` 命令本身：交互式编码 agent CLI | 核心，已发布 |
| `tui` (pi-tui) | 终端 UI 库，特色是差分渲染 | 核心，已发布 |
| `telemetry` (pi-telemetry) | 与厂商无关的可观测性契约 | 已发布 |
| `durable` (pi-durable) | 持久化的会话/任务/文档运行时（内部代号 Pico） | 演进中 |
| `env` (pi-env) | SSH 远程执行环境：另一台机器上的 durable ExecutionEnv，配一个 Rust 守护进程（2026-10-05 引入） | 新独立 |
| `chord` | 应用组合运行时（2026-10 中旬被 chord2 宿主运行时整体替换，见下文） | 独立项目，不依赖任何 Pi 包 |
| `codemode` | 沙箱化的 JavaScript 执行环境，脚本唯一的能力是调用注入进来的工具 | 服务于 coding-agent |
| `mcp` | 独立的 MCP 客户端包，pi 和其他应用都能用 | 服务于 coding-agent |
| `evals` | Pi 编码 agent 的行为评测 | 内部工具 |

这张表比 README 官方的"All Packages"表多了 4 个：官方表格只列已发布的 7 个（chord、telemetry、ai、durable、agent、coding-agent、tui），evals、codemode、mcp、env 这 4 个要么还在实验期、要么属于内部工具，但都在同一个仓库里演进。原来的第 12 个包 `session-backends`（SQLite 会话存储后端）已随 agent-core 瘦身整个删除；实验性的 `server`、`protocol`（CBOR 编解码）、`client` 三包连同 coding-agent 的实验性服务栈在 2026-10-13 整体删除（见"server、protocol、client"一节）。另外 README 提到 Slack/聊天自动化和 workflow 场景由姊妹仓库 earendil-works/pi-chat 承接，不在这 11 个包里。

### 分层与依赖方向

依赖是单向的，下层不知道上层的存在：

```text
coding-agent ──→ agent-core ──→ ai
     │                             ↑
     └──→ tui                      │
                                   │
durable ──→ (引用 chord 的类型与 delta、ai 的类型)
server ──→ agent-core + protocol + chord
client ──→ protocol + chord
chord ──→ （不依赖任何 Pi 包）
telemetry ──→ （被 ai / agent-core 引用，自己不依赖它们）
```

几个值得留意的边界：chord 明确声明"不是 Pi 包"，可以给任何应用用；protocol 只负责消息封装和字节流，不懂业务语义；agent-core 在 2026-10-01 做了一次大瘦身——实验性 harness、会话与会话存储、pico3、harness 工具、压缩、技能、提示词模板、遥测 schema 全部移除，`session-backends` 包也整个删掉，包里只剩 `Agent`、agent 循环、代理流（proxy stream）和它们的类型，durable 会话统一住在 pi-durable 里（原生依赖指需要编译本地 C 代码的库，装起来麻烦还常出兼容问题，此前把 SQLite 后端单独分包就是为了不让 agent-core 拖上这些，如今这个顾虑随存储一并移交 durable）。这种"每个包各管一摊、依赖只朝一个方向流"的布局，跟 opencode 的 monorepo 思路一致，是大型 TypeScript 项目的常见章法。

## 核心机制

### pi-ai：统一 LLM API 的抽象方式

LLM（Large Language Model，大语言模型）就是 ChatGPT、Claude、Gemini 背后的模型。麻烦在于每家的调用接口都不一样：请求格式不同、流式返回的事件不同、参数名不同。pi-ai 的解法是三层抽象：

1. **API 实现（wire protocol）**：最底层，每一种"线上协议"一个模块，共 9 个：`anthropic-messages`、`openai-completions`、`openai-responses`、`openai-codex-responses`、`azure-openai-responses`、`google-generative-ai`、`google-vertex`、`mistral-conversations`、`bedrock-converse-stream`。每个模块只导出 `stream` 和 `streamSimple` 两个函数，把统一格式翻译成该协议的实际请求。
2. **Provider（提供商）**：运行时单元，拥有三样东西：模型目录（有哪些模型、上下文窗口多大、支不支持图片和推理）、认证方式（API key 怎么解析、OAuth 怎么登录续期）、以及它用的是哪个 API 实现。关键设计是"模型按协议归类"：xAI、Groq、Cerebras、OpenRouter 这些厂商的接口都兼容 OpenAI 协议，所以它们共用 `openai-completions` 实现，不用每个厂商写一遍。
3. **Models 集合**：注册多个 provider，按 `(provider, model id)` 路由每次请求。查询是同步的（返回最近已知列表），刷新是显式的异步动作（`models.refresh()`），适合 llama.cpp 这类动态列模型的本地服务器。

这个结构就像万能转换插头：你手上是各国插头（各家 API），墙上是一个统一插座（pi-ai 的 `Context`/`Tool`/事件流格式），中间每种插座标准配一个转换头（API 实现）。换模型改一行配置，业务代码不动。

几个具体的设计点：

- **compat 兼容标志位**。"兼容 OpenAI 协议"只是大体兼容，细节千差万别：有的服务器不认识 `developer` 角色，有的不支持 `reasoning_effort` 参数，思考模式的参数格式有十来种写法（`thinkingFormat` 一项就列出 openai、deepseek、qwen、zai 等十余种）。pi-ai 把这些差异收敛成一个 `compat` 标志对象，对已知厂商按 baseUrl 自动检测，未知端点允许手工覆盖。这是"统一接口"落地的真实成本所在，也是这个库最有价值的经验沉淀。
- **统一的流式事件**。所有厂商的流式返回都归一到一条事件序列：`start → text_delta / thinking_delta / toolcall_delta… → done`（出错则以 `error` 结尾）。工具调用的参数在流式过程中边到边解析（partial JSON），UI 可以在参数没到齐时就显示"正在写哪个文件"。
- **模型目录是生成物**。`models.generated.ts` 由脚本从各家上游拉数据生成，人不能手改；要改就改生成脚本再重新生成。目录里带价格、上下文窗口、能力标记，供成本统计和能力判断用。
- **认证解析有优先级**：显式传入的 key 最优先，其次是存储的凭证（`CredentialStore`，每厂商一条，带串行化的读改写，防止并发刷新 OAuth token 出错），最后才是环境变量。存储的凭证"拥有"它的厂商：刷新失败不会悄悄退回环境变量里的旧 key。
- **为测试准备的 faux provider**：一个内存里的假厂商，可以编排"先思考、再调工具、再回答"的脚本化响应，让整套 agent 逻辑的测试不花一分钱 API 费。编码 agent 的测试套件就是靠它跑的。
- **消息帧编码器**（`AssistantMessageFrameEncoder`）：把一条流式响应压缩成可持久化的帧序列，配套一个纯函数 reducer 能从帧重建完整消息。这是"流式过程也能存盘、断点能恢复"的基础。
- **思考等级的统一刻度**。各家控制模型"想多久"的参数五花八门，pi-ai 定了七档：`off/minimal/low/medium/high/xhigh/max`，简单的 `completeSimple(reasoning: 'medium')` 一律走这个刻度；模型元数据里的 `thinkingLevelMap` 负责把刻度翻译成各家的真实参数，不支持的档位标 `null`。要精细控制再下沉到各家专属选项（比如 Anthropic 的 `thinkingBudgetTokens`）。
- **约束采样（constrained sampling）**。普通工具调用是"模型生成参数，事后再校验"，约束采样则让服务商在生成时就强制参数符合 schema，从源头杜绝格式错误。`strict: 'prefer'` 表示支持就用、不支持退回普通调用，`strict: 'require'` 表示不支持就报错；OpenAI 还支持更细的语法级约束（用 Lark 文法直接限定生成内容）。
- **图片是独立的一块能力**。图片输入（把图发给视觉模型看）走普通聊天 API；图片生成（让模型画图）是单独的 `ImagesModels` 集合和 `generateImages()` 一次性调用，两者不混用。传图给不支持的模型会被静默忽略，查能力看模型元数据的 `input`/`output` 字段。
- **错误处理约定：流开始后不抛异常**。请求一旦开始生成，失败（包括用户取消）都以 `error` 事件和最终消息的 `stopReason`（`error`/`aborted`）传达，已生成的部分内容照常保留，还能接回上下文继续问。这个约定让调用方不用在"流式消费"和"异常捕获"两套逻辑之间反复横跳。
- **订阅也能当凭证**。除了 API key，Anthropic Claude Pro/Max、ChatGPT Plus/Pro、GitHub Copilot 的订阅都支持 OAuth 登录直接用，Vertex AI 和 Bedrock 则走各自的云凭证链。
- **打包友好**。每个厂商一个子路径导出（`@earendil-works/pi-ai/providers/anthropic`），各家 SDK（Anthropic、OpenAI、Google 的官方包）按需懒加载；这样只用两三个厂商的应用不会被拖着下载全部 SDK。
- **请求头合并有一条明确的顺序链**：provider 认证头 → 模型元数据头 → 调用方显式传入的头 → `transformHeaders` 回调（最后说了算，返回 `null` 可以删掉下层默认值）。这个回调属于 Models 集合而不属于 provider，设计说明里特意解释了为什么：否则调用方得先手动解析一遍认证，请求就重复解析了。调试请求格式还有个 `onPayload` 回调，能直接看到发给厂商的原始载荷。
- **会话 ID 服务于上游提示缓存**。给请求带一个稳定的 `sessionId`，支持的厂商可以做提示缓存亲和（同一会话的路由到同一缓存），token 花费显著下降；`Usage` 结构把 input/output/cacheRead/cacheWrite 和对应费用分开统计，页脚显示的"缓存命中率"就来自这里。
- **会话中途的工具变更可以内联定义**（Anthropic 路线，2026-10-02）。此前 pi 对 Anthropic 的工具变更靠"检测到同名重定义就回退到重发全量工具清单"，后切到官方 inline tools beta：会话中途新增的工具按值定义在 `tool_addition` 块里，顶层工具清单保持固定，同名重定义不再触发全量重发。动机跟上一条是同一个账：工具清单越稳定，请求前缀越稳定，缓存越值钱。
- **思考等级切换不必丢提示词缓存**（OpenAI GPT-6 路线，2026-10 中旬）。同会话内调整思考等级现在保留前缀缓存（PR 10930）；kimi-coding 提供方改走 Chat Completions 协议，NVIDIA 的默认模型换成 Nemotron 3.5 Super VL。
- **请求超限的判定不再靠估算**（2026-10-20，Copilot 路线）。此前判断"请求是否超出厂商允许的体积"靠 token 数启发式估计，而各家的实际上限并不一致，估算经常不准。现在只要请求收到 HTTP 413（请求体超过服务器允许上限时返回的状态码），一律判定为请求体积超限，直接触发上下文压缩后重试，不再依赖估算（提交 73f8de8）。

### pi-agent-core：agent 循环与工具调用

这一层回答的问题是："模型说要调工具，然后呢？"它的核心是 agent 循环（agent loop）：模型回复 → 如果带工具调用就执行工具 → 把结果塞回上下文 → 再问模型 → 重复，直到模型给出不带工具调用的最终回答。

几个关键设计：

- **两种消息类型，一道转换边界**。agent 内部用 `AgentMessage`（可以是标准的 user/assistant/toolResult，也可以是应用自定义类型，比如"通知"），但模型只认识标准三种。每次调模型前，`convertToLlm` 负责过滤和转换；更前面还有可选的 `transformContext` 做裁剪和注入。数据流是：`AgentMessage[] → transformContext → convertToLlm → Message[] → LLM`。这样 UI 专用的消息可以进会话记录但不进模型上下文。
- **事件流驱动 UI**。`agent.prompt()` 会按固定顺序吐事件：`agent_start → turn_start → message_start/update/end（流式增量）→ tool_execution_start/update/end → turn_end → agent_end`。界面只要订阅事件就能画进度，不用自己轮询。
- **工具执行的并行与顺序**。默认 `parallel`：先逐个预检，再并发执行，事件按完成顺序发，但最终写进会话的 toolResult 按模型给定的原始顺序排列（保证上下文可重放）。任何一个工具声明 `executionMode: "sequential"`，整批就退回逐个执行。
- **钩子点**。`beforeToolCall` 在参数校验后、执行前运行，可以拦下调用（"bash 里有 rm -rf，先问用户"）；`afterToolCall` 在执行后运行，可以改写结果。`shouldStopAfterTurn` 允许在回合正常结束后提前收尾，比如"上下文快满了，先压缩再继续"。
- **转向（steering）与跟进（follow-up）两条队列**。agent 干活时你插的话进 steering 队列，当前回合的工具跑完就注入，模型下一回合就能看到（相当于拍肩膀改方向）；follow-up 队列则是"干完这单再说"的排队需求。两者的注入时机不同，默认都是一条一条喂，可配一次性全给。
- **系统提示词是可重放的补丁序列**。这是一个很干净的建模：会话的第一条 system 消息带完整提示词和工具清单，之后的变更（换了模型、加载了技能、增删了工具）都以 system 消息的形式追加，每条只记录差异（`sections` 按名字替换小节，`toolsAdded`/`toolsRemoved` 记增删）。按顺序重放就得到任意时刻的有效提示词和工具集，不需要单独存"当前状态"。换模型、换思考等级同样以条目形式记进会话。
- **低层与高层两套 API**。`agentLoop()` 是纯函数式的低层循环（观察者模式，不等你处理完事件就继续跑）；`Agent` 类是高层封装，消息处理会作为屏障（等处理完才进入工具预检），还管 steering/follow-up 队列。README 明确建议：需要屏障语义就用 `Agent` 类。
- **提前收尾的约定**。工具执行结果、被拦下的调用、`afterToolCall` 改写都可以带 `terminate: true`，意思是"这批工具跑完就不用再问模型了"（比如工具本身的作用就是"任务完成，通知用户"）。只有整批结果都带这个标记才生效，混批照常继续。
- **自定义消息类型用声明合并**。应用想加一种"通知"消息，不需要改库：TypeScript 的 declaration merging 机制允许你给 `AgentMessage` 联合类型补一个成员，再在 `convertToLlm` 里决定它怎么进（或不进）模型上下文。类型扩展和运行时行为在同一处对齐。
- **浏览器场景有代理方案**。浏览器里不能直接放 API key，`streamProxy` 让前端 agent 把模型请求转发给自己的后端，agent 循环本身照常跑在前端。

会话上下文还有一个"权威来源"的设计（2026-09-21 的 `session context boundaries` 特性把它定型；这套机制连同 SessionManager 在 2026-10-01 的 agent-core 瘦身中整体移出，会话状态的权威投影由 durable 的提交模型接管，见下文 durable 一节）。发给模型的请求内容可能来自很多地方：对话历史、系统提示词、工具结果、中途插入的指令。如果各处自己拼，容易出现"各说各话"的不一致。这个特性规定：**发给 provider 的请求上下文，以 SessionManager 的投影（projection，从会话事件推导出的当前状态视图）为唯一权威来源**——"会话现在长什么样"只有一个出处，谁要发请求都从这里取。同时上下文编辑是**追加式**的：改动只能追加在已有内容之后，不能改写或删除已经发出去的内容，保证"发给模型的东西"和"会话记录"永远对得上、随时可重放。

同批特性还给了"回合结束"一个可编程的决策点：`FinishTurn` 回调在一轮完成、所有工具结果都产出之后被调用，它可以返回 `{ action: "continue" }`（继续下一轮）或 `{ action: "end" }`（就此收尾）。原来的提前收尾逻辑藏在循环内部，现在把"这轮结束后是继续还是结束"暴露成宿主程序可以接管的地方。会话延续（continuation）和恢复（recovery）时，排队中的指令调度保持不变，不会因为恢复流程打断。

### pi-coding-agent：编码 agent CLI

这是 `pi` 命令本体，把上面两层加上 TUI、会话管理、扩展系统组装成产品。README 自述是"minimal terminal coding harness"（极简终端编码套件），理念是"让 pi 适应你的工作流，而不是你迁就它"。

**默认配置依然很轻**：给模型四个核心工具 `read`、`write`、`edit`、`bash`（完整内置清单还有 `grep`、`find`、`ls` 和 Windows 的 `powershell`，共 8 个，可用参数增删），加上 AGENTS.md 上下文文件（从全局、父目录、当前目录逐级拼接）。这个"轻"的边界在演进：初版连 MCP（Model Context Protocol，一个连接外部工具的行业标准协议）都不带，理由写在当年的 Philosophy 一节和作者博客里——这些功能"正确做法不止一种"，与其替你选一种，不如留好挂载点让你自己搭，或直接让 pi 给你写一个扩展。到 2026-10 前后态度松动，MCP 成了内置能力：全局 `mcp.json` 注册服务器（`pi mcp add`、`pi mcp list` 管理命令），stdio 或 streamable HTTP 连接，工具和资源直接进模型上下文；OAuth 登录做得很细——凭证按服务器名加网址分开存（同一个网址的不同服务器可以登不同账号）、签名服务器元数据可以手工指定（对付 advertised 错误的服务器）、RFC 9207 的 `iss` 参数校验（防止授权响应冒名别的服务器；10 月上旬又收紧一步：连服务器元数据都没发现的情况下，`iss` 一律拒绝，不给冒名者留模糊地带）、授权升级（step-up）时保留已授予的权限范围。子 agent、计划模式、内置待办清单、后台 shell 仍然不在默认清单里，扩展机制照旧是补齐它们的正路。

**四种运行模式**覆盖不同使用场景：

- 交互模式（默认）：终端里的完整 TUI；
- `-p` 打印模式：答完就退出，可接管道（`cat README.md | pi -p "总结"`）；
- `--mode json`：把所有事件按 JSON 行输出，给程序消费；
- `--mode rpc`：通过 stdin/stdout 的 JSONL 帧跟外部进程通信（RPC 即 Remote Procedure Call，远程过程调用，这里是进程间调用的意思），供非 Node 语言集成；
- 另有 SDK（`createAgentSession`），把整个编码 agent 嵌进你自己的应用。

交互界面从上到下是四段：启动头部（显示快捷键提示、加载了哪些 AGENTS.md、模板、技能、扩展）、消息区（对话、工具调用与结果、通知、扩展 UI）、编辑器（输入区，边框颜色指示当前思考等级）、页脚（工作目录、会话名、token 与缓存的进出量、缓存命中率、花费、上下文占用、当前模型）。页脚的成本统计把助手回复、工具自己汇报的用量、摘要生成的开销都算在内，这是"agent 帮你花钱，账要让你看见"的朴素做法。

**会话是一棵存在 JSONL 里的树**。JSONL 即 JSON Lines，一行一条 JSON 记录。Pi 的会话文件里每条记录有 `id` 和 `parentId`，彼此连成一棵树：

- 好处是**原地分支**：`/tree` 命令打开会话树，选任何一个历史节点从那里继续，等于 git 的"回到某个 commit 另开分支"，但全程在同一个文件里，历史一点不丢。`/fork`、`/clone` 则把某个节点复制成新文件。
- 条目类型分工明确：`message` 是对话消息；`compaction` 是压缩记录；`branch_summary` 是切分支时对被放弃路径的摘要；`custom` 是扩展的私有状态（不进模型上下文）；`custom_message` 是扩展注入的消息（进上下文）；还有 `label`（书签）、`model_change`、`usage` 等元数据条目。
- 重建模型上下文时，从当前叶子（最新位置）往回走到根，取这条路径上的条目，再应用最新的压缩记录。旧版本会话（v1 线性结构）加载时自动迁移到 v3。

**上下文压缩（compaction）**。模型有上下文窗口上限（一次能"看"多少内容），会话太长就装不下了。压缩的做法是让模型把较早的消息总结成一段摘要，保留较新的消息。手动用 `/compact`；自动模式默认开启，两种触发：上下文溢出时（恢复并重试）和接近上限时（提前压缩）。压缩是有损的，但完整历史还在 JSONL 里，随时可以 `/tree` 回去看。

**项目信任（project trust）**。启动时如果发现项目里有 `.pi/` 配置、扩展、技能等"会自动执行的本地资源"，而你没有对这个目录做过决定，pi 会先问一句"信任这个项目吗"。信任了才加载项目级配置和扩展；不信任就跳过。决定存在 `~/.pi/agent/trust.json`。注意它的定位：这只是一个"输入加载守卫"，防止陌生仓库偷偷改你的 pi 配置，不是沙箱，后面设计哲学一节还会回到这点。

**扩展机制是它的灵魂**。扩展就是一个导出默认工厂函数的 TypeScript 文件（用 jiti 加载，免预编译），能做的事覆盖了 agent 的所有缝：

- `pi.registerTool()` 注册自定义工具（或整个替换内置工具）；
- `pi.registerCommand()` 注册 `/命令`，`registerShortcut()` 加快捷键，`registerFlag()` 加 CLI 参数；
- `pi.on(...)` 订阅事件：会话生命周期、工具调用前后、模型切换、压缩等，可以在 `tool_call` 事件里拦下危险命令实现权限门；
- `ctx.ui` 提供选择框、确认框、输入框、通知，甚至 `ctx.ui.custom()` 让你塞一个完整的自定义 TUI 组件（比如让模型提问、用户在结构化表单里作答）；
- `pi.appendEntry()` 把扩展状态写进会话文件，重启后恢复；`registerEntryRenderer()` 控制这些条目怎么显示；
- `pi.registerProvider()` 注册新的模型厂商（工厂函数支持 async，可以先去远端拉模型列表再注册）。

官方例子覆盖了权限门、git 检查点、路径保护（禁止写 `.env`）、自定义压缩、MCP 集成、SSH 远程执行，甚至等结果时玩 Doom。共享方式是 "pi packages"：在 `package.json` 里加一个 `pi` 字段声明扩展、技能、提示词模板、主题的目录，就能通过 `pi install npm:...` 或 `pi install git:...` 分发。技能（skills）遵循 Agent Skills 标准（agentskills.io），本质是"按需加载的说明文档"：一段告诉模型"什么情况下用我、步骤是什么"的 Markdown。

**日常使用的一些设计细节**（README 用一整章 CLI Reference 讲，这里挑有代表性的）：

- 编辑器支持 `@` 模糊搜项目文件、Tab 补全路径、`!command` 直接跑 shell 并把输出交给模型（`!!command` 则只跑不给模型，适合自己想看结果的场景）、Ctrl+G 唤起外部编辑器写长文、直接粘贴或拖入图片。
- 干活时插话有两档：Enter 发 steering 消息（当前工具批次跑完就注入），Alt+Enter 发 follow-up 消息（全部干完才注入）；Escape 中止并找回排队的消息。
- 配置分两层：`~/.pi/agent/settings.json` 全局，`.pi/settings.json` 项目级覆盖全局。系统提示词可用 `.pi/SYSTEM.md` 整体替换、`APPEND_SYSTEM.md` 追加；上下文文件 `AGENTS.md` 支持目录级的 `AGENTS.override.md` 覆盖。
- 会话可导出成 HTML 或 JSONL、可上传成私有 GitHub gist 分享；`/bug` 直接把会话打包报 bug 给开发者（离线时会明确拒绝，这正是当前 commit c7cdb46 修的行为）。
- bash 工具跑的命令会拿到一组会话环境变量（`PI_SESSION_ID`、`PI_PROVIDER`、`PI_MODEL` 等），方便脚本感知自己跑在 pi 里。
- 启动时的联网行为全部可关：`PI_OFFLINE=1` 关掉更新检查、包更新检查和遥测 ping 三件事；安装遥测单独一个开关。
- 除了云厂商，也支持 llama.cpp 路由服务器，本地模型下载、加载、切换都在 `/llama` 和 `/model` 里完成。

**热重载和模型切换值得一提**。`/reload` 可以不退出进程，重新加载键位、扩展、技能、模板、主题和上下文文件，改完配置立即生效；`/model` 随时换模型，还能用 `/scoped-models` 圈一组模型用 Ctrl+P 快速轮换（换模型之所以能随时做，靠的是 pi-ai 那个可序列化的 `Context`：对话历史就是普通 JSON，拿到新模型手里接着用）。`/thinking` 和 Shift+Tab 循环切换思考等级，按 Ctrl+S 能把当前选择存为启动默认。

**虚拟模型（Virtual models，实验性）把"选模型"这件事也交给了扩展**。2026-09-29 引入：扩展用 `pi.registerVirtualModel()` 注册一个目录条目，它自己不连任何模型服务商，每个请求来时才挑选一个物理模型和思考档位。动机是让路由策略可插拔——"计划用强模型、实现用便宜模型""用一个分类器挑模型""上下文变大就换大窗口"这类策略，此前只能靠手动切模型或在请求准备流程里打补丁。选择保持虚拟的形态：`model_change` 条目记录它，恢复会话时还原它，每条响应记录下真正产生它的物理模型和思考档位；路由器的状态存在会话分支上，跟着 `/tree` 和 fork 走，压缩后也不丢。上下文限制、压缩预留额度、图片尺寸都按请求实际路由到的物理模型计算——虚拟名字背后干活的是谁，账就算在谁头上。官方示例 `jev-router.ts` 用 Jev 分类器在 Sol 和 Terra 之间挑计划模型、第一次编辑后把活交给 Luna，一条完整的路由策略几十行扩展代码。

顺带一提 README 的第一行和 extensions.md 的第一行都是同一句话的两个变体："pi can create extensions. Ask it to build one for your use case."（pi 能给自己写扩展，需要什么直接开口。）这就是"问 agent 解释自己、让 agent 扩展自己"理念的具体落点。

### pi-tui：终端界面与差分渲染

pi-tui 是一个极简的终端 UI 框架（TUI，Terminal User Interface，在纯文本终端里画界面的技术），支撑着 `pi` 的整个交互界面。

**核心抽象小到一句话**：组件就是"给我宽度，还你若干行文本"：`render(width): string[]`，可选地再加键盘、鼠标、缓存失效三个方法。所有内置组件（文本、输入框、编辑器、Markdown 渲染、选择列表、加载动画、图片、容器）都实现这一个接口。

**差分渲染（differential rendering）**是它的招牌。大白话：每次刷新时不重画整个屏幕，只重画变化的行。就像老师改作业不重抄一遍全文，只在错的地方改。终端里这个优化尤其重要，因为终端 I/O 慢，全屏重画会闪。主屏渲染器有三种策略：首次全量输出；宽度变了或视口上方变了就清屏重画；否则把光标移到第一处变化，清到行尾，只写变化的行。所有更新都包在"同步输出"控制序列（CSI 2026）里，让终端把一帧当作原子操作一次呈现，彻底不闪。

**两种渲染器，按终端滚动区的归属划分**：

- `TuiMainScreen`（主屏）：内容画进终端的正常缓冲区，滚动条归终端管。好处是历史可以往上翻（终端的 scrollback），坏处是应用控制不了布局。
- `TuiAltScreen`（备用屏，alt screen，终端的另一块画布，退出后恢复原内容，vim、htop 都用它）：整屏视口归应用管，可以做"上面滚动、下面固定"的布局。退出时打印完整逻辑文档，不丢内容。

**布局系统**（源自仓库根目录的 `tui-plan.md` 设计文档，值得单独一读；该文件 2026-10-05 已从仓库移除，设计已落进代码）：`VStack`/`HStack` 纵横堆叠，`ScrollView` 管一个独立滚动区，条目支持 `basis/grow/shrink/min/max`（与 CSS flexbox 概念相通）。内部实现分两棵树：公开的组件树长存有状态；内部的布局树（每个盒子的位置、裁剪区域）**每一帧重建**，画完原子替换。重建布局很便宜，因为昂贵的叶子组件（Markdown、文本、图片）自带渲染缓存，按内容和宽度缓存结果，几何变了也只需重新摆放，不必重新渲染。鼠标事件走"命中测试"（hit testing，从坐标反查点到了哪个组件），滚轮事件先给指针下的滚动区，吃不完的增量向上冒泡（想象嵌套的滚动页面：内层滚到底，外层接着滚）。

**细节打磨到惊人的程度**：支持中文/emoji 等宽字符的正确宽度计算（CJK 即 Chinese-Japanese-Korean，这类字符占两列，普通按字符数算宽度会错位）；`CURSOR_MARKER` 标记让假光标位置精确对齐硬件光标，从而支持输入法（IME）候选框定位；bracketed paste 模式处理大段粘贴（超过 10 行折叠成一个标记）；OSC 8 超链接可点击、OSC 52 剪贴板复制、选区拖拽自动滚屏；Kitty 图形协议内嵌图片（iTerm2 协议因为不能删除已画的图，备用屏下降级为文本占位）；测试用 `VirtualTerminal`（基于 @xterm/headless 的虚拟终端）验证真实 ANSI 输出。

**主题不是一张预设配色表，而是从终端真实颜色推导出来的结果。** 新的默认主题 `system`（2026-09-26）不再自带一套固定颜色，而是查询终端自己的前景色、背景色和 ANSI 调色板（OSC 10/11/4 查询，一轮问完、以 DA1 应答收尾），在这些颜色上构建整套界面配色：色相和相对饱和度取自调色板槽位，亮度按 OKLab 亮度差和 WCAG 2 对比度下限放置（OKLab 是一种感知均匀的色彩空间，让"看起来一样亮"在数值上也一样亮；WCAG 2 是 Web 无障碍标准的文字对比度规范，保证文字在背景上读得清）。终端只报背景色的，用内置色相补齐；什么都不报的，退回 ANSI 索引加淡化的次要文字。启动这件事刻意**不等待**：界面先以灰度跑起来，颜色应答到了再上色——快启动和准配色两头都要。明暗检测也随之改用报告的颜色（弃用旧的 `?996` 查询），终端明暗模式变化时（mode 2031 通知）自动重新查询。配套还有一个 `ThemedText` 组件：启动横幅、资源加载提示这些地方之前把主题颜色"烤"进文本里，颜色应答来得晚的终端（如 Windows Terminal）会一直灰着，现在文本由函数在主题失效时重建，颜色永远跟当前主题一致。

**浮层（overlay）系统**负责对话框、菜单这类"盖在现有内容上"的界面：定位支持锚点、百分比、绝对值三种写法，可设边距和最大尺寸，甚至能按终端大小决定显不显示（小屏自动隐藏）。浮层有自己的焦点管理：浮层关闭后键盘焦点归还给之前的组件，容器组件里嵌输入框时必须把焦点状态传给子输入框，否则中文输入法的候选框会飘到错误位置，README 专门用一节讲这个坑。

**编辑器组件的完成度接近一个小型文本编辑器**：多行编辑、按词跳转和删除、撤销栈和 kill-ring（Emacs 传统：被剪切的文本进一个环形缓冲，可以按顺序依次粘回）、斜杠命令和文件路径两套自动补全、大粘贴折叠。键盘匹配工具 `matchesKey` 支持 Kitty 键盘协议（一种能精确区分按键组合的新式终端协议），自定义快捷键必须走统一的键位表配置，禁止在代码里硬编码按键判断（AGENTS.md 明文规定），保证所有键位用户都能改。调试时设 `PI_TUI_WRITE_LOG` 可以把发往终端的原始 ANSI 流存下来逐帧检查。

### chord：应用组合运行时

chord 是 12 个包里最"独立"的一个：不依赖任何 Pi 包，可以给无关应用用。它解决的问题是：一个功能（比如"显示 agent 正在做什么"）可能要同时活在多个环境里（后台工作进程、终端 UI、远程网页）。怎么把一个插件拆成几瓣，各瓣跑到各自该在的进程里，还能互相调用、同步状态？chord 给的积木是：

- **plugin 与 facet**：plugin 是一个声明"我提供哪些服务、需要哪些服务"的安装单元；facet 是 plugin 的一片，比如同一插件拆出 `worker` 片和 `presentation` 片，分别装进后台和前端。宿主校验完整个依赖图后，先激活提供方、再激活消费方，销毁时按反向依赖顺序拆。
- **service**：带类型的稳定令牌，要么单例要么按键取实例（keyed）。消费者拿到的句柄是稳定的，背后的提供方断线或热替换都不影响调用方。
- **replicated state（复制状态）与 delta 跟踪**：生产方在一个被跟踪的 `state` 代理对象上改数据，调 `publish()`；chord 把改动记成基于路径的操作（operation）批量发出，消费方收到完整不可变值或增量。字符串追加、数组插入这类常见操作都有专门的操作类型以省流量。
- **远程服务边界**：定义了与传输无关的服务调用语法（严格 JSON 的参数、快照、更新），但不规定怎么打包、走什么网络。应用自己选 WebSocket 还是 Unix socket，chord 只管语义。
- **安全加载**：facet 用 esbuild 打包成内容寻址（按内容哈希命名）的独立 `.cjs` 文件，加载时校验 SHA-256 哈希，用 Node 的 `node:vm` 在受控上下文里编译执行，不进 Node 的模块缓存。热重载设计得很细：先加载并验证新版本，成功后逐个原子替换单例服务，整个过程旧服务一直在路由，调用方无感知；失败则丢弃候选版本，旧版本继续服务。
- **Go 风格的 context**：用于取消和调用作用域内的附加数据（权限、遥测），chord 本身不解释这些数据。
- **包管理上同样守供应链纪律**：chord 的打包器声明"从不安装依赖、从不执行包的生命周期脚本"，外部依赖一律外置（externalized），加载时由宿主解析。跟主仓库的 `--ignore-scripts` 原则一脉相承。

**上面这套 facet/service 运行时已被 chord2 整体替换**（2026-10-13，Package 里的 chord 包原地换血）。chord2 原本长在姊妹项目 p2 里，这批提交把它迁进 `packages/chord`：契约（contracts）、对象（objects）、单元（cells）、根（roots）、插件、热重载，连接方式是内存、IPC（进程间通信）和 WebSocket 三种链路；`./node` 放宿主管理器、子进程控制和进程锁，`./context` 与 `./delta` 保持原样，`./bundler` 删除。老的 facet/service 概念停用：复制状态（replicatedState）独立成模块、导出不变，但不再暴露旧服务提供方用的精确操作订阅——可变状态的监听失败一律改为异步上报。依赖从 esbuild 换成 typebox（一个 JSON Schema 构建库），线上格式仍是 JSON。替换落地后还有一串打磨：链路支持端到端流控并限制每条链的调用数（防止慢链路拖垮宿主）、大帧走 deflate 压缩且中继不读内容直接转发、macOS 上进程锁的 socket 路径守住 `sun_path` 长度上限、已关闭链路的连接即刻清理。

chord2 的线上协议随后推进了两个版本（2026-10 中旬，协议 3 与 4，提交 aa9cf15、b45e0eb）。协议 3：`self.state(name, schema, { init })` 允许对象上线前声明初始状态；主体（principal，"这条连接代表谁"的类型化身份）成为类型参数，createHost 的自身、connect 与 route 三类主体必填；远程契约要声明自己挂在哪个宿主（hostNamed/hostFamily），`get` 不再接收位置参数。协议 4：引用在使用前先补齐状态——`await get` 或就绪的 hold 保证其变体的全部 state 成员可用（本地远程一律如此），hold 自动订阅变体状态、单个成员可用 `state: false` 退出订阅，远程绑定键改用规范身份，形状帧不再携带 codeVersion。类型系统还接管了一处运行时约束：`serve(Type, factory, { lifetime: "host" })` 只对"输入没有必填字段的本地类型"编译通过、且不能与 linger 同用，运行时检查降为后备（提交 301617d）。OutputBuffer 与 Progress 两个类型从 durable 挪到 `@earendil-works/chord/output`，包边界跟着依赖方向走。

### durable：持久化运行时 Pico

pi-durable 目前公开的 API 还很小（一个内存存储实现和几个常量），但 `docs/pico-v5.md` 是一份规范性设计文档，值得当"持久化 agent 运行时该怎么设计"的范文读。它的核心规则一句话：**Session 的每次提交原子地写入不可变的条目、完整的任务记录和被跟踪的文档；只有已提交的状态对外可见**。展开成几条硬性不变量：

- 提交是原子的：一次提交里的条目、任务、文档改动要么全可见、要么全不可见；
- 文档更新只在存储提交成功后才发布（外面永远看不到"还没存稳"的状态）；
- 一切可见进度都是持久的，没有"只在内存里晃一下"的发布路径；
- 外部副作用（发请求、跑命令）不放在提交事务里；
- 条目和 ID 一旦提交就不可变、不复用；
- 文档草稿和拿到的可变对象不许逃出事务（先靠文档约定，未来才考虑运行时强制）；
- 提交后如果存储写入失败（比如检查点或存储层出错），这个打开中的 Session 直接判死：什么都不再发布，必须重新打开。宁可让会话显式失败，也不留下"存了一半"的暧昧状态。

几个术语对上号：会话（conversation）是可以 fork 的对话范围；条目（entry）是不可变的记录；任务（task）是挂在会话上的持久状态机（比如"这个子任务跑到哪一步了"）；文档（document）是用 chord 的 delta 操作表达的可变 JSON 状态。这套设计显然是为"agent 跑到一半崩溃或换机器，回来接着跑"的场景准备的，也是 server/protocol 那套远程架构的地基。

复制状态的更新还进一步事务化（transactional）了：对一份共享的可变状态做改动，不再是"手动应用操作、再手动发布"两步，而是包进一个事务回调里——`view.change(ctx, (draft) => { ... })`，在回调里修改草稿（draft），应用操作、更新事件都在这一次提交里完成。**要么全部生效，要么全部不生效**，多进程、多视图之间共享的状态不会看到"改了一半"的中间状态。这个"改草稿、一次提交"的形态，跟上面会话提交的原子性原则是同一条纪律：共享状态的每一次变更，都要有一个清晰的、原子的边界。

**存储后端从内存走向 SQLite。** durable 早期只有一个内存存储实现（session-backends 里另有 SQLite 适配器，但 durable 自己的持久化当时还停留在设计文档）。2026-09-22 补上了 durable 自己的 **SQLite 存储后端**：一个可移植的 SQLite 存储核心加 Node 适配器，带**版本化迁移**（数据库结构升级按版本脚本走，而不是就地乱改）、一致性测试覆盖和存储基准测试——"变快了"这类说法有基准数据作证，不是拍脑袋。配套的**版本化文档存储**进一步把文档模型做实：文档有原子的基础版本（base）、delta 增量、历史读取、回收与索引化查询，会话文档从"设计里的概念"变成有数据库支撑的实际能力。2026-09-24 又补了第二个后端 **JSONL**（按行追加的日志式文件格式，适合小规模、人类可读、可整目录拷贝的部署），配套的 **sidecar 回收**处理两件事：任务文档的清理（retirement，不再需要的任务记录从存储里清掉）和"只留当前版本 base"的回收（旧 base 被增量盖住后不再保留）。回收不是想删就删，而是以**已提交的标记（marker）**授权——只有提交确认过的记录才能被清理，破坏性操作前还先刷一次主文件，保证授权标记不会在清理中途消失；刷写失败就跳过回收，但不会让已经发布的提交失败。durable 的一致性测试套件也**导出**了，外部存储后端可以复用同一套验证，不用自己重新发明检查清单。

**会话层也从"设计"走向"实现"。** 存储后端就位后，2026-09-25 的提交把会话运行时的机制做实：**原子 Session 事务**——一次事务里对条目、任务、文档的改动要么全部提交发布、要么全部不发布，跟文档里"提交是原子的"那条不变量对齐；**检查点（checkpoint）**——会话可以落检查点，为"跑一半崩溃回来接着跑"提供明确的恢复位置；**文档迁移（document migration）**——文档的存储结构演进有迁移路径，不靠就地改旧数据。文档的**重放也做了优化**：重放不再一条条应用操作，而是把一批不可变 delta 批次（batch）一次性应用（chord 的 delta 引擎配合做了批量应用的优化），恢复会话的速度因此不再是"每条操作一次磁盘往返"。chord 一侧的**不可变 delta 跟踪器**也顺势规范化：此前存在多套实现，现在收敛为一套优化的不可变 overlay（叠加在不可变基线上的增量视图）作为唯一实现，草稿位置（draft placement，事务里改动尚未提交的临时数据位）强制走严格 JSON 校验，所有权契约写进文档——"谁拥有哪份数据、谁负责清理"不再是口头约定。

接着补上的是**会话文档的 fork 与类型化 ID**。fork（分叉，从现有会话复制出一条独立演进的副本）现在是一等操作：新建会话（`Tx.createConversation()`）和分叉会话（`Tx.forkConversation()`）拆成两个独立事务操作，fork 时的会话文档复制由存储后端按策略执行——复制是后端的活，调用方不用自己搬数据，配索引化的所有权查询和"保证无副作用"的拒绝处理，fork 不了就明确拒绝，不做一半。ID 系统同时换上**类型化标识**：原来裸的数字 ID 和 `TaskRef` 包装，换成带品牌的数字 ID 类型（branded type，TypeScript 的技巧：给数字打一个编译期标记，"任务 ID"和"会话 ID"虽然都是数字却不能互相赋值，混用直接编译报错），任务 ID 还带结果类型（`TaskId<R>`）、提交序列号单独品牌化；创建会话必须显式声明"无主"还是"归属于某个任务"，且任务的会话成员关系一经创建不可变——所有权从"约定俗成"变成"类型系统盯着"。

**任务运行时把"任务"做成了会话里的一等公民。** 2026-09-29 的提交给出 `defineTask()`：任务定义必须列出完整的阶段（phases）表和 abort 处理器，各阶段的处理器由注册表解析；调度器在会话提交线上用**一个回调**决定任务的每一次状态转换——预留与迁移、依赖等待、每阶段前的优先规则、向替换后的定义交接、直接中止（标记、发信号、等待汇合、重新调 abort）、被阻塞任务在依赖消失后的 orphaned 结算、开放对账、收尾时汇合所有调用但不写结果。任务改状态的方式也被收拢了：事务接口不再暴露"直接改任务"的方法，任务在自己 `runtime.commit()` 回调的返回值里声明下一个状态——状态机的每次转移都有类型、有出处。配套的 Harness 补了 `resume`、`getTask`、`waitForTask`、`abortTask` 和任务感知的 `waitForIdle`（等的是"没有任务在跑"而不是"没有请求在跑"），文档状态与缓冲监视、检查点谓词可见"自基线以来的 delta"也都就位。durable 从设计文档出发，如今存储、会话、文档、任务四层都有了可运行的实现。

**durable 跑通了第一次真实的工具调用。** 设计文档按交付包分期（Package 编号），2026-09-30 一口气落了三个：Package 16 让 durable 会话第一次跑通 coding-agent 的**工具回合**——生成阶段通过位置式系统条目把已注册的活跃工具提供给模型，工具调用被交给并行或串行的 `pi.tool` 任务执行（校验参数、跑前后钩子、按重放策略记录意图、进度发布是有界且净化的），再由 `pi.post-tools` 任务应用工具增删并继续跑；`read`/`bash`/`edit`/`write` 四个内置工具随包就位。Package 17 补上运行的可观测面：`pi.usage` 用量记录（每个模型的助手用量、每个工具的工具用量、会话总计）、结构化会话视图（`Conversation.viewState()`/`watch()`）、收件箱与实验性的事件适配器。Package 18 把**中止**做成有语义的操作：`Conversation.abort()` 撤回排队输入、中止会话拥有的工作，且中止会**沿所有权级联**——被中止、失败、故障或 orphaned（任务等待的目标消失后无人认领地收场）的任务名下的前台任务一起停、它们的排队输入一起撤，而后台任务是级联的边界，不受牵连；工具任务抛异常或重启后无法安全重跑就判 `failed`（错误结果条目照留），并中止它拥有的会话。至此 durable 不只是"能存"，而是能跑工具、能看用量、能干净地停下来的完整运行时了。

同一天落地的 **Package 19 和 Package 20** 把"任务的家族关系"和"上下文的回收"补齐。Package 19（结构化并发，structured concurrency：父任务结束前必须等子任务收尾，任务不能生下来没人管）给任务立了三条规矩。其一，**归属必须显式声明**：创建任务必须说明它属于某个会话还是属于另一个任务，任务树由此有形可查，"谁的中止该传给谁"不再靠猜。其二，**等待是显式动作**：任务要等其他任务，就提交一个 `waiting` 状态并声明等待策略（failFast 是一个失败全体收摊，allSettled 是等全部出结果），完成后用 `runtime.outcomes()` 取回各自结果——原先那种"按先后排序"的 after 依赖被移除。其三，**收尾要等名下清空**：任务想结束时还有活跃的工作在跑，就进 `completing` 状态等它们全部收尾才算完；中止处理器自底向上运行（先停叶子任务再停父任务），级联只沿着活着的所有者走。工具链也在这一包里收拢：generation 现在自己拥有 `pi.tool` 任务，在自己的 tools 阶段等它们跑完、应用工具增删后接着开下一个 generation——上一包里的 `pi.post-tools` 任务被整体删除（`afterTools` 钩子移到 generation 上），工具回合从"两段接力"简化成"一段等待"。Package 20 处理的是老问题"上下文满了怎么办"：`pi.compaction` 任务总结模型上下文中较旧的前缀，产出一条压缩条目放在保留的第一条条目之前当"标题"（旧条目不删，还躺在存储里，随时能翻回去）。触发路径有四条：手动 `compact()`；超过软阈值后**后台**起一个压缩（会话照常干活，摘要好了在空闲时立刻放、否则等下一个回合边界）；超过"窗口减保留额度"后**阻塞**着等压缩完成再发下一个请求；服务商报"上下文超长"时压缩一次并重试一次。多个摘要同时在生成时按 stale 规则裁决：切的位置早于当前上下文起点的摘要判 `stale`（作废），切得最深的那份生效——并发压缩不会互相打架。`beforeCompact` 钩子可以拒绝压缩或自己提供摘要，压缩的开销计入 `pi.usage`。同一天 durable 的 Node SQLite 存储还改成了异步执行（此前同步调用会卡住整个进程），存储调用排队跑、按语句缓存编译结果，事务回调能直接拿到事务句柄。

次日（2026-10-01）连落 **Package 21 到 23，durable 有了自己的扩展系统和自己的前端**。Package 21 把扩展机制做进 durable：具名扩展的注册表（注册工具、提示词小节、钩子、工具与小节的包装器、任务），带安装与卸载；每个会话可以挂一份 `pi.agent` 文档——模型、思考档位、启用哪些扩展、工具过滤、附加指令、工作目录——创建和分叉会话时都能指定 agent，任务名下的会话自动复制主人的 agent，"一个运行时里跑多个不同配置的 agent"从此是数据而不是代码。四个内置工具（read/write/edit/bash）也改成了扩展形态（CodingTools 扩展）：核心只留机制，能力全部走同一条扩展通道。Package 22 补生命周期的收尾纪律：调用结束后 runtime 的取时（`runtime.now()`）和上报（`report()`）直接抛错、失败的 `Harness.open` 会自行关闭并重抛原始错误、只含迁移的提交也照常发布；Package 23 给出 `Harness.taskGraph()` 和 `watchTaskGraph()`——把每个活跃任务连同它的所有者边、已提交状态、拥有的会话挂成一张图，随提交事件推进，任务系统第一次有了整体视图。**前端也随之而来**：`src/experimental/durable` 里跑着一个实验性的 durable TUI 编码 agent，复用 pi 的模型运行时、设置、系统提示词、主题和交互组件，架在 durable Harness 加 SQLite 存储上——流式、工具、插话与跟进、中止、模型与思考档选择、压缩、`--continue` 恢复被打断的回合都能用，还带一个 subagent 工具（`/agents` 面板切换、跟 subagent 对话）和任务图驱动的 `/tasks` 实时面板。实验性 client/server 也移植到 durable：会话工作进程各自托管一个 durable Harness（每会话独立的 SQLite 存储），服务器不碰存储本身，任务图还有活跃任务的工作进程推迟退休、重新打开时恢复被打断的工作。同一天 pi 发布 **v1.0.0**，agent-core 的瘦身（见架构一节）与这条 durable 主线是同一件事的两面：仓库里会话运行时收敛到 durable 一条线，旧路正式拆掉。

**durable 的执行环境有了正式接口**（2026-10-04）。Package 21-23 之后的一批提交把 durable 里"工具干活的环境"（FileSystem 与 Shell）做成了有接口、有边界、有测试的正式抽象：`FileSystem.openBinaryReader()` 提供单文件的位置式读取（noFollow，不跟符号链接走）、`openDirReader()` 分页列目录（中途消失的条目直接跳过）、`Shell.exec()` 支持不经 shell 的 argv 数组执行并在 `onOutput` 里报告输出来自哪条流（标准输出还是标准错误）；`FileSystem.watch()` 提供目录监视——递归目标可带排除、原生或轮询两种模式（网络与 FUSE 文件系统自动退到轮询）、目录数量有上限。"有界"是贯穿这批提交的关键词：读工具经 `BinaryReader.scanLines()` 在环境内一次遍历定位一段行，只解码要展示的头部（图片按文件头和 PNG 块头识别），shell 输出可以只保留尾部窗口、环境报告准确的跳过计数（省传输也不丢信息），进度提交的节流间隔从硬编码 100 毫秒变成可配置。工程信号也明确：环境的**一致性测试套件**进了 `/testing` 导出，Linux、macOS、Windows 三平台的测试流水线跟着建立——外部执行环境想接入 durable，跑同一套测试就知道行为对不对，跟 pi-ai 的 faux provider、telemetry 的适配器一致性测试是同一条方法论。

**环境抽象接着有了远程实现，上下文的读取也省了重复功**（2026-10-05 至 10-07，期间连发 v1.0.3/v1.0.4/v1.1.0 三个版本）。新包 `@earendil-works/pi-env`（见架构一节）把执行环境搬到另一台机器上：`RemoteExecutionEnv` 是一个跑在远端的 durable ExecutionEnv，在那台机器上产生的结果语义与本地 NodeExecutionEnv 一致，靠一个 Rust 写的 pi-env 守护进程支撑（覆盖 Linux、macOS、Android、Windows，遵循 Node 与 libuv 的语义；SSH 引导、懒连接、卡死超时都做了），测试矩阵在三平台同时跑 durable 环境测试、守护进程检查和 pi-env 的一致性与差分测试。durable 这边，`runtime.context()` 开始把每个会话最近读取的范围和派生视图保留在内存里：同一头标记的后续读取只扫描和派生新提交的条目，generation 不再为每次模型请求重读并重派生整份转录；忙的会话一直保留，闲的按 `contextRetentionMs`（默认十分钟）释放，保留的条目冻结、每次读取拿到自己的视图数组。配套还把"读旧状态"做成了正经能力：可以按较早的某个条目读取会话当时的上下文。存储侧新增 Cloudflare **Durable Object SQLite 适配器**（Durable Object 是 Cloudflare 边缘网络上的有状态对象，自带一个 SQLite），`openDurableObjectSqliteStorage()` 让 durable 的存储可以直接跑在 Cloudflare 边缘上，不逐行拷贝。

**工具可以调工具了，存储错误也有了明确纪律**（2026-10-09 起）。durable 里 `ToolExecutionApi.executeTool()` 允许一个工具在执行中把另一个工具当作独立的 pi.tool 子任务跑起来，任务归调用者所有、按键挂接，重启后能安全地重新接上；程序（用脚本编排工具的那一侧）拿到的结果存进调用者自己的嵌套结果文档，直到整个调用结算。`pi.live.nestedTools` 面板能看到每次嵌套调用跑的是什么、参数是什么、结果摘要是什么。配套加了 `TaskOptions.abandonOnRestart`：只在内存里被等待、重启后要重跑的工作，重跑前先放弃。结构化输出也齐了：工具可以声明 `structuredOutputSchema`，产出经校验的 `structuredOutput`；没声明 schema 的工具给程序的输出同样有界（一条文本就给字符串、一张图片就给图片本身、否则给清单）。谁有权调用哪个工具由 `callers` 和 `modelTools` 决定。另一件事是给存储错误立规矩：Storage 方法抛出的任何错误都判会话失败，会话自行关闭，后续调用和等待以 SessionFailed 拒绝、监视以 session_failed 结束——Harness 里**不做任何重试**；此前"把错误咽下去只让单次调用失败"的 StorageRejected 被移除（StorageRequestError 只标记无效读取这类确实只影响单次调用的情况）。听者和上报回调的错误被就地拦下、单独报告，不再波及会话本身。这套纪律的立场很清楚：底层存不下数据，上层就该立刻知道，而不是带着缺损继续跑。

**read 工具能读图片了，调度器占的内存也重新算了**（2026-10-10 前后）。read 遇到图片文件不再报"不支持"，而是返回一个图片块（程序侧拿到 ImageContent）加一条说明；限额取会话模型自己的 `inputLimits.images.resize`（默认 2000x2000 像素、4.5 MB base64）。不带图像处理器时，PNG、JPEG、GIF、WebP 在限额内原样透传不解码（先查字节上限再读文件，尺寸从文件头拿）；配了可选的 Photon 处理器（一个 WASM 图像库）则按 EXIF 信息摆正方向、BMP 转 PNG、缩到限额内。一条有趣的边界：图片必须整体读，读的途中文件变了就整张重读——跟可以边长边读的文本文件不同。模型本身不支持图片输入时，返回"你看到的是占位符"的说明。同期的调度器瘦身把归属索引按"活的工作"定界：此前每条见过的会话归属边、每个已结束任务的归属都记到进程结束；现在只有下方还有活任务时才保留归属链，不再需要的节点排队、在每次发布结束时清扫，另外调度器不再每轮遍历所有活任务。基准很直白：活父任务下 1 万个已结束的 subagent 循环，需要记的归属边从 10001 条降到 1 条。

**执行环境补了终端和文件树两件套**（2026-10-12 前后）。pi-env 守护进程新增 pty、ptyWrite、ptyResize 三个操作：程序可以跑进伪终端（PTY，pseudo-terminal，模拟真实终端让程序以为自己连着一台终端机，vim 这类全屏交互程序因此能用），原始终端输出按事件流出、输入按到达顺序写入、窗口尺寸变化触发 SIGWINCH 信号，取消或连接断开时杀掉整个进程组；Unix 专用，Windows 明确返回不支持。配套的 ptyForeground 能报告终端当前的前台任务。文件侧，ExecutionEnv 新增可选的 listFiles：一次列出某目录下全部文件、目录和符号链接（相对路径、按名字深度优先、跳过 .git 和 git 忽略的文件、带 truncated 标记表示是否截断）；环境没实现或不支持时，调用方退回自己遍历，环境一致性测试套件也加了对应的选项。

durable 这轮还有三笔（2026-10 中旬，v1.3.0 前后）。存储侧支持**命名空间化的 SQLite**（提交 f7e3cb5）：迁移和运行时查询从同一张 schema 表生成，表可加前缀，多个 durable 实例共用一个 SQLite 文件（包括 Cloudflare Durable Object 里的共享存储）而互不串表。压缩配置支持**按模型覆盖**（提交 c4cf795）：`compaction.modelOverrides` 按 provider/modelId 键生效于阈值判断、溢出恢复和压缩范围选择三处——大上下文模型和小模型各用各的压缩策略。接口侧 `HarnessOptions.models` 与设置项接受函数形式，宿主可以按需给出模型清单，而不是一次性写死。

**codemode 的脚本沙箱也多了一个去处：Cloudflare Workers**（2026-10-10）。宿主先拆成"与传输无关的执行逻辑"加一个 VM 驱动，于是 VM 可以跑在 worker 线程（原样保留）、调用者线程（内联模式）、或经 RPC 跑在另一个 isolate（独立运行实例）里。新增的 `/cloudflare` 入口把沙箱做成一个远程 Durable Object，部署成独立的 Worker——脚本永远不占调用者的线程；`/portable` 入口则不带任何 Node 依赖，浏览器和边缘环境都能用。纯计算、不等待工具的脚本受中断额度约束，而且现在在排空 promise 任务队列时也会轮询，"从不让出"的死循环同样能被打断。官方仓库附了一个 cloudflare 示例，agent 和沙箱各一个 Worker。

**durable 有了自己的代码模式，工具也可以先不给模型**（2026-10-12 前后，v1.2.0 前后）。`@earendil-works/pi-durable/codemode` 给 durable 会话配了一个 codemode 工具：脚本以嵌套调用的方式使用会话的工具，而且**重启也能活下去**。脚本确定性运行（pi-codemode 的 `start()`），每个结果在交付前先写入日志（journal，按序持久的事件记录）；重跑时从存好的 start 数据重新启动脚本，按序交付日志里的结果（嵌套调用按键 `c<id>` 重新挂接），核对每次记录的调用次数，日志里没有的调用先扣住、等重放结束后再与此前尝试的键核对，然后才继续活跑。一次调用只有在发起它的那次排空已提交后才真正发出，重跑因此能证明哪些调用是新的。偏离（divergence）、模块版本或沙箱指纹变化、超过期限，都会以存好的错误结束调用；已有定论的结果直接返回、不再启动虚拟机。`store()`/`load()` 按会话存值，可回退、随 fork 走。配套的一串提交把代码模式做完整：结果列出脚本的全部工具调用、关闭沙箱不再决定脚本的结果、工具错误有了统一规则、MCP 与 codemode 解耦、schema 约束直接显示在工具声明里、输出拼接改为线性。另一条线是**延迟加载的工具**：工具注册可以标记 deferred，不主动提供给模型，直到 modelTools 指名；modelTools 支持 `{remove?, add?}`，控制面的 addTools 把名字写进 add，加载的工具按加载顺序追加；Agent.deferred 列出尚未加载的延迟工具。配套的 `@earendil-works/pi-durable/tool-search` 扩展把 coding-agent 的 tool_search（BM25 检索）带进 durable——工具多到不适合全量出现在模型面前时，模型先搜再取。

**durable 迎来“持久执行平台”化的一轮大升级（2026-10-17 至 10-18，v1.3.0 之后的主线）。** 四块能力把这层运行时从“任务状态机”推向完整的持久执行平台。其一，**睡眠成了持久状态**（提交 f75b324）：任务状态机扩为六态——pending / running / waiting / sleeping / completing / terminal（packages/durable/src/types.ts:599-643），新增的 sleeping 结构是 `{ status: "sleeping", checkpoint, until }`（types.ts:622-631）：任务睡到 Harness 时钟到达 `until`，然后从 checkpoint 恢复；睡眠期间不持有任何调用句柄，deadline 存进 SQLite，进程重启后依然有效——此前的 `runtime.sleep` 是进程内概念，进程一死睡眠就丢。中止标记不必等睡眠结束。generation 重试轮询与压缩重试都改成了这种睡眠。存储层跟着迁到 schema version 2（storage/sqlite/migrations.ts:91-115）：SQLite 没法原地改 CHECK 约束，迁移是重建整张任务表把 sleeping 加进状态枚举。其二，**时间可以外包给宿主**（timers.ts，79 行的新文件）：`HarnessOptions.setTimer` 让宿主接管全部定时，`HarnessOptions.onWake` 与 `Harness.wake()` 让宿主自己唤醒进程，`scheduler.resume()`（harness/scheduler.ts:423-427）把到期的 sleeping 任务变回可运行——宿主可以用 Cloudflare Durable Object 的 alarm 接管所有定时。默认实现 `nativeTimer`（timers.ts:17-36）包了 setTimeout，超长延迟分步等待（单次上限约 24.8 天）；有个专门的 Cloudflare Workers 特判（timers.ts:10）：在 Workers 里一个挂着回调的定时器会让 Durable Object 无法休眠、持续计费，所以“只负责释放内存”的后台定时器在 Workers 里直接拒绝。`hostTimers()`（timers.ts:51-79）把宿主的 setTimer 包成两类：foreground 定时器负责调节或恢复工作，宿主抛异常或拒绝时退回 setTimeout 并上报一次；background 定时器宿主拒绝就放弃。其三，**崩溃循环有了上限**（提交 a30c2a8）：此前崩溃恢复是“running 一律翻回 pending 重跑”，带 bug 的任务会每次重启都重跑再崩、无限循环。现在 `Harness.open` 时给每个上次崩溃时正在跑的任务记 `restarts` 计数再翻回 pending（scheduler.ts:394-398），计数达到 `settings.restartLimit`（默认 5，harness/agent.ts:56-60）且未被请求中止的任务停驻为 blocked，上报 `task_parked`，错误消息直说“被切断 N 次没有进展；停驻直到 retryTask()”（scheduler.ts:406-412）；`retry()` 由人显式调用，清零计数恢复调度（scheduler.ts:433-442）。判定留了余地：运行中、持有结果、被请求中止的任务永不判 blocked（scheduler.ts:448-453），每轮调度把停驻任务再查一遍（restartLimit 可能上调），注册表变化或迁移失败也触发重判。上报原因同时类型化：`ReportInfo` 是九类判别联合（types.ts:164-186），`task_faulted` 分 threw/restart_limit、`task_outdated` 分 missing_task/incompatible_task、`host_error` 直接指名出错的宿主回调——此前靠解析 error.cause 字符串，现在是类型化字段。配套还有 `settings.afterFailedRun`（一个任务失败后，排在它后面的输入保留、继续还是撤回）和任务图节点的 blocked 属性（外部观察者能看到谁卡住了、为什么）。其四，**真机验证**（提交 ea7c648）：`packages/durable/test-workers/` 用 Cloudflare 官方运行时 workerd 跑真实的 Durable Object 唤醒测试（wrangler 工具链、wake 测试、fallback alarm 示例）——durable 的目标宿主之一就是 Cloudflare Durable Objects，关键行为从此有了真实环境的验证。同期 c268154 把 durable 的 SQLite 驱动挪进 `chord/sqlite`，与之前的命名空间化 SQLite 一脉相承：chord 层提供驱动，durable 层只调用它。另外 2026-10-19 起 `checkpointWhen` 谓词成为 durable 文档定义的必填项（提交 5c1736f）：`defineDoc`/`defineDocFamily` 不再提供默认值，每个文档必须显式声明“什么时候可以打快照”——快照压实策略从隐式默认变成逐文档的显式决定。

**遥测成了全线贯通的正式能力（2026-10-18 至 10-20）。** 这轮在 `packages/telemetry` 建了一套与厂商无关的可观测性 API，再把 durable、pi-ai 逐层接上。基础件（telemetry/src/index.ts）有三个特点：传播协议是不透明的（`TelemetryCarrier` 就是 `Record<string, string>`，`TelemetryPropagation` 只定义 inject/extract，不绑定 W3C traceparent 之类的具体格式）；span 链接是一等公民（`SpanOptions.links`、`addLink`）；最大的特色是**声明式 schema 加编译期类型检查**——`defineTelemetrySchema`（index.ts:108）声明 span、事件、指标的属性类型/必选性/枚举值，配一整套类型推断工具，属性用错名字或枚举值在编译期就报错。pi-ai 与 durable 的遥测词汇都用它定义。durable 的 `DURABLE_TELEMETRY_SCHEMA`（harness/telemetry.ts:50-139）设计原则写在注释里："IDs address durable records, never content"——遥测只带任务 ID、提交 ID 这类记录标识，绝不携带对话内容。两个 span：`pi.durable.submit`（提交的准入与落库；新输入从自己的根 trace 开始、link 到调用者的 trace）和 `pi.durable.phase`（一次任务阶段调用及其决策提交；描述明确写了“绝不是一次等待或重启”——等待与重启不是执行，不产生 span）。事件五种（entry/submission/report/checkpoint/replay，其中 `pi.durable.replay` 记录“嵌套工具调用直接从存储结果作答”）、指标三个（reports 按九类 ReportInfo 计数、phase.duration 直方图、tasks.active 升降计数器）。trace 上下文直接持久化：任务记录里存 `trace: TelemetryCarrier`（types.ts:648），进程重启后恢复的任务依然留在自己原来的 trace 里，trace 不因崩溃断链；错误诊断截到 500 字符以内。pi-ai 侧（`AI_TELEMETRY_SCHEMA`，packages/ai/src/telemetry.ts）是 span `pi.ai.request` 与 `pi.ai.poll`（后者是“一次延迟响应的轮询，不是延迟请求的一生”）、事件 `pi.ai.first_chunk` 与 `pi.ai.block`（粒度刻意停在“首块”和“块”，绝不按 delta 记，否则一个长回复就是上万条事件）、指标 duration/first_chunk/tokens/cost；接入方式很克制——`observeModelRequest` 经 `lazyStream` 观察者包住请求（models.ts:23），provider 层零改动，`aborted` 保持 status ok。chord/context 的 `Context.telemetry` 从此永远是一个 span（NOOP 实现打底），扩展代码拿到的不是 undefined。随后遥测又有了持久去处（提交 d9025a5）：共享的 `TelemetryStore` 接口（findTraces 按名称/属性/错误/时间窗过滤、游标分页；getTrace；findMetrics），chord 侧落在 `@earendil-works/chord/telemetry-sqlite`——span 带内联事件与链接、指标按序列每 10 秒聚合、1 秒批量刷写带保留策略、有界队列溢出丢弃并计数；`/render` 子路径还配了一个 trace 渲染器，轨迹能直接画出来看。

### server、protocol、client：实验性远程架构

这三个包组成一套实验性的本地服务架构，目标是让"会话活在后台工作进程里，多个界面（终端、网页）同时连接"成为可能：

- **protocol** 定义线上格式：协议版本握手、路由目标（服务器级 `{serverId}`，会话级 `{serverId, sessionId, attachmentId}` 三元组）、请求关联与取消、订阅更新。帧格式是"4 字节大端长度前缀 + 一条 CBOR"（CBOR 是一种二进制 JSON，比文本 JSON 省带宽也更快），单帧上限 16 MiB。消息校验严格（拒绝未知字段、非 JSON 载荷、循环引用），但业务语义完全不碰：会话、模型、插件在协议眼里都是不透明的服务数据。
- **server** 是本地路由器：每个连接建一个 attachment（附加物，一次"接入"的凭证和路由），同一会话可以挂多个界面；路由 ID 只作为控制数据下发，业务结果里不暴露。连接断了只释放自己的 attachment，已受理的调用照常完成。Unix socket（同一台机器上进程间通信的管道）路径与逻辑 serverId 分离，长驻启动器可以换进程不换身份。工作进程（真正跑会话和 agent 的进程）的启停刻意放在公共协议之外：何时"没有界面连接、也没有 agent 在干活、可以停止"由宿主应用决定，协议层只提供稳定的路由和通用的连接变化通知。真实的 Session 对象从不跨进程传输，跨进程的只是服务调用。
- **client** 是传输无关的客户端：你提供"怎么建字节流"（WebSocket、Unix socket 都行），它负责握手、验证对端真的是目标 serverId、按 ID 关联请求。设计上刻意不自动重连、不重放请求，断线后由应用决定哪些操作安全可重做。这是个成熟的态度：网络故障时的"自动帮你重试"经常比失败本身更危险。

这套东西的状态是"实验性、无兼容性承诺"，但分层干净：协议只管消息封装，chord 管服务语义，应用管业务类型，谁也不越界。

（2026-10-13 更新：这条实验线已整体拆除。）`client`、`protocol`（含 CBOR 编解码）、`server` 三个包连同 coding-agent 里的实验性服务栈、CLI 实验入口、示例插件全部删除，coding-agent 不再依赖 chord——本节内容保留为历史记录。删除的动机没有单独成文，从提交的上下文看，durable 已经把"会话活在后台、多端接入"这件事用另一条路做起来了，实验栈的使命结束。

### session-backends、telemetry、evals：三块外围支撑

- **session-backends/sqlite-node**：agent-core 的会话存储后端之一，用 Node 内置的 `node:sqlite` 实现。默认一个会话一个数据库文件，ID 只含安全字符的直接用 `{sessionId}.sqlite` 命名，含特殊字符的做 base64url 编码再加 `~` 前缀，避免文件名出现意外字符；也支持把多个会话放进一个共享容器。open 和删除操作拒绝越出配置目录的路径、绝不顺手创建缺失的数据库；fork 会话时对源库开只读连接加 WAL 快照事务（WAL，Write-Ahead Log，SQLite 的一种写日志方式，允许读写并发），互不阻塞。它刻意不实现跨进程锁，把"同一会话只有一个写入者"的责任交给宿主，并在文档里把这条边界写得明明白白。
- **telemetry**：可观测性（telemetry，记录程序运行期间干了什么以供诊断）的契约包。设计上有三个鲜明选择：遥测上下文**显式传参**而非用全局隐藏状态（所以能在 Node、浏览器、worker 里通用）；span（一段计时的操作记录）是纯诊断数据，记录失败绝不影响业务；每个应用用 `defineTelemetrySchema` 声明自己的 span 名和属性类型表，编译期就能查错。最值得学的是它附带一套**适配器一致性测试套件**（conformance suite）：任何人写 OpenTelemetry/Sentry 适配器，跑同一套测试就知道行为对不对，不用逐条读文档猜语义。
- **evals**：行为评测（eval，evaluation，用真实任务考察 agent 表现而非玩具基准）。评测分两类：`*.eval.ts` 是直接在本机跑的普通行为评测，`*.docs.eval.ts` 是"文档提升评测"（docs lift eval）：把仓库搭成两份 Docker 镜像，一份带文档（README、docs、examples），一份删掉文档并从系统提示词里移除文档路由段落，同一个用例在两种镜像里各跑几遍，配对比较通过率的差值就是"文档到底帮了 agent 多少"。实验隔离做得很认真：容器里只给读写编辑和搜索工具、不给 shell，评测文件和判分器在运行时降权到非特权用户。结果产出也克制而完整：每次运行的实验协议、任务清单、逐臂的原生 Vitest 报告和 pi 会话记录、配对比较报告全部留在本地，事后可复查可复算；配对双方只要有一方缺失、重复或出错，这对就不计入通过率，有被阻塞的配对时干脆不公布总的提升数字（宁可不给数，不给坏数）；报告还会标记无提升、负提升和观察到的抖动，并明说"跑一遍说明不了稳定性"。这套方法回答的是"我写的文档对 agent 有没有用"这个平时只能靠感觉的问题。

## 设计哲学与取舍

### 安全边界放在外圈：无内置权限系统

README 原话：Pi 不包含内置权限系统来限制文件系统、进程、网络或凭证的访问，默认以启动它的那个用户和进程的权限运行。`docs/security.md` 把理由讲得很直白：进程内做一半的沙箱，容易被误解成安全边界，而它其实还依赖宿主 shell、文件系统、包管理器、凭证和扩展代码；真正的隔离只能来自操作系统层或虚拟化/容器边界。所以 Pi 给的是四种现成的容器化模式（`docs/containerization.md`）：

| 模式 | 隔离什么 | 特点 |
|---|---|---|
| Gondolin 扩展 | 只把内置工具和 `!` 命令路由进本地 Linux 微虚拟机（micro-VM，用 QEMU 跑的极小虚拟机，比完整虚拟机轻得多） | pi 本体和模型凭证留在宿主机 |
| 纯 Docker | 整个 pi 进程跑进容器 | 最简单，但 API key 也进了容器 |
| OpenShell | 整个 pi 进程跑进策略控制的沙箱 | 文件、进程、网络、凭证、推理调用都可设策略，网关还能替你注入模型凭证 |
| Docker Sandboxes（sbx） | 整个 pi 进程跑进托管沙箱 | 凭证不进容器：容器里放占位符，出口代理替换成真值 |

这跟其他 agent 的做法形成对照：不少产品把审批弹窗、命令名单做在程序里（内圈设防）。两种哲学各有道理：内置护栏开箱即省心，但代码复杂、边界模糊；外置边界要求使用者有安全意识，但程序保持简单透明、隔离强度高得多。Pi 值得学的地方在于它把选择、理由、替代方案全部写进了文档，包括坦承"来自仓库文件的提示注入（prompt injection，藏在文档或代码注释里诱导模型干坏事的文本）是本地 agent 的预期风险，pi 无法可靠防止"。不设防不是不负责任，把取舍说清楚、把方案给全，才是负责任。

### 核心做减法，扩展做加法

Philosophy 一节列了一串"No"：No MCP、No sub-agents、No permission popups、No plan mode、No built-in to-dos（理由是待办清单会干扰模型）、No background bash（要用 tmux，全程可见可交互）。每一条都附了替代路径：写个扩展、让 pi 给你写、或装社区包。这个产品策略的赌注是：挂载点足够好的话，官方不必亲自实现所有功能。配套工程是实打实的：扩展能注册工具、命令、快捷键、事件处理器、UI 组件，能持久化状态，能自定义渲染，理论上没有"核心做得到而扩展做不到"的墙。

### "问 agent 解释自己"：文档的另一种打开方式

README 的文档入口写着一句少见的话："Read the documentation, but you can also ask the agent to explain itself."（可以读文档，也可以直接问 agent 让它自己解释。）extensions.md 的第一行更进一步："pi can create extensions. Ask it to build one for your use case."（pi 能给自己写扩展，要什么直接开口。）这两句话背后是一个自洽的立场：对 agent 而言，文档的读者有两种，人和模型；与其只优化给人读的那份，不如把文档写成模型也能直接消费的形态，让 agent 自己当讲解员和装修工。配套的动作甚至延伸到评测：evals 包的 docs lift 实验直接量化"文档对 agent 表现的贡献"，文档质量成了被测量的对象。写文档、用文档、测文档，三件事在同一个项目里转起来了。

### 把真实会话当公共数据

README 用了两节号召开发者分享开源工作场景的 agent 会话记录（通过 pi-share-hf 工具发布到 Hugging Face 数据集），作者本人定期公开自己开发 pi-mono 的全部会话。理由写在原话里：公开的真实会话包含真实的任务、工具使用、失败与修复，比玩具基准（toy benchmarks）更能推动模型、提示词、工具和评测的改进。对一个以"极简、透明"为卖点的项目，把自己的工作过程也公开出来，是言行一致的做法。

### 把依赖当代码审查：供应链加固

供应链攻击（supply chain attack）的大白话：现代软件大量使用别人写好的库，如果某个库的作者账号被盗、新版本被塞进恶意代码，所有下游软件都会中招，相当于从上游下毒。Pi 在 README 里专门有一节"供应链加固"，把 npm 依赖变更当作代码变更来审查：

- 直接外部依赖钉死精确版本（`save-exact=true`），`min-release-age=2` 拒绝发布不满两天的版本（躲开"刚发布就被投毒"的高危窗口）；
- 锁文件是权威，pre-commit 钩子拦截意外改动（除非显式设 `PI_ALLOW_LOCKFILE_CHANGE=1`）；
- CI 安装一律 `--ignore-scripts`（不执行依赖包安装时的脚本，这是最常见的投毒入口）；
- 给用户侧的传递依赖钉死原本靠 npm shrinkwrap（把传递依赖也钉死的完整清单，生成时带生命周期脚本的依赖必须过人工白名单）；2026-10-02 起这条变了：npm 安装不再保证传递依赖被钉死，需要钉死安装的改用 pi.dev 安装器（它自带 lockfile），`pi update` 会建议 npm 全局安装的用户迁移到安装器；
- 定时任务跑 `npm audit` 查已知漏洞（已脚本化为 `scripts/npm-audit.mjs`，无法修复的漏洞公告可显式豁免）；发布前用 `release:local` 在仓库外的隔离环境里装一遍验证。

单看每条都是小功夫，合起来是一堵墙。

### 值得单独记一笔的工程文化

根目录 `AGENTS.md` 是写给"人和 agent"共同遵守的开发规则，本身就是一份示范：明确代码风格（禁 `any`、禁动态 import、只用可擦除的 TypeScript 语法以兼容 Node 的直接运行模式）、明确测试纪律（不跑真实厂商 API，测试用 faux provider）、明确 git 纪律（多个 pi 会话可能同时在改这个仓库，所以禁止 `git add -A`、禁止强推，只提交自己改的文件）、明确 changelog 规范。这份文件的存在本身就说明：这个项目的日常开发大量由编码 agent 完成，规则要先写给 agent 看。

## 值得借鉴的工程实践

- **统一接口的诚实版本**。pi-ai 没有假装"各家差异能被一个漂亮接口吸收"，而是把真实差异显式建模成 `compat` 标志位、按协议复用实现、目录数据自动生成。做兼容层的正确做法：抽象统一的那部分，把不统一的暴露为配置。
- **差分渲染 + 组件级缓存 + 每帧重建布局树**。三件事配在一起：布局便宜（每帧重算），渲染昂贵（叶子缓存），落屏最贵（只写变化的行）。任何"频繁刷新的界面"都适用这套成本分层。
- **会话树的 id/parentId 模型**。分支、回溯、书签全部是纯数据操作，一个文件装下完整历史，加载时自动迁移旧版本。比"每次快照整个会话"和"只存线性日志"都好。
- **系统提示词建模为可重放补丁**。状态即历史：不存"当前提示词"，存"提示词的每次变更"，重放即状态。可审计、可回溯、天然适合追加式存储。
- **假 provider 测试法**。faux provider 让整条 agent 链路（循环、工具、UI）的测试不碰真实 API。配套纪律：需要真实模型的评测单独隔离在 evals 包里跑容器。
- **一致性测试套件**。telemetry 包把"适配器应该有什么行为"写成可执行的测试，第三方实现跑一遍即验证。比一份容易被误读的文档强得多。
- **文档也做评测**。docs lift eval 用配对实验量化文档对 agent 表现的贡献，让"文档写得好不好"从主观感受变成可测量指标。
- **把取舍写成文档**。无权限系统、No MCP、不做沙箱，每个"不做"都附理由和替代方案。对开源项目来说，明确说清楚不做什么，比堆功能更能建立信任。
- **设计文档驱动的开发节奏**。仓库根目录曾有一份 `tui-plan.md`（2026-10-05 起已从仓库移除），它是一份"实施交接文档"（implementation handoff）：先记录设计讨论定下的决策（哪些是第一期必做、哪些明确是非目标），再给出公开 API 草案、内部数据结构、逐条的测试计划和验收标准。实施者（无论人还是 agent）照着做即可，歧义在设计阶段就被消掉。这种"先写清楚再动手"的文档在快速演进的项目里比事后补的文档值钱得多。
- **运行时少一层工具**。2026-09-25 全仓库迁到 TypeScript 7 正式版（此前用 tsgo 原生预览），检查脚本移植到 TS 7 API；跑源码不再经过 tsx（一个 Node 的 TypeScript 运行时包装工具），而是直接 `node --import` 挂源码解析钩子。构建链上能去掉的中间层就去掉：编译器是官方的，运行时是裸的，出问题时少一个"是工具的锅还是代码的锅"的排查环节。

## 更新记录

- 2026-09-21 初版：基于 commit c7cdb46 完整成文。吸收首日认识篇（架构三件套、无内置权限系统、供应链加固），扩充为 12 包全览与各子系统机制。
- 2026-09-22 基于 commit 1a584a7 补充：agent 会话上下文的权威投影与追加式编辑、可编程的回合结束决策点（FinishTurn）、chord 复制状态的事务化更新；模型目录新增 Grok 4.7 支持（模型目录持续跟进的日常体现）。
- 2026-09-23 基于 commit 9672462 补充：durable 新增 SQLite 存储后端（版本化迁移、一致性测试、基准）与版本化文档存储（原子基础版本、历史读取、回收、索引化查询）。
- 2026-09-24 基于 commit b455975 补充：durable 新增第二个存储后端 JSONL（含 sidecar 回收：任务文档清理与只留当前 base，以已提交标记授权、回收前先刷主文件），一致性测试套件导出供外部后端复用。
- 2026-09-25 基于 commit 5fd446c 补充：durable 会话层做实——原子 Session 事务、检查点、文档迁移、文档重放批量优化；chord 不可变 delta 跟踪器收敛为唯一实现并强制严格 JSON 校验。
- 2026-09-26 基于 commit d6af72e 补充：durable 新增会话文档 fork（后端按策略复制、所有权索引查询）与类型化 ID（branded type、显式所有权、任务成员关系不可变）；全仓库迁至 TypeScript 7 并以裸 node 跑源码（去 tsx 中间层）。
- 2026-09-27 基于 commit 2b0a123 补充：TUI 新增 system 默认主题——界面配色从终端报告的前景/背景/ANSI 调色板推导（OKLab 亮度、WCAG 2 对比度），启动不等待（灰度先跑、颜色到达再上色），ThemedText 组件保证文本颜色跟随主题重建。
- 2026-09-29 基于 commit cb7969d 补充：coding-agent 新增实验性虚拟模型（扩展注册路由策略，按请求挑物理模型与思考档位）；durable 补任务运行时（defineTask 阶段表、单一调度回调、commit 返回类型化下一状态）与 Harness 任务接口。
- 2026-09-30 基于 commit 1b34779 补充：durable 按设计文档分期落地 Package 16-18——第一次 coding-agent 工具回合（pi.tool/pi.post-tools 任务、四个内置工具）、pi.usage 用量记录与会话视图、会话中止与所有权级联（后台任务是级联边界）。
- 2026-10-01 基于 commit 17f3dcc 补充：durable 落地 Package 19-20——结构化并发（任务显式归属、waiting/completing 状态与 failFast/allSettled 等待、自底向上中止、generation 在 tools 阶段等待自有 pi.tool 任务并删除 pi.post-tools）与上下文压缩（手动/后台/阻塞/溢出四条触发路径、stale 规则裁决并发摘要、beforeCompact 钩子）；Node SQLite 存储改异步执行并按语句缓存编译结果。
- 2026-10-02 基于 commit 7fbbd5f 补充：durable 落地 Package 21-23——扩展注册表与每会话 pi.agent 文档（内置工具改扩展形态）、生命周期收尾纪律、taskGraph 任务图视图；实验性 durable TUI 编码 agent 与 client/server 移植；pi 发布 v1.0.0，agent-core 瘦身（实验性 harness 与 session-backends 包整体移除）；架构表补录 codemode/mcp 两包，修正 MCP 支持现状（mcp.json 内置、OAuth 凭证按服务器与网址分开存）。
- 2026-10-03 基于 commit a276dab 补充：pi-ai 切换 Anthropic inline tools beta（会话中途工具变更按值内联定义、顶层工具清单固定、同名重定义不再全量重发）；供应链策略调整——移除 npm shrinkwrap、传递依赖钉死移交 pi.dev 安装器 lockfile、npm audit 脚本化；codemode 脚本输出加上限（16 Mi 字符/10 万项）防 print 循环耗尽宿主内存。
- 2026-10-05 基于 commit b2b5c42 补充：durable 执行环境有了正式接口——FileSystem（openBinaryReader 位置读/openDirReader 分页目录/FileSystem.watch 原生或轮询监视）与 Shell（argv 数组免 shell 执行、流信息上报）、全链有界（scanLines 一次遍历定位行段、shell 输出尾部窗口带跳过计数、进度节流可配置）、环境一致性测试套件导出并跑通三平台。
- 2026-10-06 基于 commit 1cedd32 补充：新包 pi-env 让执行环境搬去 SSH 远端（RemoteExecutionEnv 语义与本地一致、Rust 守护进程覆盖四平台、三平台一致性测试矩阵）；durable 的 runtime.context() 把最近读取范围与派生视图驻留内存（闲会话按 contextRetentionMs 释放）、支持按较早条目读取上下文、新增 Cloudflare Durable Object SQLite 适配器；连发 v1.0.3/v1.0.4/v1.1.0；架构表补录 env 包（现 14 个包），注明 tui-plan.md 设计文档已从仓库移除。
- 2026-10-11 基于 commit ea448f45 补充：durable 嵌套工具调用（executeTool 派生归调用者所有的 pi.tool 子任务、嵌套结果文档、重启后可重挂接、abandonOnRestart）与结构化输出（structuredOutputSchema 校验、ToolExecutionResult.content 更名 output）；存储错误判会话失败且 Harness 不重试；read 工具支持图片（限额随模型、可选 Photon WASM 处理器、图片必须整体读）；调度器归属索引按活工作定界（1 万已结束循环的归属边 10001 条降为 1 条）且不再每轮遍历全部活任务；codemode 沙箱可经远程 Durable Object 跑在 Cloudflare Workers（VM 驱动与执行逻辑拆分、中断额度覆盖 promise 排空）；coding-agent 发布配置 schema；MCP iss 校验收紧（未发现服务器元数据即拒绝）。
- 2026-10-14 基于 commit 3daa2028 补充：chord 包被 chord2 宿主运行时整体替换（契约/对象/单元/根/插件/热重载、内存与 IPC 与 WebSocket 链路、esbuild 换 typebox、流控与压缩帧等打磨）；实验性 server/protocol/client 三包与 coding-agent 实验服务栈整体删除（现 11 个包）；durable 新增代码模式（脚本以嵌套调用用工具、日志化结果重启可重放、偏离与沙箱指纹变化判错、store/load 随 fork）、延迟工具（deferred 注册加 modelTools 指名）与 tool_search 扩展；执行环境新增伪终端（pty/ptyWrite/ptyResize，Unix 专用）与 listFiles 树形列举；pi 发布 v1.2.0；杂项——会话文档缓存按使用定界（documentRetentionMs）、输入记为应用条目且仅展示的写入一次追加、图片预处理挪进 worker 线程、压缩摘要不再包含 thinking、阻塞压缩失败判运行失败、会话分享目的地可配置、OpenRouter 只列出密钥可用的模型、PI_CLIPBOARD_OSC52 强制 OSC 52 复制、修复工具调用参数里的非法 \u 转义。
- 2026-10-17 基于 commit 8efd314fa3 补充：chord 线上协议推进到 3 与 4（状态 init、类型化主体与契约放置、宿主生命周期类型检查、引用使用前状态补齐与订阅、绑定键规范化、形状帧去 codeVersion）；durable 支持命名空间化 SQLite 存储、按模型覆盖的压缩设置（compaction.modelOverrides）、函数形式的 HarnessOptions；pi-ai 的 GPT-6 思考等级切换保留提示词缓存、kimi-coding 改走 Chat Completions、NVIDIA 默认 Nemotron 3.5 Super VL；OutputBuffer/Progress 挪到 @earendil-works/chord/output；pi 发布 v1.3.0。
- 2026-10-20 基于 commit 73f8de80bc 补充：durable 升级为持久执行平台——睡眠成为持久状态（六态状态机、sleeping 带 checkpoint 与 until、deadline 存 SQLite 重启存活、schema v2 重建任务表迁移）、定时可外包宿主（HarnessOptions.setTimer/onWake、Harness.wake()、scheduler.resume()、Workers 里拒绝后台定时器的特判）、崩溃循环防护（重启计数达 restartLimit（默认 5）判 blocked 并上报 task_parked、retry() 清零恢复、ReportInfo 九类类型化上报、settings.afterFailedRun 决定失败后排队输入的去留、任务图节点带 blocked 属性）、workerd 真机测试套件（packages/durable/test-workers/）、SQLite 驱动挪至 chord/sqlite；checkpointWhen 成为 durable 文档定义必填（不再有默认值）；遥测体系落地（pi-telemetry 声明式 schema 与编译期类型检查、durable 的 DURABLE_TELEMETRY_SCHEMA 坚持只记记录标识不记内容、pi-ai 的 AI_TELEMETRY_SCHEMA 事件粒度停在块级、Context.telemetry 恒为 span、trace 上下文随任务记录持久化、chord/telemetry-sqlite 提供 TelemetryStore 存储与 trace 渲染器）；pi-ai 收到 HTTP 413 一律判请求体积超限并触发压缩重试。
