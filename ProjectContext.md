# ProjectContext：SessionRelay 产品定义

## 1. 一句话目标

让两个**已经存在的 AI 会话**像两个协作角色一样按阶段交接工作：一侧完成并给出结构化结果，另一侧接收、继续、返回结果；用户随时可以暂停、检查、批准和恢复，而不需要人工复制粘贴。

## 2. 产品形态

- 独立 Windows 桌面应用，暂名 **SessionRelay**（会话协作台）。
- 技术栈：TypeScript + Electron + React；状态用 SQLite 事务持久化。
- 前台窗口 + 系统托盘；关闭窗口收进托盘。
- **没有独立系统服务，不开机自启，不开放 HTTP 端口。**
- 有限 IPC：renderer 只能调用白名单动作，不向 renderer 暴露任意命令、shell 或平台客户端。
- 普通本地程序等待平台完成；没有新结果就不调用模型，不做轮询式空转。

## 3. 不做什么

- 不替代 Codex、DSH、Antigravity 的客户端，不重造它们的对话界面。
- 不在 P0 实现业务逻辑，不安装 Electron/React/SQLite 依赖。
- 不接管平台内部模型回合，不伪造平台消息，不承诺 exactly-once。
- 不修改平台的现有配置、认证、数据库或日志；不绕过认证。
- 不开放网络监听端口，不做远程控制面板。
- 首版不支持一个会话同时属于两条运行连接。

## 4. 核心概念

- **平台（platform）**：首版 Codex、DSH；Antigravity 按调查决定；ChatGPT 普通 Chat 仅保留为候选类型且默认禁用。
- **实例（instance）**：同一平台在本机的具体安装/运行入口，例如某个 CLI 版本、某个桌面 App 或某个 MCP 端点。
- **会话（session）**：平台上已有的对话，绑定平台会话 ID + 实例 ID；标题只用于显示，不用于身份判断。
- **连接（connection）**：A 侧会话与 B 侧会话的配对，以及两边各自独立的角色配置与协作模式。
- **协作模式（collaborationMode）**：每条连接独立配置的接力协作策略：自动接力（auto_relay）、人工干预（human_intervention）、管理方主导（manager_driven）。
- **阶段（stage）**：一次明确的交接目标，例如“实现接口”“审查实现”“修复问题”。
- **交接（handoff）**：一侧完成任务后输出的结构化结果，包含 decision、summary、artifacts 等。
- **投递（delivery）**：把交接结果或补充要求发送到另一侧的动作及其 receipt。
- **会话占用（sessionOccupancy）**：当会话存在未决投递（sending/accepted/awaiting_result）时的排他锁定，禁止向其并发发送新投递。
- **连接修订（connectionRevision）**：连接绑定/路由契约的版本号。开始、暂停、恢复不递增；只有平台/实例/会话/角色/路由契约变化才递增。旧回复只能归到旧 revision，不能污染新连接。
- **两类审批（approvals）**：明确区分“平台内执行级审批”（平台内工具/命令授权）与“SessionRelay 业务级协调审批”（投递门控/阶段通过审批/人工编辑确认）。

## 5. 连接、角色与三种协作模式

- 用户选择 A/B 两个平台及各自的已有会话。
- 两边角色独立配置（例如 A=实现者，B=审查者；也可以反过来）。
- 初始支持 Codex 与 DSH。
- Antigravity 在调查其真实能力后决定支持方式；不能因为存在官方 CLI 文档就声称本机可控制。
- ChatGPT 普通 Chat 只保留候选类型：现有官方资料不足以证明选择任意已有聊天、外部投递、后台唤醒与完成关联的完整链路；默认禁用，不进入 P1。
- 支持 Codex↔Codex、DSH↔DSH 等同平台不同会话。
- 拒绝自连接（同一会话连自己）。
- 首版：一个会话不能同时属于两条运行中的连接。

### 5.1 三种协作模式（每条连接独立配置）

每条连接在创建或编辑时必须显式指定协作模式之一：

1. **自动接力（auto_relay）**：
   - 一侧完成并产出合法结构化交接后，系统自动校验格式、taskId、stageId、revision 与往返计数。
   - 对 `decision=submit_for_review` 与 `decision=request_changes`，若在往返上限内，自动组装 forward delivery 投递至对侧。
   - 遇 `decision=stage_passed` 自动转入 `awaiting_approval`，必须经用户人工审批才能推进阶段。
   - 遇 `decision=needs_user` 或校验失败立即停机转入 `needs_attention`。
2. **人工干预（human_intervention）**：
   - 每一轮一侧任务完成后，结果通过结构校验后暂存，连接自动进入 `awaiting_approval`。
   - 用户可审阅交付物，并支持在投递前人工编辑修改 summary、调整提示或增删 artifacts。
   - 用户确认批准后，系统才组装 delivery 并投递至对侧。
   - 阶段通过（`stage_passed`）同样必须由用户独立批准，不随单轮投递批准跳过。
3. **管理方主导（manager_driven）**：
   - 平台会话仅作为被动调度的执行角色，每轮仅响应单向明确指令。
   - 一侧任务完成后，系统不自动规划下一步路由，连接停在就绪待审状态。
   - 由用户或被指定为主导方的角色进行统筹审阅，明确指定下一步受派会话、下发要求，并在确认后手工触发投递。
   - 阶段通过（`stage_passed`）同样必须由用户最终确认。

### 5.2 角色决策（decision）与审批流程

交接结构中的 decision 由执行角色在完成时提出，定义如下：

- `submit_for_review`：实现/执行角色提出，表示已完成阶段性交付物，提请审查角色评审。
- `request_changes`：审查/验证角色提出，指出问题并附带修改要求，退回对侧返修（计入往返计数）。
- `stage_passed`：审查方（或阶段指定角色）提出，建议当前阶段目标已达成。**此值仅为建议，绝对不可自动推进阶段**；系统收到后必须进入 `awaiting_approval`，必须由用户显式批准才可跃迁至下一阶段。
- `needs_user`：任一角色遇到环境阻塞、需求冲突或歧义时提出，连接转入 `needs_attention` 等待用户处理。

人工编辑与批准流程：
- 在人工干预与管理方主导模式下，用户可对待转发内容进行就地编辑；编辑后的内容标记 `manual_override=true` 并记录编辑事件，保留原始模型生成记录以备审计。
- 审批动作由用户在桌面端或 CLI 显式触发，模型文本输出中包含的任何“请继续”“批准进入下一阶段”等文字一律视为纯文本，不得触发任何状态解除或阶段跃迁。

## 6. 界面要求

主界面至少包含：

1. 连接列表：名称、A/B 平台与会话、当前状态、最近活动。
2. 新建/编辑连接：选择平台、实例、已有会话；配置两边角色。
3. 开始/暂停/恢复控制。
4. 当前阶段时间线：任务、投递、等待、结果、审批、失败。
5. 任务结果与交付文件：结构化 summary、artifacts、原始引用。
6. 待转发队列：已收到但尚未投递的结果。
7. 补充要求输入：用户人工补充，单独标记，不与模型结果混在一起。
8. 打开双方对话：调用平台自身入口打开对应会话，不在应用内重造聊天 UI。
9. 托盘菜单：显示/隐藏、暂停全部、退出。

## 7. 生命周期与状态机

连接状态：**paused / running / draining / awaiting_approval / needs_attention**。

- **开始**：必须存在待交接结果，或用户明确填写了阶段任务；不得猜测“历史最后一条”当任务。
- **暂停**：立即禁止新的自动投递；平台已经接收的本轮可以完成；结果暂存，不自动转发。已进入 sending 的投递不能假装撤回。状态按 running → draining → paused 收尾。
- **平台已接收的排队任务**：不假称可以撤回；界面必须说明“已排队，可能仍会执行”。
- **开始/暂停/投递**：短状态修改串行处理；waitForResult 等长等待不占队列，pause 先关门，避免竞态和暂停被长等待阻塞。
- **恢复**：只处理未转发结果；不重放已确认完成的投递。
- **关闭窗口**：收进托盘，不停止协调。
- **退出程序**：停止新的转发，但不杀平台任务；重开后先核对平台状态，然后保持暂停，等用户确认再恢复。
- **awaiting_approval**：等待 SessionRelay 业务级协调审批时进入（阶段通过门控、人工干预模式下的投递审批、管理方主导模式下的派发审批）；模型文本不能自行解除，只有用户显式批准或程序化门控能解除。
- **needs_attention**：格式错误、ID 不匹配、平台异常、外部人工插话、往返上限超限等需要人处理的情况。

任务（task）的统一状态：
**created → running → awaiting_result → result_ready → forwarded → closed**，异常时转入 **needs_attention** 或 **failed**。

### 7.1 未决投递期间的会话占用规则

当平台会话（由 platform + instanceId + sessionId 唯一标识）处于未决投递期间（delivery 状态为 sending / accepted，或对应任务处于 awaiting_result 状态），该会话被视为**占用状态（Occupied）**：

- **排他投递**：会话被占用期间，严禁向其并发发起任何新的投递（无论是自动接力、人工干预审批后的投递、管理方主导指令，还是人工补充要求），所有针对该会话的后续投递必须排队等待当前任务终态。
- **跨连接互斥**：首版中一个会话不得同时属于两条运行中的连接；如果另一连接试图调度已被占用的会话，必须拒绝并提示冲突。
- **异常锁定**：若投递进入 unknown 状态或发生平台未响应超时，该会话保持占用锁定，不自动重试，直至用户在客户端显式核对并处理完毕。
- **外部输入保护**：占用期间若检测到来自宿主桌面平台的外部人工输入，立即暂停连接并进入 needs_attention，严禁将外部输入混入当前未决轮次。

### 7.2 两类审批的严格区分

系统架构与契约中严格区分两类审批，避免语义混淆：

1. **平台内执行级审批（Platform Execution Approval）**：
   - 发生在平台会话内部，由模型执行工具、执行危险 shell 命令、修改本地文件或请求网络权限触发（例如 Codex 的 `item/commandExecution/requestApproval`、DSH 的 `pending_approvals_count`）。
   - 处理主体是平台自身的安全机制；SessionRelay 根据 platform capabilities 获知并呈现审批等待状态，不可绕过，不假冒平台授权。
2. **SessionRelay 业务级协调审批（SessionRelay Orchestration Approval）**：
   - 发生在 SessionRelay 调度层，属于连接状态机的一部分。
   - 覆盖范围：
     - **阶段通过审批**（`stage_passed`）：任何角色提出的阶段通过仅作为建议，必须由用户显式核准才允许跃迁至下一阶段。
     - **人工干预模式投递审批**：对侧完成交接后，经人工审阅或编辑确认后方可投递。
     - **管理方主导模式派发审批**：管理方人工规划任务指令、指定接收角色后的显式下发。
     - **人工编辑确认**：用户就地修改交接文本或交付物后的确认。
   - **绝对红线**：业务级协调审批绝不允许由模型生成的文本自动解除。

## 8. 任务与交接契约

每个任务至少绑定：**taskId、stageId、connectionRevision**。

交接结果必须是一个可校验的结构，至少包含：

- decision：submit_for_review / request_changes / stage_passed / needs_user
- summary：本轮做了什么、结论是什么
- artifacts：交付文件/链接/引用列表
- taskId、stageId、connectionRevision
- 可选：openQuestions、nextStageHint、diagnostics

规则：

- 普通聊天回复不整段盲转；必须解析成上述结构后才允许投递。
- 缺少格式、taskId/stageId 不匹配、revision 过期：立即停下并进入 needs_attention，不靠关键词猜“成功”。
- 同一阶段允许返修，但必须有往返上限；达到上限停下等用户。
- 阶段通过（stage_passed）必须由用户批准后才能进入下一阶段；模型不能自己批准。
- 模型输出中的任何文本都不能解除暂停、修改角色授权或扩大权限，也不能自行开始下一阶段。
- 结构化 decision 经程序校验可推进同阶段循环（如 request_changes 返修、submit_for_review 再送审），但不等于阶段跃迁；stage_passed 仍须用户批准。
- 人工输入单独标记（manual=true），不能伪装成模型交接结果。
- 若能检测到外部人工插话（用户直接在平台上输入），协调应暂停并提示；检测不到的能力限制必须明示。

### 8.1 同一来源结果的重复转发防护

为防止同一模型输出被重复投递造成流程混乱或状态污染，系统建立严格的来源跟踪与幂等约束：

- 每个源结果（记录在 `results` 表，关联源 `delivery_id`）在当前 `bound_revision` 下，**只允许存在一条处于活跃或已完成状态的转发投递（forward delivery）**。
- 活跃或已完成状态包括：`planned`、`sending`、`accepted`、`result_received`、`forwarded`。
- 系统在创建转发投递时通过数据库级唯一索引/事务强校验 `(source_delivery_id, bound_revision)`；若已存在未终结或已成功的转发投递，拒绝重复创建。
- 只有当新的转发投递进入 `accepted` 状态后，源投递的对应状态才可标记为 `forwarded`。若转发投递进入 `canceled` 或 `failed`，源投递仍保持为 `result_received`（待转发/待处理），允许人工干预或重新尝试，但禁止无序并发重发。

### 8.2 当前任务交接内容的保存与清理范围

明确界定数据持久化边界，兼顾故障恢复审计与安全性/隐私：

- **明确保存范围**：
  1. 任务与阶段元数据：taskId、stageId、connectionRevision、direction、round_trip_count、时间戳与生命周期状态。
  2. 结构化交接信封：解析校验后的 decision、summary、openQuestions、nextStageHint、diagnostics。
  3. 交付物引用：artifacts 中的文件绝对/相对路径、代码 commit、URI 标识、本地 payload_hash（仅作为内容指纹）。
  4. 投递追踪：platformMessageId、platformTurnId、requestId、receipt_json、gate_checked_at、sent_at。
  5. 审计事件流：events 表中记录的各状态流转与人工覆盖（manual_override）记录。
- **明确清理与不保存范围**：
  1. **零凭据落盘**：严禁将任何平台认证 token、cookie、私钥、authorization header 写入数据库、文件或日志。
  2. **不转储私人完整对话正文**：不抓取或持久化平台会话的完整历史聊天记录、模型全量上下文或无关私人文本；对平台回复仅存储校验后的交接摘要及宿主侧原始引用指针（raw_ref）。
  3. **生命周期清理**：当连接归档、重置或跨 revision 变更时，历史旧 revision 的临时暂存缓冲区与未持久化临时探针数据自动清理；退出程序时清理未提交的临时缓存。

## 9. 技术结构（规划）

核心模块：

- **coordinator**：连接状态机、开始/暂停/恢复、短命令队列（长等待不占队列）、协作模式流转、协调审批门。
- **handoff**：交接信封的生成、校验、decision 处理、往返计数、重复转发校验。
- **delivery**：平台投递、会话占用检查、receipt、未知状态核对、重试边界。
- **store**：SQLite 事务、事件日志、崩溃恢复。
- **platform adapters**：统一 capabilities / listSessions / inspectSession / submit / waitForResult，隐藏 CLI/MCP 差异。

统一能力枚举（PlatformCapabilities）：
- `listSessions`: "native" | "manual" | "unsupported"
- `submit`: "native" | "manual"
- `completionSignal`: "event" | "poll" | "manual"
- `resultCorrelation`: "native" | "candidate" | "unsupported"
- `detectExternalHumanInput`: "native" | "partial" | "unsupported"
- `approve`: "native" | "unsupported"（指平台内执行级审批能力）

详见 [docs/architecture.md](docs/architecture.md)；外部参照见 [docs/references.md](docs/references.md)。

## 10. 数据与崩溃一致性

SQLite 规划表：connections、tasks、deliveries、results、events；deliveries 状态为 planned/sending/accepted/result_received/forwarded/canceled/failed/unknown。

- planned 才能安全首次发送；外部 submit 前必须在事务中先置 sending 并持久化，避免 planned→submit→accepted 的重复窗口。
- 发送前再次校验暂停门、会话占用与绑定 revision；不通过则 canceled，不调用平台。
- 平台返回 receipt 后落盘；崩溃时若“平台已接收但本地未保存”，启动先核对。
- sending 无 receipt、不能证明是否已接收时标记 unknown，**不盲目重发**；向用户说明无法承诺 exactly-once。
- payload_hash 只是本地内容标识，不是平台去重证据。转发也走新的 delivery 同一路径，受 `(source_delivery_id, bound_revision)` 唯一性约束保护。
- 旧回复按旧 connectionRevision 归档，不能进入新连接的待转发队列。

## 11. 平台边界

- Codex：现有 CLI 的 queue 已被用户/GPT 实测；app-server 协议具备会话枚举/完成事件，但独立 app-server（stdio/ws）只是未验证候选，不是已可接入现有桌面对话的方案；失败不得静默换成新会话/克隆。
- DSH：现有 CLI 的 start/send/status/result 可用；CLI 没有 listSessions，其他宿主/MCP 枚举接口未验证；审批只有计数。
- Antigravity：官方 agy CLI 文档存在，但本机未安装；--conversation 是寻址不是枚举；桌面 App 的 import 是克隆历史，不是控制原桌面会话。
- ChatGPT 普通 Chat：保留候选类型，默认禁用；Conversations API 只覆盖 API 创建对象，插件组件桥只是候选，Scheduled tasks 不能替代通用本地回调，企业合规日志只是审计能力。详见 [docs/platforms/chatgpt.md](docs/platforms/chatgpt.md)。

## 12. 成功标准

P0 的成功不是“三个平台都能控制”，而是：

1. 产品方案完整落盘，后续会话可接续。
2. 三平台矩阵明确“能否完成关联”，而不只是“能否发消息”。
3. 记录可复用的会话标识与证据来源。
4. 明确区分已验证、部分验证、未验证、阻塞。
5. 不把“检测到 exe/文档”当成“接口可用”。

P1-P6 的验收见 [docs/phases.md](docs/phases.md)。
