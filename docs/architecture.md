# 架构：SessionRelay 的实现结构与原理

本文描述 P1 以后要实现的真实结构。P0 只落盘设计，不写业务代码。独立验收修订后，投递状态、串行边界、绑定 revision 与平台候选路径以下文为准。

## 1. 进程与信任边界

~~~
┌──────────────────────────────────────────────────────────┐
│ Electron main（Node，唯一可信层）                         │
│  coordinator · handoff · delivery · store                │
│  platform adapters（Codex / DSH / Antigravity）           │
│  托盘 · 单实例锁 · 退出清理                               │
└───────────────▲──────────────────────────┬───────────────┘
                │ 白名单 IPC               │ 子进程/本机 socket/MCP
┌───────────────┴───────────────┐          ▼
│ Renderer（React，仅 UI）       │   Codex CLI / app-server
│ 不接触凭据、不执行任意命令     │   DSH CLI / 3080 MCP
└───────────────────────────────┘   Antigravity（受 capability 门控）
~~~

- renderer 只能通过 preload 暴露的白名单通道调用主进程：读取视图、开始/暂停/恢复、提交人工补充、打开平台对话、导出结果。
- 不提供通用 shell、文件系统、HTTP 请求或平台 CLI 直通。
- 认证材料只在主进程内存中使用；不进 renderer、不写 SQLite、不进日志。
- 应用不监听 HTTP 端口；与平台的通信是**出站**的本地 CLI/socket/MCP 调用。

## 2. 核心模块

### coordinator

- 持有连接状态机与**短命令队列**。短命令指 start/pause/resume、创建投递、收 receipt、收结果、校验与状态写入等有界操作。
- 短命令串行执行，但单个命令不得等待平台长任务。waitForResult 在队列外运行；它完成时只把“结果到达”这一短事件入队，不能占住队列让 pause 排队。
- pause 先原子关闭“允许新投递”的门并置 draining，不等待 waitForResult 或网络调用；已进入 sending 的投递按在途处理，暂停不能撤回平台已接收的任务。
- 每次真正 submit 前，在同一事务内重新读取暂停门、会话占用锁、连接状态与绑定 revision；任一不通过则把 delivery 置 canceled，不调用平台。
- start/resume 不改变绑定 revision；只有平台/实例/会话/角色/路由契约变化时才递增 revision。
- 调度**三种协作模式**（每条连接独立配置）：
  1. **自动接力（auto_relay）**：收到合法结果且 decision 为 `submit_for_review` 或 `request_changes` 时，在未达往返上限前提下自动创建 forward delivery 发往下游；遇 `stage_passed` 进入 `awaiting_approval` 等待用户批准阶段通过；遇 `needs_user` 进入 `needs_attention`。
  2. **人工干预（human_intervention）**：一侧完成后暂存结果，连接进入 `awaiting_approval`。允许用户在投递前审阅并就地编辑 summary/artifacts（标记 `manual_override=true`），确认批准后才发起对侧投递；`stage_passed` 同样必须独立用户批准。
  3. **管理方主导（manager_driven）**：一侧完成后连接进入就绪待审状态，不自动推导接收方，由用户（或主导角色）检视结果后手工指定下一目标会话、输入明确指示，再触发下发；`stage_passed` 仍必须经用户最终核准。
- 严格区分**两类审批**：
  - **平台内执行级审批**：模型在平台侧触发的工具/命令/修改确认，由 platform adapter 报告并由平台机制处理；
  - **SessionRelay 业务级协调审批**：包括阶段通过（`stage_passed`）门控、人工干预投递批准、管理方主导派发批准及人工编辑确认。模型文本绝对不能解除业务级审批门。

### handoff

- 定义交接信封 schema，生成与校验。
- 解析平台回复中的结构化结果；不是“看到成功二字”就通过。
- 校验 taskId、stageId、connectionRevision 与 decision 白名单。
- 明确角色决策定义：
  - `submit_for_review`：实现方提请审查。
  - `request_changes`：审查方退回返修。
  - `stage_passed`：提议阶段完成（仅为建议，必须经用户批准）。
  - `needs_user`：报告阻断并请求人工协助。
- 人工编辑流程：在人工干预与管理方主导模式下，允许用户就地修改交付摘要与补充说明，系统记录编辑事件与覆盖标记 `manual_override=true`，保留底层模型原始回复以备审计。
- 维护同阶段往返计数与上限。
- 失败分类：missing_format、wrong_task_id、stale_revision、invalid_decision、limit_exceeded。
- 模型输出不是控制指令。decision=request_changes/submit_for_review 经程序校验可在同阶段推进返修/送审循环；decision=stage_passed 只进入 awaiting_approval，必须用户批准才进入下一阶段；needs_user 进入待人工处理。模型不能自行解除暂停、改角色/权限或开始下一阶段。

### delivery

- 把“要发什么”先事务持久化为 outbox，再调用平台。
- delivery 从 planned 进入 sending 前必须再次通过暂停门、会话占用检查与绑定 revision 校验；planned 才可安全首次发送。
- **未决投递期间的会话占用**：当某会话存在状态为 `sending`、`accepted` 或对应任务处于 `awaiting_result` 的未决投递时，该会话处于占用排他状态，严禁并发提交新投递；首版禁止其他连接调度该会话。
- **同一来源结果的重复转发防护**：每条来源结果（记录在 `results` 表）在当前 `bound_revision` 下，只允许生成一条有效 forward delivery。系统在数据库事务中通过 `(source_delivery_id, bound_revision)` 约束防重；仅在 forward delivery 进入 `accepted` 后，源 delivery 才置 `forwarded`。
- sending 没有 receipt 时必须与平台核对；不能证明已接收时置 unknown，禁止自动重发。
- 转发不是临时 forwarding 状态：新建一条 kind=forward 的 delivery，走同一 outbox 路径，并用 source_delivery_id 指向来源结果。
- payload_hash 只是本地内容标识，不是平台去重证据。
- 等待结果时使用平台的原生完成信号，不做高频轮询。

### store

- SQLite（WAL 模式）单写者；所有状态变更在事务内完成。
- 事件表追加写，作为时间线与审计来源。
- 启动时执行恢复检查，不自动恢复运行中的连接。
- **交接内容的保存与清理范围**：
  - **保存**：任务状态与元数据、结构化交接信封（summary、decision、artifacts、diagnostics）、外部引用指针（文件路径、URL、hash、commit）、投递 receipts、审计事件流。
  - **不保存 / 清理**：不存储任何凭据、token、cookie；不转储平台完整私人对话正文或大段聊天历史 dump；跨 revision 或连接归档时清理历史临时缓冲区，程序退出时清理未持久化缓存。

### platform adapters

统一接口，平台差异只存在于 adapter 内部：

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
~~~

能力对象必须显式声明“不能做什么”，枚举统一如下：

~~~ts
export interface PlatformCapabilities {
  listSessions: "native" | "manual" | "unsupported";
  submit: "native" | "manual";
  completionSignal: "event" | "poll" | "manual";
  resultCorrelation: "native" | "candidate" | "unsupported";
  detectExternalHumanInput: "native" | "partial" | "unsupported";
  approve: "native" | "unsupported"; // 指平台内执行级审批，不代表协调层审批
}
~~~

- `listSessions`: "native"（平台支持外部列出已有会话） | "manual"（缺少列出会话入口，仅支持手填 ID 临时诊断） | "unsupported"（无外部会话入口）。
- `submit`: "native"（可通过 API/CLI 投递至已有会话） | "manual"（需人工手动复制）。
- `completionSignal`: "event"（具备完成推送通知/事件） | "poll"（需通过状态/结果命令轮询） | "manual"（需人工标记完成）。
- `resultCorrelation`: "native"（平台原生返回 request_id/turnId 关联结果） | "candidate"（存在 clientUserMessageId 等候选字段但待验证） | "unsupported"（无法关联）。
- `detectExternalHumanInput`: "native"（可可靠区分宿主用户直接输入） | "partial"（仅部分信号如未读位置或候选 clientId 对比） | "unsupported"（无法区分外部人工输入）。
- `approve`: "native"（平台支持通过接口接收执行级批准/拒绝） | "unsupported"（平台内审批无法通过接口操作，仅为计数或只读提示）。

UI 与 coordinator 依据 capabilities 决策，不允许在业务层写“Codex 就是这样”的分支。

- waitForResult 由 adapter 在短命令队列之外执行，完成/失败只投递事件。
- listSessions=manual 只表示临时诊断入口（例如手填会话 ID），不取代最终“选择已有会话”的产品需求；正式 UI 仍需会话选择能力，找不到就明确阻塞。
- Codex 的独立 app-server（stdio/ws）只是未验证候选，不是已可接入现有桌面对话的方案。连接失败时不得静默新建会话或克隆历史；必须进入 needs_attention，由用户决定。
- DSH 的 CLI 没有 listSessions，只能说明该 CLI 缺入口；其他宿主/MCP 枚举接口未验证，不能写成“DSH 无会话枚举”。

## 3. 数据模型（SQLite 草案）

~~~sql
CREATE TABLE connections (
  id                 TEXT PRIMARY KEY,
  name               TEXT NOT NULL,
  collaboration_mode TEXT NOT NULL DEFAULT 'auto_relay', -- auto_relay / human_intervention / manager_driven
  platform_a         TEXT NOT NULL,
  instance_a         TEXT NOT NULL,
  session_a          TEXT NOT NULL,
  title_a            TEXT,
  role_a             TEXT NOT NULL,
  platform_b         TEXT NOT NULL,
  instance_b         TEXT NOT NULL,
  session_b          TEXT NOT NULL,
  title_b            TEXT,
  role_b             TEXT NOT NULL,
  state              TEXT NOT NULL,      -- paused/running/draining/awaiting_approval/needs_attention
  revision           INTEGER NOT NULL,   -- connectionRevision
  round_trip_limit   INTEGER NOT NULL,
  created_at         TEXT NOT NULL,
  updated_at         TEXT NOT NULL
);

CREATE TABLE tasks (
  id                 TEXT PRIMARY KEY,   -- taskId
  connection_id      TEXT NOT NULL REFERENCES connections(id),
  stage_id           TEXT NOT NULL,
  revision           INTEGER NOT NULL,   -- bound revision
  direction          TEXT NOT NULL,      -- a_to_b / b_to_a
  objective          TEXT NOT NULL,
  state              TEXT NOT NULL,      -- created/running/awaiting_result/result_ready/forwarded/closed/needs_attention/failed
  round_trip_count   INTEGER NOT NULL DEFAULT 0,
  created_at         TEXT NOT NULL,
  updated_at         TEXT NOT NULL
);

CREATE TABLE deliveries (
  id                 TEXT PRIMARY KEY,
  task_id            TEXT NOT NULL REFERENCES tasks(id),
  platform           TEXT NOT NULL,
  session_ref        TEXT NOT NULL,
  kind               TEXT NOT NULL,      -- handoff/forward/supplement/manual
  source_delivery_id TEXT REFERENCES deliveries(id), -- forward 的来源
  manual_override    INTEGER NOT NULL DEFAULT 0,     -- 是否经人工编辑修改
  payload_json       TEXT NOT NULL,
  payload_hash       TEXT NOT NULL,      -- 本地内容标识，不是平台去重证据
  bound_revision     INTEGER NOT NULL,   -- 创建时固定的 connectionRevision
  attempt_no         INTEGER NOT NULL DEFAULT 1,
  state              TEXT NOT NULL,      -- planned/sending/accepted/result_received/forwarded/canceled/failed/unknown
  failure_code       TEXT,               -- gate_closed/stale_revision/platform_error/session_occupied/...
  platform_message_id TEXT,
  platform_turn_id   TEXT,
  request_id         TEXT,
  gate_checked_at    TEXT,               -- 最后一次发送前门校验
  sent_at            TEXT,
  receipt_json       TEXT,
  created_at         TEXT NOT NULL,
  updated_at         TEXT NOT NULL
);

CREATE TABLE results (
  id                 TEXT PRIMARY KEY,
  delivery_id        TEXT NOT NULL REFERENCES deliveries(id),
  revision           INTEGER NOT NULL,   -- 与 delivery.bound_revision 一致
  state              TEXT NOT NULL,      -- received/validated/rejected
  decision           TEXT,               -- submit_for_review / request_changes / stage_passed / needs_user
  summary            TEXT,
  artifacts_json     TEXT,
  raw_ref            TEXT,               -- 平台侧引用/文件路径，不复制私人正文
  received_at        TEXT NOT NULL
);

CREATE TABLE events (
  seq                INTEGER PRIMARY KEY AUTOINCREMENT,
  connection_id      TEXT,
  task_id            TEXT,
  delivery_id        TEXT,
  type               TEXT NOT NULL,
  payload_json       TEXT,
  created_at         TEXT NOT NULL
);

-- 防重索引：同一来源结果在同一 revision 下仅允许一条未取消/未失败的有效转发
CREATE UNIQUE INDEX idx_deliveries_source_forward
  ON deliveries(source_delivery_id, bound_revision)
  WHERE kind = 'forward' AND state NOT IN ('canceled', 'failed');
~~~

索引至少：connections(state)、tasks(connection_id, state)、deliveries(task_id, state)、deliveries(state, gate_checked_at)、deliveries(platform_message_id)、deliveries(source_delivery_id)、deliveries(session_ref, state)、events(connection_id, seq)。

## 4. 事务性 outbox 与崩溃一致性

handoff、forward、supplement、manual 走同一条投递路径：

1. **事务 P（计划）**：生成 delivery id，写入 state=planned、bound_revision、payload_json、payload_hash、task 状态与 event。此时还没有调用平台。
2. **事务 G（发送前门与占用校验）**：
   - 重新读取连接状态、暂停门、绑定 revision。
   - **会话占用检查**：查询目标 `session_ref` 是否有处于 `sending`、`accepted` 或对应任务处于 `awaiting_result` 的其他未决投递；若已被占用，阻塞或取消当前投递（置 `canceled`，原因 `session_occupied`），严禁并发调用平台。
   - **重复转发防护**：若为 forward 投递，核验 `idx_deliveries_source_forward` 唯一约束，确保未对同一 source_delivery 重复创建活跃投递。
   - 校验 payload 与 task/stage/revision。通过则 planned → sending，记录 attempt_no 与 gate_checked_at。不通过则 planned → canceled，不调用平台。
3. **外部 submit（同一短动作内发起）**：gate 通过并持久化 sending 后，立即发起平台调用；发起动作必须与前面的 G 连续，不能把提交交给一个晚于 pause 的异步回调。调用本身可以异步或阻塞，但短命令队列不等待完整结果；receipt/结果等待在队列外。pause 在 submit 发起前能关门，发起后只能按在途处理。
4. **事务 R（receipt）**：写平台 message/request/turn ID 与 receipt_json，sending → accepted，task → awaiting_result。此时会话仍保持占用状态。
5. **结果到达**：写 result、delivery → result_received、task → result_ready。会话占用解除。
6. **转发（受协作模式门控）**：
   - 在 `auto_relay` 模式：校验合法且 decision 为 `submit_for_review`/`request_changes` 时，事务 F 新建 kind=forward 的 delivery（source_delivery_id 指向来源），回到步骤 1；它 accepted 后，源 delivery 才置 forwarded。
   - 在 `human_intervention` 模式：进入 `awaiting_approval`，待用户审阅/编辑后，由用户批准动作触发事务 F。
   - 在 `manager_driven` 模式：连接就绪待审，由主导方明确指派目标并触发事务 F。
   - 若遇到 `decision=stage_passed`，无论何种模式均进入 `awaiting_approval`，必须用户显式批准阶段跃迁，绝不自动触发 forward。
   - 转发失败/unknown 只影响 forward delivery，源结果仍是 result_received，继续待核对/待转发。

崩溃窗口与处理：

| 崩溃点 | 本地状态 | 启动动作 |
|---|---|---|
| P 之后、G 之前 | planned | 可安全首次发送；重发前仍要重新过门、会话占用与 revision 校验 |
| G 之后、submit 之前 | sending | 不可直接重发；先核对平台，不能证明未接收则 unknown，会话保持占用锁定 |
| submit 中、receipt 未落盘 | sending | 查平台会话/队列；能证明已接收则补 receipt，否则 unknown，保持占用待人工处理 |
| receipt 已落盘、结果未到 | accepted | 继续等待或按平台状态核对，不重发，会话保持占用 |
| 结果已收、未转发 | result_received | 会话占用已释放；放入待转发列表，恢复时只处理这类 |
| 转发 delivery 在 sending | sending（kind=forward） | 与普通投递同样核对；源结果保持 result_received |

**不承诺 exactly-once。** 能承诺的是：发送前必落盘、发送前重过门、receipt 可核对、unknown 不自动重发、重复风险可见。planned 才可安全首次发送；sending 没有 receipt 不能当作没发过。payload_hash 不能证明平台侧唯一。

**交接内容的持久化与清理边界**：
- SQLite 仅持久化协调必需元数据：任务参数、结构化交接（summary/decision/artifacts/diagnostics）、外部文件路径或 commit hash 引用、投递回执与审计事件。
- 绝不持久化任何平台的认证凭据、token、cookie、私人对话历史全文。
- 连接关闭、归档或跨 revision 变更时，内存及磁盘临时暂存区中已废弃的旧版本未决缓存一并清空。

**可执行并发规则（P1 必须按此实现）**：发送前 gate 检查、持久化 sending、发起 submit 必须在同一个短调度动作里连续完成；不能在 gate 已过、sending 已落盘后把提交交给一个可能晚于 pause 的异步回调。若发送前需要异步准备，准备结束后必须重新过门并重新确认 sending；只有 receipt 或结果的等待可以在短命令队列外运行。这样 pause 与 submit 才有明确的线性顺序：pause 先到则不再发起新的 submit，submit 先发起则按在途处理且不可假装撤回。

## 5. 状态机与两类审批

### 两类审批的定义与边界

1. **平台内执行级审批（Platform Execution Approval）**：
   - 发生在平台引擎内部。例如 Codex 的 `item/commandExecution/requestApproval` 或 DSH 的 `pending_approvals_count`。
   - 涉及本地命令执行、文件写操作或高风险工具调用。
   - Platform adapter 负责检测并向上报告此类状态；SessionRelay UI 进行状态展示或经由专用接口操作，不可绕过，不假冒平台授权。
2. **SessionRelay 业务级协调审批（SessionRelay Orchestration Approval）**：
   - 发生在 SessionRelay 状态机内部，用于控制跨会话流转与阶段跃迁。
   - 触发场景：
     - **阶段通过（`stage_passed`）**：模型提议完成阶段，必须经过人类最终批准才可关闭当前阶段、进入下一阶段。
     - **人工干预模式（`human_intervention`）**：每一轮结果生成后进入 `awaiting_approval`，由用户核准或人工编辑后才允许下发。
     - **管理方主导模式（`manager_driven`）**：由管理方审查后明确派发。
     - **人工编辑确认（`manual_override`）**：用户修改交接内容后的保存与放行。
   - **核心约束**：业务级协调审批必须由用户在 SessionRelay 界面显式点击或通过管理命令触发，**任何模型生成的回复文本均不具备解除协调审批的效力**。

### 连接状态机

~~~
paused ──start（有合法任务/待交接结果）──► running
running ──pause──► draining ──本轮收尾/无在途──► paused
running ──需要业务协调审批（stage_passed/人工干预/管理方确认）──► awaiting_approval ──用户批准──► running
任意 ──格式错误/平台异常/外部人工插话/超出往返上限──► needs_attention
needs_attention ──用户处理并确认──► paused
~~~

不变量：

- running 才允许新的自动投递。
- draining 不允许新投递，但允许已接收本轮完成；不能撤回平台已接收的任务。
- paused 不自动投递；恢复只消费未转发结果。
- start/resume 不改变绑定 revision；只有绑定/路由契约变化才递增。
- 模型输出不直接改变连接状态、角色或权限；结构化 decision 经校验只推进同阶段循环，stage_passed 仍需用户批准。
- 旧 revision 的结果不能进入新绑定的待转发队列。

### 任务状态机（Task States）

~~~
created ──开始执行──► running ──投递已发，等待结果──► awaiting_result ──收到结果并校验通过──► result_ready ──转发成功──► forwarded ──阶段结束──► closed
   │                     │                              │
   │                     │                              └──(校验失败/ID不匹配/超限)──► needs_attention
   │                     └──(平台调用不可逆失败)──► failed
   └──(取消)──► closed
~~~

- `created`：任务已初始化，参数已固化。
- `running`：正在准备投递或正在进行发送前过门。
- `awaiting_result`：投递已被平台接收（accepted），正在等待平台执行与生成结果。
- `result_ready`：已收到平台结构化结果并通过本地 schema 校验。
- `forwarded`：结果已成功组装并被对侧接收（accepted）。
- `closed`：任务正常完成或随阶段完结而关闭。
- `needs_attention`：发生格式不符、ID 冲突、外部插话或往返超限，挂起等待人工干预。
- `failed`：发生不可恢复平台异常或门校验失败终止。

### 投递状态机（Delivery States）

~~~
planned → sending → accepted → result_received → forwarded
   │         │           └──► failed
   │         └──► unknown（人工核对）
   └──► canceled（门/绑定校验未过/会话占用，未发送）
~~~

forward 是新的 delivery，不是临时状态；不存在 forwarding 状态。

forward 是新的 delivery，不是临时状态；不存在 forwarding 状态。

### 任务

~~~
created → running → awaiting_result → result_ready → forwarded → closed
                                  └──► needs_attention（格式/ID/上限）
~~~

## 6. 关键时序

### 开始

1. 用户选择 A/B 会话与角色，或使用已有连接。
2. 校验：非自连接、会话未属于其他运行连接、存在待交接结果或明确阶段任务。
3. 首次绑定创建 revision=1；已有绑定沿用当前 revision。只有平台/实例/会话/角色/路由契约变化才递增。
4. 固化 task/stage 与 bound_revision。
5. 进入 running，按方向投递第一条阶段任务。
6. 创建 delivery（planned），发送前过门，再调用平台。

### 暂停

1. pause 是短命令，直接进入短命令队列；它不能排在 waitForResult 后面。
2. 原子关闭“允许新投递”的门，状态置 draining。
3. 已进入 sending 的投递可能已被平台接收，不能撤回；按 receipt/核对流程处理。
4. 已接收的本轮允许完成；结果落 result_received，但不转发。
5. 无在途后置 paused；界面显示“已暂停，有 N 条待转发”。若存在 sending/unknown，则显示待核对，不假装已停止。

### 恢复

1. 核对平台状态与本地 accepted/sending/unknown 投递。
2. 只处理 result_received 且未转发的条目；旧 revision 不处理。
3. 用户确认后置 running，按序转发；转发本身创建新的 forward delivery。
4. 不重放 forwarded；不复活旧 revision 的结果；开始/恢复不改变绑定 revision。

### 退出与重开

1. 关闭窗口只收托盘，不改变连接状态。
2. 退出：停止新投递，等待当前写事务完成；不杀平台任务。
3. 重开：先加载 SQLite，核对平台；默认 paused，展示待核对/待转发，用户确认后才恢复。

## 7. 人工输入与外部插话

- 应用内“补充要求”是显式人工输入：manual=true，单独事件类型，不能伪装成交接结果。
- 外部人工插话：平台若提供 user message + clientId/read state 等信号，adapter 尽量识别；识别不到时 capabilities 必须标 unsupported/partial，UI 明示“本平台无法自动检测外部插话”。
- 检测到外部插话时：立即暂停协调（进入 needs_attention），不自动把人的话当模型结果转发。
- 人工输入不能绕过暂停门；补充要求也要走同一 delivery 路径与发送前校验。

## 8. 结果解析与控制边界

平台回复先落到原始引用，再由 handoff 解析：

- 只接受 JSON 或明确约定的结构化块。
- 必填：taskId、stageId、connectionRevision、decision、summary、artifacts。
- decision 白名单之外的文本 → needs_attention。
- 普通聊天回复、缺少字段、ID 不匹配 → 不投递。
- 同一阶段返修计数达到上限 → needs_attention，等用户。

控制边界：

- 模型文本不直接改变暂停状态、角色授权、权限或阶段。
- request_changes/submit_for_review 经程序校验可推进同阶段循环；这不是模型自行“跃迁”。
- stage_passed 只进入 awaiting_approval，必须用户批准才进入下一阶段。
- needs_user 只进入待人工处理，不能自动开始下一阶段。

## 9. UI 结构（P3）

- 左侧连接列表；右侧连接详情。
- 顶部状态条：paused/running/draining/awaiting_approval/needs_attention + 暂停/恢复按钮。
- 中部阶段时间线：task 创建、投递、receipt、结果、审批、失败。
- 结果面板：decision、summary、artifacts、打开文件/对话。
- 待转发面板：逐条显示来源、目标、payload 摘要、转发按钮。
- 补充要求输入：标记为人工输入，发送前确认。
- 设置：平台实例、会话选择、角色、往返上限、托盘行为。
- 能力提示：listSessions=manual 只用于临时诊断（手填 ID），正式仍需“选择已有会话”的入口；找不到就显示阻塞。detectExternalHumanInput=unsupported 的平台明确显示限制。

## 10. 非目标

- 不做云端中转、不做多机同步。
- 不做平台内部消息伪造、不做认证绕过。
- 不做自动批准、不自动扩大权限。
- 不做“保证恰好一次”的承诺。
- 不在首版支持一个会话属于两条运行连接。

## 11. 参考与设计取舍

外部项目只作设计参考，不是依赖；部分固定 commit 源码已读，未在本机安装或运行。详见 [references.md](references.md)。

- Agent Deck：当前固定 commit 的 `committed_inbox` 已转为持久 per-parent outbox 的 pull model，父会话在 turn 边界 drain；借鉴事件身份、来源归属、持久收件箱与去重，不把其 exactly-once 宣传或 README 旧 nudge 当作“空闲桌面立即唤醒”。
- formiat/multi-agent-orchestration：借鉴明确绑定、持久交接、实现→审查→返修循环；不把单文件 inbox/outbox 直接套到并发连接。它是技能/工作流参考，不是编译源码。
- agent-orchestrator：借鉴 newline JSON 帧、pending map 按 id 关联、写锁不跨等待、通知/审批分流与会话单写者；它自管 app-server，不证明可无冲突附着运行中的桌面会话，不引入 worker/worktree/PR 流水线。
- LangGraph：借鉴 checkpoint 分区与人工批准门；checkpoint 不保证外部发送不重复，业务仍需幂等/receipt 核对，interrupt 节点重执行要谨慎；本项目暂不引入该框架。
- OpenAI Codex 上游：固定 commit 的 queue_cmd、message_processor、transport 只解释 CLI 入口与协议设计；上游版本不等于本机 alpha，查到源码不等于接入完成。
