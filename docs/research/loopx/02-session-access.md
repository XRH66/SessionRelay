# 02 · 既有会话接入：源码证明了什么

本页区分四个层次：官方协议、上游实现、本机可用入口、在用户选定原会话上的实测。当前只有前两层的研究补充；本轮没有产生任何真实发送或完成链证据。来源见 [证据清单](07-evidence-and-corrections.md)。

## 1. Codex：有可复用客户端，不等于解决桌面共享控制

### 已读实现

`loopx/chat_agent.py` 的 `CodexChatAgentSession.start` 自己启动 `codex app-server --listen stdio://`，完成 initialize/initialized，然后按 resume_thread_id 是否存在选择 thread/resume 或 thread/start。请求等待器按 JSON-RPC id 匹配；通知单独排队。发送 turn/start 后提取原生 turn id，在事件循环中排除其他 thread/turn，区分 final_answer 和 commentary，并处理 failed/interrupted。[LX05]

这比“只查到一个方法名”更有价值：它给出了可读的收发结构、短写入边界、流式消息筛选和错误处理路径。测试文件还包含合成 app-server、静默事件读者与控制回复并行、慢写入和 Unicode 载荷等用例；本轮只阅读选定测试，未执行。[LX10]

### 不能由此推导

1. 子进程能 resume thread，不证明可无冲突附着正在由桌面 App 持有的同一 thread。
2. LoopX 自己发出的 turn/start 能得到 turnId，不证明本项目已知 queue 路径的 messageId 与结果可可靠对应。
3. 进程内 RPC id 只区分一次连接上的请求/响应，不是崩溃后可查询的业务 delivery ID。
4. 已知原生 turnId 的事件过滤，不证明丢失 receipt 后能够定位是哪次任务被接受。
5. LoopX 自管运行时的终止/清理行为不能直接套到用户自己打开的客户端；SessionRelay 的暂停不是杀死客户端。

本轮读到的 turn/start 参数并未展示本项目要求的 clientUserMessageId 关联方案，因此不能把这段代码当作该字段已经验证的证据。[LX05；SR03]

### 提取时的改造边界

只借鉴消息 framing、waiter、通知分流、精确 turn 过滤和 final 输出选择。不要整段导入账号认证、隔离运行目录、runtime ownership 和默认 approvalPolicy。所读启动路径使用 approvalPolicy=never，但该值不应被解释为适合所有宿主的安全默认值，必须保留本项目和宿主的授权要求。[LX05]

适配器应更严格地要求关联所需的非空身份、明确终态白名单和原生 readback。所读循环对 failed/interrupted 有显式分支；移植时应额外测试缺失/未知 status、缺失 thread/turn id，不能把任何名为 turn/completed 的通知都当作成功。官方协议也区分 completed、interrupted 与 failed。[LX05、WEB01]

**结论：Codex 完成链仍是 P1 前置；LoopX 提供实现参考，未替本机解阻。**

## 2. DeepSeek Harness：原生插件是最值得追踪的新线索

### 原生插件与既有 CLI 是不同接入面

`packages/dsh-loopx-plugin/src/driver.ts` 运行在 DSH 宿主内，可使用 ctx.agents.list()/get()，监听 agent/status、inbox inserted/claimed/discarded、agent/pre-step、session/event 等事件，并检查 Agent、Session 对象和 id/header 的一致性。它为自己的续接消息设置独立 source 标记及 turnInstanceId；有竞争输入时取消待调度工作。[LX08]

这证明**宿主原生层存在活跃 Agent 枚举与来源事件线索**，纠正“现有 CLI 没 list，因此平台一定没有枚举能力”的推断。但它仍不等于：

- 已导出给外部软件的会话列表 API；
- 包含全部休眠/历史会话的目录；
- 用户当前安装的 DSH 版本提供同样契约；
- 能将 turn/end 唯一映射到现有 CLI 的 request_id。

现有 SessionRelay CLI 的 start/send/status/result 与 request_id 证据应继续复用。先核查现有宿主/MCP 是否已有所需入口；不足时，再提出一个窄插件桥，不能直接安装整个 LoopX 插件替代已工作的链路。[SR04、LX08]

### 暂停与人工输入方面的价值

driver 的 reservation 区分 queued、claimed、admitted、retired。取消未来工作只移除自己尚未领取的消息；已领取/已准入任务可标记 preserveAcrossPause。输入来源与精确 reservation 匹配，避免只凭 agent idle 或“最后一条文本”猜结果。[LX08]

这些机制适合转化成本项目测试，但原生 queued 能删除，不代表外部 CLI 已交付任务也能撤回。每条适配器只能按自己的实证 capability 声明取消能力。

### 部署和权限边界

插件 package.json 有特定 DSH peer ranges、Node 要求与 beta 版本；不能按包名一致就认定兼容。其完整安装/初始化会引入 LoopX 环境及集成配置，本轮未执行。原生事件桥如果确需安装，应单独获得授权并说明改动范围。[LX02、LX09]

`native-operation.ts` 展示了有界本地 socket 帧、原生 call identity 与未知效果不重试的设计，但它是 operation 授权工具，不是通用 send_message 接口；不能换个名称就拿来桥接聊天。[LX09]

**结论：DSH 原生插件可明显缩小探索范围；正式历史会话发现与外部完成关联仍需验证。**

## 3. Antigravity：集成宣传不能替代原桌面对话控制

本轮查到的 LoopX 宿主资料是运行时/Goal 连接方式的参考，不能自动视为任意已有 Antigravity 桌面对话的外部读写接口。Google 的 /resume 文档明确把 App 导入描述为克隆历史到 CLI；这与持续操作原桌面对话不同。[WEB02]

因此，本轮不把 Antigravity 从阻塞改成已支持。后续若评估 CLI、skill、schedule 或其他接口，必须分别核对身份是否不变、消息是否进入原对话、完成关联和权限。不能用周期模型唤醒替代 SessionRelay 的结果到达触发，也不因研究资料存在就安装 CLI 或启动服务。

## 4. 平台能力结论表

| 能力 | Codex | DSH | Antigravity |
|---|---|---|---|
| 上游可读实现 | 自管 app-server 的启动/resume、turn 与消息处理 | 宿主内精确会话 driver、来源事件、活跃 Agent 枚举 | 本轮不足以确认符合原桌面对话控制需求的实现 |
| 原会话不被替换 | 实现有 resume 路径；桌面并发附着未证实 | 原生 driver 绑定当前 Session；外部桥未证实 | App→CLI 导入不能当原会话控制 |
| 完成关联 | 自己创建的 turnId 有关联处理；queue 映射未解决 | 沿用 P0 request_id；与原生事件映射仍待验 | 未解决 |
| 同平台两条旧对话 | 未实测 | 未实测；活跃 Agent 发现是候选 | 未实测 |
| 是否可修改正式 capability 为已支持 | 否 | 仅保留 P0 已验证能力，不新增实测标签 | 否 |

## 5. 最小验证顺序建议

先把 Codex 与 DSH 的候选路线各自缩成一次有界验证，然后才做 A→B→A。不要求两条调查串行阻塞彼此，不扩大为所有厂商全通。具体输入、授权、终点和负例见 [验证计划](06-validation-and-handoff.md)。
