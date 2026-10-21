# GitHub 参照、上游源码与取舍

访问日期：2026-10-21。本文件区分两类证据：

- **固定 commit 源码**：GPT 已通过 GitHub API 核实 SHA 并实际读取；本文件只做短摘要，未在本机编译、安装或运行。
- **README / 官方文档**：此前只读页面，未实测；默认分支内容会变化，未来借代码前必须复核 commit 与许可证原文。

引用不等于复制代码，也不等于接入完成。GitHub 上游版本与本机已安装的 Codex alpha 二进制不是同一个东西。

## 1. 源码定位（固定 commit）

| 项目 | 源码 | 职责摘要 | 对应本项目 | 局限 |
|---|---|---|---|---|
| Agent Deck | [internal/session/transition_notifier.go](https://github.com/asheshgoplani/agent-deck/blob/721f1bf0f530f3e85bf1524a68b927d03b184111/internal/session/transition_notifier.go) | `TransitionNotificationEvent`、`TurnFingerprint`、`TargetSessionID`、dead-letter、dispatch；当前常量 `committed_inbox` 及注释表明已转为持久 per-parent outbox 的 pull model，父会话在 turn 边界 drain | delivery/outbox、事件身份、来源归属、去重 | README 的旧 nudge 描述不能证明“空闲桌面立即唤醒”；不得把其 exactly-once 宣传套到跨平台 |
| Agent Deck | [internal/session/transition_notifier_dedup_test.go](https://github.com/asheshgoplani/agent-deck/blob/721f1bf0f530f3e85bf1524a68b927d03b184111/internal/session/transition_notifier_dedup_test.go) | 去重测试入口 | 去重设计参考 | **仅目录发现，未逐行审查**；不能据此声称测试行为已验证 |
| agent-orchestrator | [backend/internal/adapters/chatdriver/codexappserver/rpc.go](https://github.com/OrchestratorInc/agent-orchestrator/blob/9db690843dec90ed448e4f072d1d187a26baa4c7/backend/internal/adapters/chatdriver/codexappserver/rpc.go) | newline JSON 帧、pending map 按 id 关联、writeMu 不持锁等回复、独立通知队列、server→client 审批请求 | Codex adapter 的收发关联与通知/审批分流 | 它自己管理 app-server，是会话唯一写者；不证明能无冲突控制已运行的 Codex 桌面会话 |
| agent-orchestrator | [backend/internal/adapters/chatdriver/codexappserver/conversation.go](https://github.com/OrchestratorInc/agent-orchestrator/blob/9db690843dec90ed448e4f072d1d187a26baa4c7/backend/internal/adapters/chatdriver/codexappserver/conversation.go) | `threadID`、`activeTurn`、`sendMu`、权限审批白名单、统一 `ChatEvent` | 会话绑定、turn 生命周期、审批门 | 同上；其单写者模型不等于桌面 App 内部状态可被外部安全共享 |
| LangGraph | [libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py](https://github.com/langchain-ai/langgraph/blob/3cd57c038ad827d05f80c6488adea9994156c58c/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py) | `SqliteSaver.setup`；`checkpoints`/`writes` 表；`thread_id`/`checkpoint_ns`/`checkpoint_id`/`task_id` 作用域；锁与事务 | store 的持久分区、恢复边界 | 只借持久化分区；checkpoint 本身不保证外部发送不重复，业务仍要幂等/receipt 核对 |
| openai/codex | [codex-rs/cli/src/queue_cmd.rs](https://github.com/openai/codex/blob/d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4/codex-rs/cli/src/queue_cmd.rs) | `QueueCommand`、`run_queue_command`；实际委托 `codex_tui::run_session_queue_command` | 理解 queue CLI 入口 | **不是全链路源码**；上游 commit 不等于本机 alpha 二进制对应 commit |
| openai/codex | [codex-rs/app-server/src/message_processor.rs](https://github.com/openai/codex/blob/d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4/codex-rs/app-server/src/message_processor.rs) | Thread/Turn/ThreadQueue request processor，JSONRPC 解码/分派 | app-server 协议对象与请求处理参考 | 代码存在不等于本机可连接；本机 app-server 未实测 |
| openai/codex | [codex-rs/app-server/src/transport.rs](https://github.com/openai/codex/blob/d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4/codex-rs/app-server/src/transport.rs) | stdio/websocket/control socket 导出、连接状态、通知能力过滤 | 未来 app-server 传输候选 | 只读上游代码；本机 stdio/ws 未连接；不能据此解释本机 daemon socket 错误 |
| LoopX | [huangruiteng/loopx](https://github.com/huangruiteng/loopx) | 本地状态内核、状态与执行分离、Proof-or-Stop 证据门、标准化交接信封、Quota & Cost Guard | 证据门实体文件校验、发送前配额/模型档位防踩坑哨兵、结构化信封分区 | 见 [loopx-analysis.md](references/loopx-analysis.md)；为 Python 栈且偏单 Agent 滚动，不搬源码，只借架构与状态思想 |

许可证核验程度：

- Agent Deck：此前 README 标注 MIT；本轮未重读 LICENSE 原文。
- agent-orchestrator：此前 README 标注 Apache-2.0；本轮未重读 LICENSE 原文。
- LangGraph：此前未逐文件核验；本轮不假定其许可证，实际借用前必须复核。
- openai/codex：本轮未重新核验许可证原文；实际借用前必须复核。
- LoopX：开源仓库标注 Apache-2.0 / MIT。
- formiat：许可证本轮未核验；它只作为技能/工作流参考，不是编译源码。

## 2. 机制摘要与取舍

### Agent Deck：事件身份与持久收件箱

- 固定 commit 源码中的 `committed_inbox` 与注释说明：通知已转成持久 per-parent outbox 的 pull model，父会话在 turn 边界 drain。README 里较早的“nudge 唤醒”描述不能直接当作当前版本能力。
- **借鉴**：事件身份（`TurnFingerprint`）、目标会话归属（`TargetSessionID`）、持久收件箱、dead-letter、去重与显式来源。
- **不照搬**：不承诺“空闲桌面立即唤醒”；不把 Agent Deck 的 exactly-once 说法扩大成跨平台恰好一次；其 heartbeat 会变成模型定时唤醒，与本项目“无新结果不调用模型”冲突。
- 去重测试文件只发现目录，未逐行审查，不能声称其测试行为已复现。

### agent-orchestrator：Codex app-server 客户端

- `rpc.go` 使用 newline JSON 帧，按 id 维护 pending map；writeMu 不持锁等待回复；通知走独立队列；server→client 审批请求单独处理。
- `conversation.go` 维护 `threadID`、`activeTurn`、`sendMu`、权限审批白名单，并统一为 `ChatEvent`。
- **借鉴**：请求/响应按 id 关联、写入锁不要跨越等待、通知与审批不能混在普通响应里、会话级单写者。
- **局限**：它自己启动/管理 app-server，是唯一写者；这不证明外部程序能安全附着到正在运行的 Codex 桌面会话，也不证明桌面内部线程锁可共享。

### LangGraph：持久化分区，不是外部投递的幂等证明

- `SqliteSaver.setup` 建 `checkpoints`/`writes` 表，以 `thread_id`/`checkpoint_ns`/`checkpoint_id`/`task_id` 划作用域，并用锁和事务维护一致性。
- **借鉴**：持久化分区、恢复边界、人工批准门。
- **修正**：此前写的“恢复时避免重复副作用”不准确。checkpoint 本身不保证外部发送不重复；业务还要幂等键、receipt 核对与人工对账。带 interrupt 的节点重执行也要单独评估，不能把 checkpoint 当副作用隔离。
- **不引入依赖**：本项目仍用轻量 Electron + SQLite 自行实现，不把 LangGraph 当运行时。

### openai/codex 上游：只解释协议设计，不证明本机能力

- `queue_cmd.rs` 只是 CLI 的 `QueueCommand` 定义和委托，真正的排队/唤醒链路在 `codex_tui::run_session_queue_command`，不能把该文件当作完整实现。
- `message_processor.rs` 展示 Thread/Turn/ThreadQueue 请求处理与 JSONRPC 分派；`transport.rs` 展示 stdio/websocket/control socket 导出、连接状态和通知能力过滤。
- **借鉴**：协议分层、请求处理器、传输抽象、通知过滤。**局限**：上游代码不等于本机安装版本；本机未连接 app-server，未跑过这些源码；不能据此声称“本机已有稳定 API”。
- 安装版本与边界见 [P0 evidence summary](p0-evidence-summary.md) 与 [docs/platforms/codex.md](platforms/codex.md)。

### formiat/multi-agent-orchestration：技能/工作流参考

- 它不是编译源码，也没有可引用的固定 commit 源码定位；只按此前读到的 README 记录其“复用已有会话、歧义即停、inbox/outbox 交接、实现→审查→返修”的工作流。
- **借鉴**：明确绑定、持久交接、角色分工、歧义即停。**不照搬**：单一覆盖式 mailbox 不适合两条并发连接；它不是控制三款桌面会话的 connector。
- 不伪造源码路径，不把它写成已经集成。

### LoopX：证据门、成本防踩坑与状态分层

- 核心架构是“状态与执行分离”的本地状态内核（Local-first State Kernel），不亲自调用 LLM，不直接修改业务代码，将持久状态与底层 Agent Runtime 分离。
- **借鉴**：
  1. **“Proof-or-Stop” 证据门**：在 `handoff` 校验中强化对交付文件的实体存在性、大小与 SHA-256 哈希核验，杜绝模型虚报成功；
  2. **Quota & Model Guard（成本防踩坑哨兵）**：投递前检查模型档位与思考参数，严格防止意外调度高成本旗舰模型；
  3. **标准化交接信封分区**：结构化细化为已证事实、受检产物、未决决策、下一棒指令。
- **不照搬**：LoopX 为 Python 栈，且面向单 Agent 多天长周期任务恢复；SessionRelay 为 Windows 桌面跨会话（Codex ↔ DSH）协作台，采用 TypeScript + Electron + SQLite 原生实现其思想，不引入 Python 依赖。详细分析见 [docs/references/loopx-analysis.md](references/loopx-analysis.md)。

## 3. README / 官方文档入口

- Agent Deck README（固定 commit）：<https://github.com/asheshgoplani/agent-deck/blob/721f1bf0f530f3e85bf1524a68b927d03b184111/README.md>
- agent-orchestrator README（固定 commit）：<https://github.com/OrchestratorInc/agent-orchestrator/blob/9db690843dec90ed448e4f072d1d187a26baa4c7/README.md>
- LoopX 仓库与文档：<https://github.com/huangruiteng/loopx>
- LangGraph 持久化文档：<https://docs.langchain.com/oss/python/langgraph/persistence>
- LangGraph 人工中断文档：<https://docs.langchain.com/oss/python/langgraph/interrupts>
- formiat README（默认分支，无固定 commit；仅工作流说明）：<https://github.com/formiat/multi-agent-orchestration>

以上 README/文档仍是阅读级证据；只有在固定 commit 上读到的源码才进入第 1 节表格。两者不能混写为“已实测”。

## 4. 本项目取舍

- **主要借鉴**：Agent Deck 的事件身份、来源归属、持久收件箱与去重；agent-orchestrator 的 JSON-RPC 关联、单写者与通知/审批分流；LoopX 的“Proof-or-Stop”证据实体核验与成本防踩坑哨兵；LangGraph 的持久分区与人工批准；openai/codex 上游的协议分层。
- **保持轻量**：Electron + React + SQLite 自实现，不引入上述项目作为运行时依赖，不照搬它们的进程模型、heartbeat、worker/worktree 或 app-server 管理方式。
- **接入边界**：查到源码只说明设计可参考；Codex 是否可附着现有桌面对话、messageId 是否能关联结果、DSH 是否有正式会话枚举、Antigravity 是否有稳定入口，仍以平台矩阵为准。
- **不宣称**：不宣称已有完全符合本需求的成品，不宣称 GitHub 上不存在其他相关项目，不宣称上述固定 commit 的代码已被本项目使用。
