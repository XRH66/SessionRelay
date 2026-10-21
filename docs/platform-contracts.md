# 平台契约：Codex / DSH / Antigravity；ChatGPT 候选（P0）

本文回答一个比“能不能发消息”更严格的问题：**能不能把一次投递与它的完成结果稳定关联起来**。结论按证据分级书写，不把文档、exe 或一次成功发送当成完整能力。ChatGPT 普通 Chat 单列为候选附录，不计入三平台 P0 验收，也不进入 P1。

## 0. 结论摘要

| 平台 | 会话枚举 | 投递 | 完成识别 | 投递↔结果关联 | 外部人工输入 | P0 判定 |
|---|---|---|---|---|---|---|
| Codex | 协议有 thread/list，未实测 | queue 已由用户/GPT 实测 | 协议有 turn/completed，未实测 | 候选（messageId/clientId），未验证 | 候选（clientId 对比），未验证 | 部分验证，P1 需打通 app-server 入口 |
| DSH | CLI 无 listSessions；其他宿主/MCP 枚举未验证 | start/send 已验证 | result.state 可判定 | **原生 request_id** | 无直接信号 | 可用于 P1；会话选择仍缺正式入口 |
| Antigravity | 官方 CLI 的 --conversation 是寻址参数，不是枚举；本机未安装 | 本机无稳定入口 | 官方 CLI 每轮 result，本机未验证 | 官方 CLI conversation_id，本机未验证 | 未验证 | **阻塞**，不能声称支持 |
| ChatGPT 普通 Chat（候选） | 未发现外部枚举入口 | 无通用外部接口；插件组件桥为候选 | 未验证 | 未验证 | 未验证 | 候选，默认禁用；不进入 P1 |

## 1. 统一适配器契约

~~~ts
export interface PlatformAdapter {
  readonly platform: PlatformId;
  capabilities(): Promise<PlatformCapabilities>;
  listInstances(): Promise<PlatformInstance[]>;
  listSessions(instanceId: string): Promise<SessionRef[]>;
  inspectSession(ref: SessionRef): Promise<SessionSnapshot>;
  submit(ref: SessionRef, task: SubmitTask, options: SubmitOptions): Promise<SubmitReceipt>;
  waitForResult(ref: SessionRef, receipt: SubmitReceipt, signal: AbortSignal): Promise<PlatformResult>;
  openConversation(ref: SessionRef): Promise<void>;
}

export interface PlatformCapabilities {
  listSessions: "native" | "manual" | "unsupported";
  submit: "native" | "manual";
  completionSignal: "event" | "poll" | "manual";
  resultCorrelation: "native" | "candidate" | "unsupported";
  detectExternalHumanInput: "native" | "partial" | "unsupported";
  approve: "native" | "unsupported"; // 指平台内执行级审批，不代表 SessionRelay 业务级协调审批
}
~~~

- SessionRef 必须包含 platform、instanceId、sessionId；标题只做展示。
- SubmitReceipt 至少包含 requestId、platformMessageId、platformTurnId（可空）、acceptedAt、rawReceipt。
- PlatformResult 至少包含 state、summary、artifacts、rawRef、correlationProof。
- capabilities 必须严格反映真实能力，显式声明“不支持/仅手动/仅候选”，UI 与调度器依据 capabilities 处理分支，不得假设全平台一致。
- 审批明确分流：`approve` 仅表示平台内部执行级工具/动作审批支持情况；SessionRelay 业务级协调审批（阶段门控、人工干预、管理方指令）由宿主客户端状态机统一接管，不依赖平台能力。

## 2. 证据分级

- **官方公开**：厂商文档、公开协议、CLI 帮助。
- **本机私有**：本机安装、本机生成的 schema、本机 SQLite/日志元数据、私有 shim。
- **用户/GPT 实证**：在真实会话中执行过并观察到结果。
- **本轮实测**：本次 P0 实际执行并记录输出。
- **仅推断/未验证**：协议存在但未连接、未跑通、未观察完成事件。

## 3. Codex

### 3.1 本机入口与版本

- CLI：C:\Program Files\WindowsApps\OpenAI.Codex_26.1016.6573.0_x64__2p2nqsd0c76g0\app\resources\codex.exe
- 版本：codex-cli 0.163.0-alpha.11（本轮 --version 实测）
- 运行中的 codex 进程：位于 Codex 桌面应用的本机安装目录，具体路径因安装方式而异。
- 桌面 App：ChatGPT.exe 多进程（WindowsApps 安装）
- 命名管道：codex-ipc 存在；另有 codex-app-tools、codex-browser-use、codex-computer-use。用途与协议未验证，本轮未连接。

### 3.2 已知可用的投递（用户/GPT 实证）

~~~text
codex queue --thread <SESSION_UUID_OR_EXACT_NAME> --message <TEXT>
~~~

- 已在用户指定的真实桌面会话实测：CLI 返回 messageId，当前轮结束后，排队消息唤醒下一轮并得到回复。
- 本轮**没有**再次向任何 Codex 会话投递；该证据属于用户/GPT，不伪称本轮测试。
- queue 帮助没有 --json/--wait，也没有公开的 receipt schema；messageId 的精确 JSON 外形未在本轮复核。
- queue 支持 --remote <ADDR>（ws://、wss://、unix://、unix://PATH）与 --remote-auth-token-env；实际端点、认证与传输未验证，不能据此推断桌面 queue 与 daemon 共用或使用不同底层传输。

### 3.3 app-server 协议（本机生成，experimental）

用 codex app-server generate-json-schema 生成了协议（临时 4.1MB，已清理；方法清单保留在 evidence）。关键对象：

- 会话枚举：thread/list（archived、cursor、cwd、excludedThreadIds、limit、modelProviders、searchTerm、sectionId、sortDirection、sortKey、sourceKinds、useStateDbOnly）、thread/loaded/list、thread/read、thread/turns/list、thread/items/list。
- 会话恢复：thread/resume；另有 thread/start、thread/fork、thread/archive、thread/delete。
- 投递：turn/start（input[]、clientUserMessageId、approvalPolicy、sandboxPolicy、outputSchema、parentTurnId/rootTurnId 等）；turn/steer（expectedTurnId 必须匹配当前活动 turn）；turn/interrupt。
- 完成事件：turn/started、turn/completed（threadId + turn）；turn.status 枚举 completed/interrupted/failed/inProgress；turn 含 id、items、completedAt、durationMs、error。
- 状态事件：thread/status/changed（active/idle/systemError/notLoaded，active 带 activeFlags）、thread/queue/changed（只有 threadId）、thread/readState/changed（只表示未读位置）。
- 审批/用户输入：item/tool/requestUserInput、item/commandExecution/requestApproval、item/fileChange/requestApproval、item/permissions/requestApproval、serverRequest/resolved。
- 用户消息项：ThreadItem 的 userMessage 分支含 id、clientId（nullable）、content；clientId 是关联我方投递的候选字段。

这些证明协议层**具备**会话枚举、完成事件、审批请求与用户输入的稳定对象；**不证明** SessionRelay 已经能连上，也不证明桌面 App 正在使用同一协议。

### 3.4 完成与关联

- 完成识别候选：turn/completed（首选）；thread/status/changed 的 idle 可作辅助。
- 结果内容：turn.items 中的 agentMessage；或 thread/turns/list、thread/items/list 回读。
- 关联候选：turn/start 的 clientUserMessageId → UserMessage.clientId；或 queue 返回的 messageId → userMessage.id/clientId。
- **未验证**：queue 是否把 messageId 写入 clientId；桌面端人工消息是否总带 clientId；turn/completed 是否会在 SessionRelay 的连接上实时到达。
- 独立 app-server（stdio/ws）只是未验证候选，**不是已可接入现有桌面对话的方案**。连接/恢复失败时必须停下，不允许静默换成新会话或克隆历史。结果读取与关联仍是 P1 前置缺口。
- 外部人工输入：若我方每条投递都带唯一 clientUserMessageId，而桌面人工消息没有对应 clientId，可做“非我方消息”检测；但这依赖平台保留 clientId，未验证。thread/readState/changed 只表示未读，不能证明是谁输入。

### 3.5 daemon 与 queue 不能混为一谈

- codex app-server daemon version 失败：failed to connect to `%USERPROFILE%\.codex\app-server-control\app-server-control.sock`；套接字操作遇到了一个已死的网络（os error 10050）。该目录不存在。
- 但 queue 在此前真实成功。两条路径结果不同，只能说明当前入口表现不同；不能据此推断底层传输、协议或 daemon 可用性。queue 与 daemon 的传输关系未验证。既不能因为 daemon socket 失败否定 queue，也不能因为 queue 成功声称 daemon 可用。
- P0 未启动 daemon，未探测 codex-ipc 的写入协议。

### 3.6 风险与失效条件

- app-server 协议标注 experimental，版本升级可能改字段/方法。
- 桌面 App 与独立 app-server 可能同时持有线程状态；恢复已有线程时的锁/冲突未验证。
- 远程 ws 需要 capability-token 或 signed-bearer-token，配置方式未验证。
- queue 的 messageId 与 turn 的映射未验证，不能承诺 exactly-once。
- 版本变化（CLI/桌面 App）后，本矩阵的入口与协议都要重验。

## 4. DSH

### 4.1 入口

- CLI：本机受控 DeepSeek Harness 适配器（本轮读过源码）
- 传输：现有 3080 MCP Web listener；认证读取现有 Codex 配置，仅内存使用，不输出。
- 命令（CLI 帮助实测）：
  - start --cwd <path> --task-file <file> [--permission <preset>]
  - send --session <id> --task-file <file> [--steer]
  - status --session <id>
  - result --session <id> --request <id> [--wait-ms 20000]

### 4.2 会话与关联

- start 用 randomUUID 生成 request_id，调用 MCP session_start（cwd、prompt、permission_preset、request_id），返回 session_id + request_id。
- send 同样生成 request_id，delivery 为 queue 或 steer，返回 session_id + request_id。
- result 必须带 request_id，返回 state、reason、final_message、diagnostics；这是**原生的 request_id 关联**。
- CLI 没有 list 命令；源码只调用 session_start/session_send/session_status/turn_result 四个 MCP 工具。这只能证明**该 CLI 缺入口**；宿主/MCP 的其他枚举接口本轮未验证，不能写成“DSH 无会话枚举”。
- 手填 session id 只作临时诊断，不取代最终“选择已有会话”的产品需求；正式实现仍需会话选择入口或明确阻塞。
- session id 形状：`session-<UUID>`。

### 4.3 状态与审批

本轮只读执行：

~~~text
node <DSH_CLI> status --session <DSH_SESSION_ID>
~~~

返回（脱敏，无凭据）：

~~~json
{"session_id":"<DSH_SESSION_ID>","agent_status":"running","head_seq":403,"turn":{"turn":1,"open":true,"reason":null},"queue_count":0,"pending_approvals_count":0}
~~~

- status 提供 agent_status、head_seq、turn、queue_count、pending_approvals_count，可用于 running/awaiting_approval/needs_attention 的判定。
- 审批只有计数，CLI 没有批准/拒绝动作；UI 只能提示用户去平台处理。
- 外部人工输入：没有直接事件或来源字段；head_seq 变化不能区分人与模型。标 unsupported，必须在 UI 明示。

### 4.4 完成链

- 用户/GPT 既有验证：start/send/result/status 均可用；本次 start 即真实派工。
- 本轮只记录了脱敏后的 start/result 关联结论：receipt 同时包含 session_id 与 request_id，result 按 request_id 回读；原始派工文件不进入公开仓库。
- 失败/中断：result.state + reason 可区分；具体枚举以实际返回为准，本轮未跑完整失败用例。

### 4.5 风险与失效条件

- CLI 没有 listSessions；P1 临时可手填 session id 作诊断，但最终 UI 仍必须有会话选择入口，不能用手填替代。
- 审批只有计数，无法自动批准。
- 认证依赖现有 Codex 配置与 3080 服务；服务未起或配置变化时 CLI 失败。
- 默认 permission preset 为 danger-full-access（CLI 约定），P1 应显式选择更小权限，不能默认放大。
- 会话 ID 生命周期/清理规则未验证。

## 5. Antigravity

### 5.1 官方公开能力（文档，不等于本机可用）

来源：
- https://www.antigravity.google/docs/cli/headless/
- https://www.antigravity.google/docs/cli/commands/resume/

headless 关键点：

- 命令名 agy；-p/--print/--prompt 单次运行；--output-format text|json|stream-json；--input-format stream-json 必须配 stream-json 输出。
- json 信封：conversation_id、status、response、error、duration_seconds、num_turns、structured_output/json_schema、usage。
- stream-json 为 NDJSON：一次 init（conversation_id、cwd、tools、permission_mode 等）、若干 step_update（step_index、state ACTIVE/DONE、step_type、text_delta、tool_name、tool_info、subagent_info、usage）、每轮一个 result。
- 状态值：SUCCESS/ERROR/CANCELED/INTERRUPTED/INVALID/WAITING/RUNNING。
- 续会话：--continue/-c 续最近，--conversation <id> 续指定 ID；都是另起进程。
- --print-timeout 默认 5m；headless 默认 request-review，无法交互审批会 soft-deny；可配置 permissions.allow。
- resume 文档：Session Picker 可把桌面 App 对话 import 到 CLI；官方原文是 “The CLI clones the history, context, and tool trajectories into your terminal session.” **这是克隆，不是控制原桌面会话。**

### 5.2 本机实际

- 桌面 App：Antigravity 2.24.0.0（文件版本信息实测），Electron 安装；路径因用户安装位置而异。
- resources/bin 只有 language_server.exe（约 171MB）与 webm_encoder.exe；没有 agy。
- Get-Command agy/antigravity：当前 PATH 未发现命令。
- `%USERPROFILE%\.gemini\antigravity-cli` 不存在；已检查的目录中也未发现官方 CLI 的 cache/last_conversations.json。
- 以上只能说明“当前 PATH 与已检查目录未发现”，不等于机器任何位置都没有 CLI。
- `%USERPROFILE%\.gemini\antigravity` 下有 conversation_summaries.db、conversations/、bin/、builtin/、lib/ 等桌面 App 数据；这是本机私有存储，不是公开接口，本轮未读取会话正文。
- Antigravity 安装目录内的 `agentapi.bat` 会调用 `language_server.exe agentapi %*`；这是本机私有 shim，未运行、未验证，不能当作稳定入口。
- 本机另有一个独立环境工具进程，与 SessionRelay 无关；P0 未把它当作平台适配器。

### 5.3 判定

- 官方 agy CLI 若安装并认证，可能具备 headless 投递/每轮 result；但本机当前 PATH 与已检查目录未发现，P0 未验证。
- --conversation 是会话寻址/续接参数，不是会话枚举；不能把它当作会话列表能力。
- 即使安装，resume 的 import 是克隆历史，仍不能控制原桌面会话；产品里必须区分“克隆后接管新 CLI 会话”和“控制原桌面会话”。
- 本机桌面 App 没有已发现的稳定控制接口；agentapi shim 未验证。
- **P5 阻塞。** 当前只能提供手动导入/复制方案，不能声称 Antigravity 已支持。

## 6. ChatGPT 普通 Chat（候选，默认禁用）

这一节不是三平台验收的一部分，也不进入 P1。只依据官方公开资料评估，未在本机安装插件、未运行浏览器自动化、未创建定时任务、未发送探测。

- Chat、Work、Codex 是分开的产品面；工具内部能看到历史，不等于外部程序有枚举/读取/投递接口。
- Conversations API 只保存 **API 创建** 的对象，不能接管 ChatGPT 侧已有聊天；API conversation_id 不能替代已有对话 ID。
- 插件组件可用 window.openai.sendFollowUpMessage，MCP Apps 可用 ui/message，请求 ChatGPT 追加组件消息；但组件生命周期、后台运行、指定任意旧对话、完成回读与关联都未验证。这是候选桥，不是通用 Chat 发送 HTTP 接口。
- Scheduled tasks 的 Gmail/Slack/GitHub 触发是特定事件能力，不能替代通用本地回调或唤醒任意普通 Chat。
- Enterprise/Edu 合规日志是审计读取能力，不是推理、唤醒或普通聊天接口。

判定：**候选，默认禁用。** 会话枚举、自动投递、后台唤醒、完成识别、投递↔结果关联、外部人工输入、审批、断线恢复全部未验证；不得在 capabilities 或 UI 中写成 native/已支持。详见 [platforms/chatgpt.md](platforms/chatgpt.md)。

## 7. 能力矩阵（明细）

| 能力 | Codex | DSH | Antigravity | ChatGPT 普通 Chat（候选） |
|---|---|---|---|---|
| 本机入口 | CLI 0.163.0-alpha.11 + 桌面 App | dsh-cli.mjs + 3080 MCP | 桌面 App 2.24.0；无 agy | 已有桌面/网页产品；本机无已验证外部入口 |
| 会话枚举 | thread/list 协议（未实测）；CLI agents 仅 TUI | CLI 无 listSessions；其他宿主入口未验证 | 官方 --conversation 仅寻址，不是枚举；未安装 | 未发现外部枚举入口 |
| 会话标识 | thread UUID（如 `<CODEX_THREAD_UUID>`） | `session-<UUID>` + request UUID | conversation_id（官方文档） | 未验证；API conversation 不等于已有 Chat |
| 投递 | codex queue（用户/GPT 实证） | start/send（已验证） | 无稳定入口（官方 CLI 未安装） | 无通用外部接口；插件组件桥候选 |
| receipt | messageId（用户/GPT 接续记录）；完整 JSON 未保留 | request_id（原生） | result 事件（官方文档） | 未验证 |
| 完成识别 | turn/completed（协议） | result.state（CLI） | 每轮 result（官方文档） | 未验证 |
| 结果关联 | clientUserMessageId/clientId 候选（未验证） | **request_id 原生** | conversation_id + result（未验证） | 未验证 |
| 外部人工输入 | clientId 对比候选（未验证） | 无直接信号 | 未验证 | 未验证 |
| 审批 | requestApproval 协议（未验证） | pending_approvals_count（只读） | 官方 headless soft-deny；无审批动作 | 未验证 |
| 稳定程度 | 中（协议 experimental） | 中高（CLI 已验证） | 低（本机无入口） | 候选，默认禁用 |

## 8. P1 建议与前置阻塞

P1 只做最小 Codex↔DSH，不做三平台全通。Antigravity 与 ChatGPT 普通 Chat 都不进入 P1。建议：

1. DSH 侧直接用现有 CLI：start/send/status/result，request_id 作为关联主键；手填 ID 只作临时诊断，最终仍需会话选择。
2. Codex 侧把独立 app-server（stdio 或本机 ws）作为**未验证候选**；使用 thread/resume + turn/start 只是待验证方案，不能声称已可接入现有桌面对话。
3. 每次 Codex 投递设置唯一 clientUserMessageId，等待 turn/completed；若实时事件不可靠，用 thread/turns/list 回读。
4. 先验证“app-server 能否恢复桌面已存在线程且不干扰当前会话”，再谈自动投递；失败不得静默新建/克隆。
5. DSH 的 CLI 没有 listSessions；P1 临时手填 ID，但正式 UI 必须有会话选择入口或明确阻塞，不能伪造枚举。

前置阻塞（允许带入 P1，不阻塞 P0 收尾）：

- Codex 独立 app-server 的实际连接/恢复未实测；若与桌面进程冲突，需要用户显式选择新线程或远程端点，不能静默替换。
- DSH 的 CLI 会话发现缺失；其他宿主入口未验证；审批动作缺失。
- Antigravity 不进入 P1。
- ChatGPT 普通 Chat 仅候选、默认禁用，不进入 P1，也不作为 Codex↔DSH 的前置阻塞。

## 9. 明确不做

- 不启动 Codex daemon / remote control 来“让接口可用”。
- 不修改 Codex/DSH/Antigravity 的配置或数据库来强行接入。
- 不把桌面 App 的私有存储当作 API。
- 不把“发送成功”写成“完成关联已验证”。
- 不在 P0 向任何 Codex 会话投递。

## 10. 设计参考

GitHub 参照、固定 commit 源码与取舍见 [references.md](references.md)；源码为阅读级证据，README/文档也为阅读级证据；均未安装、未编译、未实测。
