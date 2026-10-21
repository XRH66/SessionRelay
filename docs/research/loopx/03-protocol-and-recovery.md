# 03 · 消息流转、身份关联、暂停与恢复

本页左侧是已读源码事实，右侧取舍是 SessionRelay 的研究建议；不宣称已移植或测试。来源见 [证据清单](07-evidence-and-corrections.md)。

## 1. LoopX 不是简单把上一条聊天文本复制过去

已读协作边界可概括为：原始请求与受限上下文 → 目标身份和授权检查 → 持久请求 → 接收方评估 → 工作/产物关联 → 发布结果 → 返回原请求者或原会话 → 核对交付。注册、请求接收、采纳、执行、结果发布和送达是不同事实。[LX04、LX06、LX07、LX13]

`semantic_request.ts` 的真实 brief 为 `collaboration_brief_v0`，字段是 purpose、context、constraints、inputs、acceptance、return_requirement。它限制字段、文本体积、列表数量和工作区相对路径；输入 sha256 的格式通过，不等于文件内容已由可信宿主计算核验。receiverDecision 的 adopt/defer/reject/no_change 也不等于 SessionRelay 的 submit_for_review/request_changes/stage_passed/needs_user。[LX06]

**建议：保留本项目外层 taskId/stageId/connectionRevision/decision，吸收 brief 的语义分区作为内层上下文，不直接替换现有契约。** 路由、角色、审批仍由主进程校验；模型给出的目标或路径只是待核验数据。

## 2. 四类身份不能合并

| 身份域 | 含义 | 建议保存/校验方式 |
|---|---|---|
| 平台身份 | platform/instance/session | 始终指向同一原会话；显示标题不能作键 |
| 业务身份 | connection/task/stage/revision/delivery | 由 SessionRelay 创建并持久化；绑定与路由改变时隔离旧结果 |
| 传输身份 | RPC id、platform message/turn/request id | 按适配器记录映射及证据；不能把 RPC id 当可跨进程恢复凭证 |
| 产物身份 | result revision、commit、digest、artifact ref | 审查和批准绑定具体版本；新内容不覆盖已批准旧内容 |

LoopX 的 request/Goal/Agent/source/Goal instance 核验与不可变 publication 能启发上述映射，但这些 ID 域不能直接互换。父请求证明来源，不授予父级权限。[LX04、LX06、LX07]

## 3. 接收与结果分别确认

对 SessionRelay，最少保留以下区分：

- transport accepted：平台接受了投递；不代表执行完成。
- platform terminal：该原生请求/轮次已到 completed/failed/interrupted 等终态。
- handoff validated：最终输出属于正确任务、阶段与绑定，且结构合法。
- artifact checked：交付物版本和按需执行的验收证据匹配。
- forward accepted / stage approved：下一方已接收，或用户明确批准了阶段。

建议把这些作为现有 delivery/result/event 的事实和投影，不立即增加五套平行状态机。LoopX 明确将 ACK、结果文件存在、返回义务已清除分开，正适合避免 UI 误报“全部完成”。[LX04]

## 4. 不可变结果和修订链

`planCollaborationResult` 不允许用同一身份静默替换已提交结果；同内容可识别为 replay，变更使用稳定 update_id 和新的 conclusion 序列。旧请求修订不自动继承最新版本；后续结果更新不能越过尚未确认送达的前一结果。[LX07]

建议移植为以下行为：

1. 人工编辑生成新的投递载荷版本，保留模型原始交接；旧批准不再适用。
2. 审查结果记录所针对的产物版本，而不是只写一个可被覆盖的文件名。
3. 绑定切换后，新目标收到的是显式迁移包，不把旧 revision 的待发结果自动放行。
4. 同一来源结果的重复事件必须回到已存在的 forward delivery，不能创建新记录绕开 unknown。

注意现有 SQL 已按 source_delivery_id 和 bound_revision 设计防重；若未来加入多结果修订或多目标转发，需先明确 result_id/payload_version/target 的唯一键，再调整 schema。不要为未来拓扑提前扩成通用工作流引擎。[SR02]

## 5. 回传与未知状态：借鉴规则，不复制整套存储

`roundtrip.py` 保存原始 session/client turn/channel 路由，冲突时拒绝；投递记录存在但缺少 provider locator 时，不能当作没有发过。已读回传路径在 prior verification 或 attempt 存在时进入验证流程，缺 locator/verifier 则显式未验证，不直接重新发送。[LX13]

这与 SessionRelay 的 durable outbox 和 unknown 禁止盲重发方向一致。应保留本项目 SQLite 单写者，把 attempt/receipt/readback 映射放进既有存储；不同时建立另一套文件 mailbox 让两个系统分别决定是否重发。

本轮没有审计 LoopX 所有重试分支和所有崩溃窗口，不能给它或组合后的系统附加端到端 exactly-once 保证。LoopX 私有 Chat store 的确定性消息 ID，也不等于外部平台支持同一幂等键。

## 6. 暂停、等待与退出

DSH driver 的 queued/claimed/admitted 分层值得借鉴：暂停先取消未来准入，已领取工作单独处理。Codex 客户端把写入边界与长响应等待分开，也适合避免 pause 被模型等待堵住。[LX05、LX08]

保留 SessionRelay 当前规则：发送前重新检查暂停门、revision 和占用；发起后是可能已被接受的在途投递。取消等待器不代表远端任务取消；网络超时也不代表远端未收。用户关闭/退出程序不能未经授权杀死平台任务。

LoopX 有宿主周期与 quota/heartbeat 逻辑，但不应整段带入。只有新结果、明确人工操作或已授权的恢复核对触发下一步；可以有有界的状态回读，不能为了“看看有没有事”反复调用模型。

## 7. 产物证据：哈希有用，但不是正确性证明

建议采用三层验收：身份与来源正确；内容版本可复核；任务专属验收成功。文件存在、非空和摘要只覆盖前两层的一部分。空文件可能是合法交付，摘要正确的程序也可能无法编译；模型自己填的 hash 不能自动变成可信证据。

文件引用需限制在用户授权的工作区，区分数据路径与可执行命令，防止绝对路径、目录穿越、符号链接/Windows reparse point 越界；不要自动联网抓取模型给出的 URL。LoopX 的相对路径校验与结果 descriptor 可借鉴，但其文档列出的部分安全文件导出仅限已验证的 POSIX 路径；Windows 需单独验收。[LX04、LX06、LX07]

业务验收例如：记录当前提交、测试命令的可信执行回执、退出状态、结果摘要与受检产物摘要；由审查方判断质量，再由用户批准阶段。哈希使证据更易核对，不能“杜绝模型虚报成功”。
