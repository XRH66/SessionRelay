# Codex 接入架构与自查记录

本文只记录 **SessionRelay 对 Codex 的可验证接入边界**，不是闭源 Codex 桌面端的内部实现说明。证据分为：用户/GPT 实证、本机只读实测、本机生成 schema、上游固定 commit 源码、未验证候选。P0 没有把任何候选写成已接入。

入口与版本数据摘要见 [P0 证据摘要](../p0-evidence-summary.md)。源码定位与许可证程度见 [docs/references.md](../references.md) 第 1 节。

## 1. 四层模型

~~~text
┌─────────────────────────────────────────────────────────────┐
│ 1. Codex 桌面用户会话                                       │
│    ChatGPT.exe 多进程 + 运行中的 codex.exe；用户正在看的对话 │
│    内部会话/线程实现闭源，未作为公开 API 使用                 │
└───────────────────────────┬─────────────────────────────────┘
                            │ 不是已证实的公开外部接口
┌───────────────────────────▼─────────────────────────────────┐
│ 2. CLI queue 与 app-server 入口                             │
│    codex queue --thread <UUID|name> --message <TEXT>        │
│    codex app-server：stdio / ws / control socket（候选）     │
│    daemon 控制 socket 在本机报错；本轮未启动 daemon           │
└───────────────────────────┬─────────────────────────────────┘
                            │ JSON-RPC 对象（本机 schema）
┌───────────────────────────▼─────────────────────────────────┐
│ 3. thread / turn / item 协议                                │
│    thread/list、thread/read、thread/resume                  │
│    turn/start、turn/steer、turn/interrupt                   │
│    turn/started、turn/completed、审批与用户输入请求           │
└───────────────────────────┬─────────────────────────────────┘
                            │ 适配器（P1 才实现）
┌───────────────────────────▼─────────────────────────────────┐
│ 4. SessionRelay 进程                                        │
│    main 进程的 Codex adapter；renderer 不接触 CLI/凭据       │
│    P0 只有文档与契约，没有可运行 adapter                     │
└─────────────────────────────────────────────────────────────┘
~~~

**边界结论**：Codex 工具内部能看到 list/read/send，不等于外部程序已有同等 API。第 2、3 层是否能在 SessionRelay 自己的连接上复用，必须另证；第 1 层的桌面内部状态不能因为进程可见就当作可调用接口。

## 2. queue 真实唤醒证据（用户/GPT 实证，本轮未重测）

| 项目 | 记录 |
|---|---|
| 唤醒标记 | `WAKE-PROBE-<TIMESTAMP>` |
| 目标 thread | `<CODEX_THREAD_UUID>` |
| 入口 | `codex queue --thread <UUID> --message <TEXT>` |
| 回执形态 | CLI 返回 messageId；原始完整 JSON 未保留 |
| 观察 | GPT 结束本轮后，排队消息唤醒下一轮并回复确认；标记确实到达下一轮 |
| 本轮动作 | **没有**再向任何 Codex 会话投递 |

**只证明了什么**：queue 能把消息接受并排入该 thread，且在目标会话结束当前轮后触发下一轮，标记确实到达下一轮。

**没有证明什么**：
- 没有自动回读最终结果；
- 没有证明 messageId 能关联到具体 turn 或最终回复；
- 没有证明 `turn/completed` 会到达 SessionRelay 自己的连接；
- 没有证明外部人工消息与排队消息可区分；
- 没有 exactly-once，也没有可撤回平台已接收任务的能力。

因此不能把“唤醒成功”写成“完成链已打通”。P1 的完成链验收必须另做，见第 6 节。

## 2.1 独立隔离会话与最低档位实测（本轮实测，2026-10-22）

为解决测试消息可能污染用户正式工作会话并消耗昂贵旗舰模型配额的问题，本轮在用户授权下执行了一次只读隔离测试：

- **入口命令**：`codex exec -m gpt-5.6-luna --ephemeral --skip-git-repo-check -s read-only -`
- **实测证据**：
  - 成功生成全新独立内存会话，未污染用户桌面任何已有对话；
  - 显式覆盖默认配置，锁定调用最轻量档位 `gpt-5.6-luna`，成功返回 `{"status":"ok"}`，耗时约 7 秒；
  - 发现本机默认开启 `reasoning effort: xhigh`，后续 P1 可通过 `-c model_reasoning_effort="low"` 进一步压降 token 消耗。
- **详见证据摘要**：[P0 evidence summary](../p0-evidence-summary.md)。

## 3. 本机生成 schema 中的协议对象（候选，未连接实测）

以下来自本机 `codex app-server generate-json-schema` 的临时产物；临时生成物已清理，只保留脱敏后的 [P0 evidence summary](../p0-evidence-summary.md)。schema 标注 experimental，升级可能改字段。

### 3.1 thread 枚举、读取与恢复

- `thread/list`：archived、cursor、cwd、excludedThreadIds、limit、modelProviders、originators、searchTerm、sectionId、sortDirection、sortKey、sourceKinds、useStateDbOnly。
- `thread/loaded/list`、`thread/read`、`thread/turns/list`、`thread/items/list`。
- `thread/resume`；另有 `thread/start`、`thread/fork`、`thread/archive`、`thread/delete`。

这些字段说明协议**设计了**会话枚举和已有 thread 恢复入口；没有说明本机 app-server 能附着到用户正在看的桌面对话，也没有说明不会与桌面进程争用线程。

### 3.2 turn 投递、完成与状态

- `turn/start`：input[]、clientUserMessageId、approvalPolicy、sandboxPolicy、outputSchema、parentTurnId/rootTurnId 等。
- `turn/steer`：expectedTurnId 必须匹配当前活动 turn。
- `turn/interrupt`。
- 通知：`turn/started`、`turn/completed`（threadId + turn）。
- `Turn.status`：completed / interrupted / failed / inProgress。
- Turn 记录字段：id、items、completedAt、durationMs、error。
- 状态事件：`thread/status/changed`（notLoaded / idle / systemError / active+activeFlags）、`thread/queue/changed`（只有 threadId）、`thread/readState/changed`（只表示未读位置）。

### 3.3 用户消息、审批与关联候选

- UserMessageThreadItem：type=userMessage、id、clientId（nullable）、content[]。
- 审批/用户输入：`item/tool/requestUserInput`、`item/commandExecution/requestApproval`、`item/fileChange/requestApproval`、`item/permissions/requestApproval`、`serverRequest/resolved`。
- 关联候选一：`turn/start` 带唯一 `clientUserMessageId`，回读 UserMessage.clientId。
- 关联候选二：queue 返回的 messageId 被写入 userMessage.id/clientId。
- 外部人工输入候选：没有我方 clientId 的 userMessage 视为“可能的人工输入”。

**所有关联都仍是候选，未实调。** 桌面消息是否总带 clientId、queue 是否保留 clientId、通知是否实时到达，均未验证。

## 4. daemon、控制 socket 与独立传输

本机实测（只读）：

~~~text
codex app-server daemon version
Error: failed to connect to %USERPROFILE%\.codex\app-server-control\app-server-control.sock
套接字操作遇到了一个已死的网络。 (os error 10050)
~~~

该控制 socket 目录不存在。此前 queue 真实成功。两条路径结果不同，只能说明当前入口表现不同：

- 不能因为 daemon socket 失败就否定 queue；
- 不能因为 queue 成功就声称 daemon 可用；
- 不能据此推断两者共用或使用不同的底层传输、协议或进程；
- P0 未启动 daemon，未探测 `codex-ipc` 的写入协议；
- 独立 app-server 的 stdio/ws 未连接、未实测；远程 ws 认证方式也未验证。

上游源码中的 `transport.rs` 有 stdio/websocket/control socket 设计，但这只是**上游代码阅读**，不是本机运行证据。public upstream 版本也不等于本机安装的 alpha 二进制。

## 5. 上游固定 commit 源码定位

以下文件只用于理解协议设计，未在本机编译或运行。详细表格、许可证程度与其余参考见 [docs/references.md](../references.md)。

- `codex-rs/cli/src/queue_cmd.rs`（commit `d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4`）：`QueueCommand`、`run_queue_command`，实际委托 `codex_tui::run_session_queue_command`。**不是 queue 全链路源码**，不能据此补出唤醒/结果关联实现。
  链接：<https://github.com/openai/codex/blob/d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4/codex-rs/cli/src/queue_cmd.rs>
- `codex-rs/app-server/src/message_processor.rs`（同一 commit）：Thread/Turn/ThreadQueue request processor，JSONRPC 解码/分派。说明协议请求处理的分层，不证明本机可连接。
  链接：<https://github.com/openai/codex/blob/d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4/codex-rs/app-server/src/message_processor.rs>
- `codex-rs/app-server/src/transport.rs`（同一 commit）：stdio/websocket/control socket 导出、连接状态、通知能力过滤。只解释候选传输设计，不解释本机 10050 的根因。
  链接：<https://github.com/openai/codex/blob/d138398cef7d49e46fe33f1d0ddbd2ca0ed16aa4/codex-rs/app-server/src/transport.rs>

补充：agent-orchestrator 的 Codex app-server 客户端源码（固定 commit `9db690843dec90ed448e4f072d1d187a26baa4c7`）可参考 JSON 帧、pending map、写锁与审批分流，但它自己管理 app-server、是唯一写者，不证明可无冲突附着到本机正在运行的桌面会话。见 [references.md](../references.md)。

## 6. 后续最小验收（P1 前置，不等于已执行）

在进入 P1 双向往返接力前，必须先行执行有明确终点的 **Codex 完成链验证任务**（详细计划见 [phases.md](../phases.md) 第 P1 节）。只有下列项目全部有证据，才能说 Codex adapter 具备最小可用接入。任何一项失败都必须停机并进入 `needs_attention`，绝不能偷偷新建会话或克隆历史。

1. **同一已选桌面对话与明确失败边界**：用户选定已有桌面 thread UUID（如 `<CODEX_THREAD_UUID>`）；连接只尝试 `thread/resume` 或只读读取，核验能否在不干扰桌面当前交互的前提下附着。若发生 10050 错误、锁冲突或桌面崩溃，立即判定失败停机，**绝对禁止通过 `thread/start`、`thread/fork` 或克隆历史伪造成功**。
2. **启动/完成关联**：每次 `turn/start` 带唯一 `clientUserMessageId`（如 `RELAY-VERIFY-<TIMESTAMP>`）；记录 threadId、turnId、clientUserMessageId；收到 `turn/completed`（或回读）后核验内部包含匹配的 `clientUserMessageId`，证明“这次投递 → 这个 turn → 这个结果”的完整链。queue 路径另证 messageId 关联，不得混用两套证明。
3. **终态分离与会话占用**：completed / interrupted / failed / 审批等待必须分开记录；在等待结果期间会话保持锁定，拒绝并发投递；不能把“没有新消息”或“进程还在”当完成；错误码与错误文本保留引用，不复制私人正文。
4. **人工插话区分**：我方消息带唯一 clientId；没有我方 clientId 的 userMessage 标为可能的人工输入，暂停协调并进入 `needs_attention`。若平台不保留 clientId，只能标 partial，UI 明示无法自动区分。
5. **暂停竞争**：发送前 gate 检查、持久化 `sending`、发起 submit 必须是同一个短调度动作连续执行；pause 不能插在中间；已发起/已被平台接收的任务不假装可撤回；receipt/结果等待在短命令队列外。实现规则见 [architecture.md](../architecture.md) 第 4 节。
6. **断线恢复**：通知流断开后先核对 thread/turn 状态与已有 receipt；无法证明是否接收则标 unknown，保持会话锁定不自动重发；不允许静默新建会话或重放旧 revision。
7. **数据与日志边界**：只持久化任务元数据、交接信封、文件指针引用与脱敏摘要；严禁持久化认证凭据，严禁整段复制平台日志或私人对话全文。

**验收反例**：只收到 queue 的 messageId、只看到下一轮回复、只查到 thread/list 方法、只读到上游源码，都不算完成关联。

## 7. P0 已知与待验证

| 类别 | 已知（有证据） | 待验证（不能写成已支持） |
|---|---|---|
| 版本/入口 | 本机 CLI 0.163.0-alpha.11；桌面 App 与运行中 codex 进程存在；queue 帮助可读 | 本机二进制对应哪个上游 commit；桌面内部调用路径 |
| 投递 | queue 用户/GPT 实证：脱敏探针到达下一轮并返回 messageId | SessionRelay 自动投递；messageId 与 turn/结果关联 |
| 协议 | 本机 schema 有 thread/list/read/resume、turn/start/completed、UserMessage.clientId、审批请求 | 实际连接、实时通知、错误/审批/人工输入字段行为 |
| 完成链 | 协议层存在 turn/completed 与 Turn 状态 | 在我们的连接上收到完成事件并回读结果 |
| 传输 | 控制 socket 报 os error 10050；上游有 stdio/ws/control socket 代码 | 独立 stdio/ws 是否可用；queue/daemon 传输关系；断线恢复 |
| 桌面会话 | 用户已有桌面 thread UUID 可记录 | 外部程序能否安全附着该 thread 且不与桌面冲突 |

**结论：P0 查到源码、协议和一次 queue 唤醒，但 Codex 接入尚未完成；完成关联是 P1 前置缺口。**
