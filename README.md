# awesome-agents-notes

六个 agent 项目（AI 编码助手与其运行框架）的中文技术文档。每篇都是完整的技术文档，不是按日日志；上游每有实质更新就把新机制融入对应章节，每 3 天增量维护一轮，每次补充都在文末"更新记录"注明依据的 commit。

## 当前跟踪的项目

| 项目 | 上游仓库 | 一句话定位 | 笔记 |
|---|---|---|---|
| codex | [openai/codex](https://github.com/openai/codex) | OpenAI 的终端编码 agent，Rust 核心，多端复用一套协议 | [notes/codex.md](notes/codex.md) |
| deepseek-harness | [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) | DeepSeek 的 agent 框架（dsh），事件溯源日志之上的可插拔能力系统 | [notes/deepseek-harness.md](notes/deepseek-harness.md) |
| hermes-agent | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) | Nous 的个人 AI 助手：聊天网关、记忆与技能的学习循环、多平台接入 | [notes/hermes-agent.md](notes/hermes-agent.md) |
| openclaw | [openclaw/openclaw](https://github.com/openclaw/openclaw) | 本地优先的个人 agent 网关，能力几乎全走插件 | [notes/openclaw.md](notes/openclaw.md) |
| opencode | [anomalyco/opencode](https://github.com/anomalyco/opencode) | 终端编码 agent，Effect 架构，核心服务化、多端接入 | [notes/opencode.md](notes/opencode.md) |
| pi | [earendil-works/pi](https://github.com/earendil-works/pi) | 极简理念的 agent 工具包：统一模型接口、终端编码 agent 与持久化运行时 | [notes/pi.md](notes/pi.md) |

## 文档结构

每篇笔记固定为七个部分：项目简介 → 它解决什么问题 → 整体架构 → 核心机制（按子系统分节）→ 设计哲学与取舍 → 值得借鉴的工程实践 → 更新记录。面向没接触过该项目的读者，冷门术语第一次出现都有大白话解释。

## 维护方式

本地克隆这六个仓库，每 3 天拉取一次上游提交，判定有无实质内容（新功能、新机制、行为变化、架构调整），有就融入对应章节并在"更新记录"小节登记。写作细节见各篇笔记本身。
