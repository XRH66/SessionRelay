# 阶段计划与验收

阶段按“先证明接口与状态机，再做 UI 与发行”的顺序推进。P0 结束必须由用户验收；未批准不进入 P1。

## 总览

| 阶段 | 目标 | 当前状态 |
|---|---|---|
| P0 | 三平台接口验证与方案落盘 | 本轮完成调查与文档，待用户验收 |
| P1 | 最小 Codex↔DSH 接力 | 未开始 |
| P2 | 暂停/恢复/崩溃核对 | 未开始 |
| P3 | 桌面 UI（Electron+React+托盘） | 未开始 |
| P4 | 同平台不同会话 | 未开始 |
| P5 | Antigravity 支持 | 阻塞 |
| P6 | 免安装发行包 | 未开始 |

## P0：接口验证（本轮）

### 范围

- 只读调查 Codex / DSH / Antigravity 的本机入口、版本、会话枚举、投递、完成识别、receipt 与结果关联、外部人工输入、审批能力。
- 建立文件库，完整保存产品方案、架构、平台矩阵、阶段计划与证据。
- 不安装依赖，不写业务代码，不向 Codex 投递，不启动 daemon。

### 退出标准

1. 文档覆盖已确认产品方案的全部要求。
2. 三平台矩阵明确“能否完成关联”，区分官方公开/本机私有/仅推断。
3. 记录可接续会话标识与证据位置。
4. 明确列出已验证、部分验证、未验证、阻塞。
5. 临时大文件清理，保留可重复只读探针。
6. 不把“检测到 exe/文档”当成“接口可用”。

### 本轮结果

- 文档与证据已落盘。
- Codex：queue 有用户/GPT 实证；app-server 协议具备枚举/完成/审批对象；实际接入未验证。
- DSH：CLI start/send/status/result 可用；request_id 原生关联；真实 request 已以 ended/completed 收尾。CLI 无 listSessions（只证明该入口缺失，其他宿主枚举接口未验证）；审批只有计数。
- Antigravity：官方 agy CLI 文档存在，本机当前 PATH 与已检查目录未发现 agy；桌面 App 无已验证控制入口；--conversation 只是寻址；P5 阻塞。
- ChatGPT 普通 Chat（候选附录）：官方资料不足以确认选择任意已有聊天、外部投递、后台唤醒、完成回读与关联的完整链路；保留候选类型、默认禁用，不进入 P1。
- 详见 [platform-contracts.md](platform-contracts.md)、[platforms/chatgpt.md](platforms/chatgpt.md) 与 [P0 evidence summary](p0-evidence-summary.md)。

### 遗留到 P1 的前置

- Codex 独立 app-server 的连接与线程恢复未实测；queue messageId 与结果/人工输入的关联未验证，结果读取与关联仍是 P1 前置缺口。
- DSH CLI 无 listSessions；其他宿主枚举接口未验证。P1 可临时手填 session id 作诊断，不取代正式会话选择需求。
- Antigravity 不进入 P1。
- ChatGPT 普通 Chat 仅候选、默认禁用，不进入 P1。

## P1：最小 Codex↔DSH 接力

### 目标

用真实平台完成一次可验证的双向往返接力：在用户已有桌面对话中，一侧收到明确阶段任务并产出结构化交接，另一侧接收、审查并回送结果；验证三种协作模式的调度与两类审批的分流，全过程具备 receipt、完成事件、防重防护与关联证据。

### 范围

- 实现 coordinator/handoff/delivery/store 的最小单进程骨架（先做 CLI/测试套件，不做 Electron UI）。
- 落地每条连接独立配置的**三种协作模式**：自动接力（auto_relay）、人工干预（human_intervention）、管理方主导（manager_driven）。
- 实现 DSH adapter（CLI start/send/status/result，request_id 关联）与 Codex adapter。
- 落地未决投递期间的会话占用互斥锁定规则。
- 落地同一来源结果的重复转发数据库级防重约束。
- 严格落实两类审批分流：区分平台内执行级审批与 SessionRelay 业务级协调审批，牢守 `stage_passed` 必须用户批准的红线。
- 临时会话入口：P1 允许临时手填已选桌面会话 ID 作诊断，但不采用新建会话或克隆历史替代原有桌面对话。

### 必须先完成：Codex 完成链验证任务（有明确终点）

在进行 Codex↔DSH 双向往返前，必须先行独立跑通有严格判定边界与终点的 Codex 完成链验证：

1. **输入与前提**：
   - 目标会话：用户指定的已有 Codex 桌面 thread UUID（如 `<CODEX_THREAD_UUID>`）。
   - 严禁行为：严禁调用 `thread/start`、`thread/fork` 或克隆历史生成新会话代替原对话；严禁启动未经授权的后台服务。
2. **验证步骤**：
   - **步骤 1（附着/读取）**：通过 app-server 候选（stdio/本地通道）对已选 thread UUID 尝试 resume/read，核实能否在不干扰桌面 App 正常渲染/持锁的前提下附着。
   - **步骤 2（带标投递）**：构造包含唯一标识 `clientUserMessageId`（如 `RELAY-VERIFY-<TIMESTAMP>`）的有界轻量测试任务（要求返回包含 decision/summary 的极简 JSON 块），发起 `turn/start`（或受控 queue 探针）。
   - **步骤 3（回执落盘）**：验证是否成功获得平台投递回执（turnId 或 messageId），本地 outbox 事务推进为 `accepted`。
   - **步骤 4（完成事件与关联核验）**：等待 `turn/completed` 通知或超时前通过 `thread/turns/list` 轮询回读，检查返回的 Turn 项中是否包含匹配的 `clientUserMessageId`，且 `Turn.status == completed`。
3. **终点判定与失败边界**：
   - **通过终点（PASS）**：成功附着原会话 + 捕获回执 + 捕获 completed 状态 + 结果中成功关联 `clientUserMessageId` + 桌面端无并发崩溃或异常。
   - **失败停机终点（FAIL）**：若发生套接字失败（如 10050）、线程锁争用异常、静默替换生成了新 threadId、或未能将结果与投递 ID 建立关联，立即判定验证未通过，进入 `needs_attention` 并停止，记录明确的失败归因。**绝不通过降级为“新建会话”或“克隆会话”伪称验证通过**。

### 通过后最小往返接力的验收步骤

Codex 完成链验证通过后，按下列 5 个具体步骤完成最小双向往返接力验收：

1. **步骤 1：连接建立与协作模式设定**
   - 绑定已选 Codex 桌面会话（角色 A：实现方）与 DSH 已有诊断会话（如 `<DSH_SESSION_ID>`，角色 B：审查方）。
   - 初始化 connectionRevision=1，往返上限设为 2。
   - 分别验证配置为 `auto_relay`、`human_intervention` 与 `manager_driven` 时的调度分流逻辑。
2. **步骤 2：第一程投递与会话占用（Codex 执行）**
   - 下发初始阶段任务（方向 a_to_b），outbox 经事务 G 校验通过，Codex 会话置为**占用（occupied）**。
   - 触发平台提交，获得 receipt 后 delivery 进入 `accepted`，task 进入 `awaiting_result`。
   - 验证会话占用保护：在此期间尝试向该 Codex 会话下发新投递，验证系统拒绝并返回 `session_occupied`。
   - Codex 生成结构化交接信封（`decision=submit_for_review`，summary，artifacts 引用），系统回读校验通过，task 进入 `result_ready`，Codex 会话占用解除。
3. **步骤 3：跨会话接力转发与防重验证（投递至 DSH）**
   - **在 `auto_relay` 模式**：系统自动生成 forward delivery，经 `idx_deliveries_source_forward` 校验无重复，通过 DSH CLI `send` 投递至 DSH 会话，DSH 会话置为占用；获得 request_id 回执后源 delivery 标为 `forwarded`。
   - **验证防重机制**：人为触发对同一源结果的重复转发，验证系统被数据库唯一约束拦截，未向 DSH 发送二次请求。
   - **在 `human_intervention` 模式**：验证结果就绪后自动停在 `awaiting_approval`，支持用户就地编辑 summary（标记 `manual_override=true`），经用户显式批准后方发起 DSH 投递。
   - **在 `manager_driven` 模式**：验证结果就绪后停在就绪态，由管理方显式指定指令与目标后触发投递。
4. **步骤 4：第二程回返（DSH 审查并生成决议）**
   - DSH 针对第一程交付物完成审查，产出结构化审查结果，decision 为 `request_changes`（退回修改）或 `stage_passed`（建议通过）。
   - 系统通过 `dsh-cli result` 配合原生 `request_id` 准确回读结果并校验 schema，DSH 会话占用解除。
5. **步骤 5：阶段通过审批与红线验收**
   - 若 DSH 返回 `request_changes`，按既定协作模式组装返修任务回传 Codex，往返计数自增，验证未超限前继续循环。
   - 若 DSH 返回 `decision=stage_passed`，系统状态机**必须无条件进入 `awaiting_approval`**；
   - 验收红线：验证模型文本中的任何“阶段已完成，请进入下一步”等文字均无法推动系统，必须由用户人工在 SessionRelay 界面/CLI 显式点击批准，阶段状态才允许转为 `closed` 并进入下一阶段。
   - 验收数据落盘范围：检查 SQLite，确认仅保存交接信封、文件指针、receipts 与事件流，无任何 token、cookie 或私人聊天正文 dump。

### 验收标准汇总

- Codex 完成链验证通过（原会话、带标关联、完成捕获），或按失败边界停机。
- 一次真实的 Codex↔DSH 双向往返完成，双方均具备不可伪造的 platformTurnId / request_id 原生关联。
- 三种协作模式（自动接力、人工干预、管理方主导）的调度分流与人工编辑流程经实测生效。
- 未决投递期间的会话占用互斥生效，并发投递被正确拒绝。
- 同一来源结果的重复转发防护生效，禁止二次重复提交。
- 阶段通过（`stage_passed`）必须经用户批准，模型文本无法越权。
- 暂停与崩溃核对：在途本轮收尾暂存，重启核对 receipt，不盲目重发。

### 风险

- Codex app-server 与桌面 App 的线程所有权冲突。
- DSH 认证/3080 服务不可用。
- 平台结果格式不稳定，需要明确的解析失败路径。

## P2：暂停、恢复与崩溃核对

### 目标

把 P1 的暂停门扩展为完整生命周期：running → draining → paused，awaiting_approval，needs_attention；退出/重开先核对再保持暂停。

### 验收

- 暂停后立即不再发起新投递；在途本轮收尾，结果暂存。
- 恢复只处理未转发结果；不重放 forwarded。
- 退出不杀平台任务；重开显示待核对/待转发，默认 paused。
- 模拟崩溃：planned/accepted/result_received 各窗口恢复行为符合设计；unknown 不自动重发。
- 审批门：stage_passed 必须用户批准，模型文本不能解除暂停。

## P3：桌面 UI

### 目标

Electron + React 前台 + 托盘；有限 IPC；不开放 HTTP 端口。

### 验收

- 连接列表、平台/会话选择、角色配置、开始/暂停/恢复、阶段时间线、结果与交付文件、待转发、补充要求、打开双方对话。
- renderer 无法执行任意命令或读取凭据。
- 关闭收托盘；退出停止转发不杀平台任务；重开核对后保持暂停。
- 能力限制可见：缺少正式枚举入口的平台要有明确的临时手填/导入方案，不能把手填当最终方案；无法检测外部插话的平台明确提示。

## P4：同平台不同会话

### 目标

支持 Codex↔Codex、DSH↔DSH。

### 验收

- 两个不同会话可独立配对；拒绝自连接。
- 首版一个会话不能属于两条运行连接；有明确冲突提示。
- 旧回复按旧 connectionRevision 归档，不进入新连接。

## P5：Antigravity

### 目标

只在有稳定、可验证接口时支持；否则保持手动导入。

### 当前阻塞

- 本机只有桌面 App 2.24.0；当前 PATH 与已检查目录未发现 agy CLI，CLI 配置目录不存在。
- 官方 agy headless 文档存在，但未安装、未认证、未验证；--conversation 是寻址参数，不是枚举。
- 官方 resume 的 App 导入是克隆历史，不是控制原桌面会话；不能把 import 当成原会话控制。
- 私有 agentapi shim 未验证。

### 若后续解阻

- 先验证 agy 的安装版本、认证、conversation_id 续接、每轮 result、权限/审批行为。
- 明确“克隆会话”与“原桌面会话”的产品差异，不能混写。
- 没有稳定入口就继续标阻塞，不用 UI 自动化冒充接口。

## P6：免安装发行包

### 目标

Windows 免安装包；不写系统服务、不开机自启、不监听端口。

### 验收

- 解压即用；数据目录可迁移；升级不丢 SQLite。
- 退出清理正确；托盘行为符合预期。
- 不含凭据；日志脱敏。
- 安装包体积、签名、更新策略另定。

## 阶段门

- P0 → P1：用户验收本库，明确批准。
- P1 → P2：最小接力真实跑通，关联证据可复现。
- P2 → P3：暂停/恢复/崩溃核对通过。
- P3 → P4：UI 主路径可用，能力限制可见。
- P4 → P5：同平台冲突规则验证。
- P5 → P6：Antigravity 要么有稳定接口并通过验证，要么明确排除在首版之外。

## 不因阶段推进而放宽的规则

- 模型文本不能解除暂停/扩大授权。
- 阶段通过必须用户批准。
- 未知投递状态不盲重发。
- 不承诺 exactly-once。
- 不修改平台配置/认证/数据库来换取“能跑”。
