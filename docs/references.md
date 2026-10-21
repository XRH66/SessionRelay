# GitHub 参照、上游源码与取舍

初始访问日期：2026-10-21；LoopX 源码评估修订：2026-10-22。其他项目条目沿用既有记录，本次不宣称重新核验。证据分为：

- **固定 commit 源码**：已按条目记录读取范围；只做摘要，未在本机编译、安装或运行。
- **README / 官方文档**：阅读级证据；默认分支会变化，实际借代码前必须复核提交与许可证。

引用不等于复制代码，也不等于接入完成。GitHub 上游版本与本机已安装的 Codex alpha 二进制不是同一个东西。

## 1. 源码定位（固定 commit）

| 项目 | 源码 | 职责摘要 | 对应本项目 | 局限 |
|---|---|---|---|---|
| Agent Deck | [internal/session/transition_notifier.go](https://github.com/asheshgoplani/agent-deck/blob/721f1bf0f530f3e85bf1524a68b927d03b184111/internal/session/transition_notifier.go) | `TransitionNotificationEvent`、`TurnFingerprint`、`TargetSessionID`、dead-letter、dispatch；当前常量 `committed_inbox` 及注释表明已转为持久 per-parent outbox 的 pull model，父会话在 turn 边界 drain | delivery/outbox、事件身份、来源归属、去重 | README 的旧 nudge 描述不能证明“空闲桌面立即唤醒”；不得把其 exactly-once 宣传套到跨平台 |
| Agent Deck | [internal/session/transition_notifier_dedup_test.go](https://github.com/asheshgoplani/agent-deck/blob/721f1bf0f530f3e85bf1524a68b927d03b184111/internal/session/transition_notifier_dedup_test.go) | 去重测试入口 | 去重设计参考 | **仅目录发现，未逐行审查**；不能据此声称测试行为已验证 |
| agent-orchestrator | [backend/internal/adapters/chatdriver/codexappserver/rpc.go](https://github.com/OrchestratorInc/agent-orchestrator/blob/9db690843dec90ed448e4f072d1d187a26baa4c7/backend/internal/adapters/chatdriver/codexappserver/rpc.go) | newline JSON 帧、pending map 按 id 关联、writeMu 不持锁等回复、独立通知队列、server→client 审批请求 | Codex adapter 的收发关联与通知/审批分流 | 它自己管理 app-server，是会话唯一写者；不证明能无冲突控制已运行的 Codex 桌面会话 |
| agent-orchestrator | [backend/internal/adapters/chatdriver/codexappserver/conversation.go](https://github.com/OrchestratorInc/agent-orchestrator/blob/9db690843dec90ed448e4f072d1d187a26baa4c7/backend/internal/adapters/chatdriver/codexappserver/conversation.go) | `threadID`、`activeTurn`、`sendMu`、权限审批白名单，并统一为 `ChatEvent` | 会话绑定、turn 生命周期、审批门 | 其单写者模型不等于桌面 App 内部状态可被外部安全共享 |
| LangGraph | [libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py](https://github.com/langchain-ai/langgraph/blob/3cd57c038ad827d05f80c6488adea9994156c58c/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py) | `SqliteSaver.setup`；`checkpoints`/`writes` 表；`thread_id`/`checkpoint_ns`/`checkpoint_id`/`task_id` 作用域；锁与事务 | store 的持久分区、恢复边界 | checkpoint 本身不保证外部发送不重复，业务仍要幂等/receipt 核对 |
| openai/codex | [codex-rs/cli/src/queue_cmd.rs](https://github.com/openai/codex/blob/d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4/codex-rs/cli/src/queue_cmd.rs) | `QueueCommand`、`run_queue_command`；实际委托 `codex_tui::run_session_queue_command` | 理解 queue CLI 入口 | **不是全链路源码**；上游 commit 不等于本机 alpha 二进制对应 commit |
| openai/codex | [codex-rs/app-server/src/message_processor.rs](https://github.com/openai/codex/blob/d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4/codex-rs/app-server/src/message_processor.rs) | Thread/Turn/ThreadQueue 请求处理，JSONRPC 解码/分派 | app-server 协议对象与请求处理参考 | 代码存在不等于本机可连接 |
| openai/codex | [codex-rs/app-server/src/transport.rs](https://github.com/openai/codex/blob/d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4/codex-rs/app-server/src/transport.rs) | stdio/websocket/control socket、连接状态和通知能力过滤 | app-server 传输候选 | 只读上游代码，不能据此解释本机 daemon socket 错误 |
| LoopX | [semantic_request.ts](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/loopx/control_plane/collaboration/semantic_request.ts)、[result_publication.ts](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/loopx/control_plane/collaboration/result_publication.ts) | 有界语义 brief、身份冲突/重放、不可变结果修订和前序交付门 | handoff、产物版本、delivery 防重与回传顺序 | 存在 Goal/Agent 与 request_control 依赖，需显式映射；不是可无改动粘贴的库 |
| LoopX | [chat_agent.py](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/loopx/chat_agent.py)、[DSH driver.ts](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/packages/dsh-loopx-plugin/src/driver.ts) | Codex RPC/通知/turn 处理；DSH 原生事件、活跃 Agent 枚举、来源与 reservation 生命周期 | 平台 adapter 与受控宿主桥候选 | 选读范围见研究证据；自管会话/宿主内接口不等于已验证的外部原桌面对话控制 |

许可证核验程度：

- Agent Deck、agent-orchestrator、LangGraph、openai/codex、formiat：沿用此前核验程度，本次未重新审计；实际采用前复核对应文件许可。
- LoopX：本次读根 LICENSE、NOTICE、pyproject 与 DSH package.json。当前快照为 Apache-2.0；NOTICE 说明截至 v0.4.7 的历史发行采用 MIT，不视为当前任意二选一。未完成所有传递依赖的许可审计。

## 2. 机制摘要与取舍

### Agent Deck：事件身份与持久收件箱

固定 commit 的 `committed_inbox` 说明持久 per-parent outbox 的 pull model，父会话在 turn 边界 drain。借鉴 TurnFingerprint、目标会话归属、持久收件箱、dead-letter 和去重。不据 README 旧 nudge 宣称空闲桌面立即唤醒，不扩大 exactly-once 保证，也不引入模型无任务定时唤醒。测试入口仅发现目录，未逐行审查。

### agent-orchestrator：Codex app-server 客户端

借鉴 JSON 帧、按 id 匹配响应、写锁不跨长等待、通知/审批分流和会话单写者。其客户端自己管理 app-server，不能证明外部程序可无冲突附着已运行的 Codex 桌面会话。

### LangGraph：持久化分区，不是外部幂等证明

借鉴 checkpoint 分区与批准门；checkpoint 不能保证外部发送不重复。业务仍需 receipt 核对、幂等边界与人工处理，interrupt 节点重执行也要单独评估。当前没有引入该运行时。

### openai/codex：协议设计与本机能力分开

queue_cmd.rs 只是入口与委托，不是完整唤醒链。message_processor.rs 与 transport.rs 解释候选协议和传输设计，不替代本机完成链验证。安装版本与边界见 [P0 摘要](p0-evidence-summary.md) 和 [codex.md](platforms/codex.md)。

### formiat/multi-agent-orchestration：技能/工作流参考

沿用 README 阅读记录：复用已有会话、歧义即停、持久交接、实现→审查→返修。它不是已集成的三平台 connector，不把单文件覆盖式 mailbox 套到并发连接，不伪造源码定位。

### LoopX：允许模块级融合，不预先排除代码复用

本轮查证 LoopX 不仅有 Python，也有 TypeScript 控制面、原生 DSH 插件、多 Agent 协作与桌面预览。建议优先吸收身份/版本校验、不可变结果链、平台事件处理和回传未知状态规则；研究保留小模块提取、可选治理子进程、整套底座三种路线的比较。

不将文件存在或 hash 当业务正确性证明，不把上游 Goal/Agent ID 当平台 session ID，不把自管新会话示例当原桌面对话桥接，也不把具体低价模型名称写成未经查证的通用规则。当前运行时和阶段状态未改变，是否采纳融合建议仍须独立决定。

详见 [LoopX 研究总览](research/loopx/README.md)；源码范围、许可和旧结论修正见 [证据清单](research/loopx/07-evidence-and-corrections.md)。旧 [loopx-analysis.md](references/loopx-analysis.md) 保留为导航入口。

## 3. 文档入口

- Agent Deck README（固定 commit）：<https://github.com/asheshgoplani/agent-deck/blob/721f1bf0f530f3e85bf1524a68b927d03b184111/README.md>
- agent-orchestrator README（固定 commit）：<https://github.com/OrchestratorInc/agent-orchestrator/blob/9db690843dec90ed448e4f072d1d187a26baa4c7/README.md>
- LoopX 研究固定快照：<https://github.com/loopx-project/loopx/tree/5bf58c1ca3d21919b15c662bc705c3e069bfb496>
- LangGraph 持久化文档：<https://docs.langchain.com/oss/python/langgraph/persistence>
- LangGraph 人工中断文档：<https://docs.langchain.com/oss/python/langgraph/interrupts>
- formiat README（仅工作流说明）：<https://github.com/formiat/multi-agent-orchestration>

默认分支文档仍是阅读级证据，不能与本机实测混写。

## 4. 本项目当前取舍

保持 SessionRelay 的会话绑定、用户控制、真实结果关联和单一投递状态所有者。当前没有引入上述项目作为运行时依赖；LoopX 的融合路线仅为研究建议，不把暂未采用写成永久禁止。

Codex 原桌面附着与 queue 结果关联、DSH 正式会话发现、Antigravity 稳定入口仍以平台矩阵和实际验证为准。本文件不宣称存在已完全符合需求的成品，也不宣称上游代码已被本项目集成。
