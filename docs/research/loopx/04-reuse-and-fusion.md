# 04 · 可复用模块、融合路线与成本判断

本页为研究建议，不是已批准的实现规格。所有模块定位使用 [固定源码清单](07-evidence-and-corrections.md)。

## 1. 推荐不是“只能借思想”，而是选择性复用真实实现

| 候选模块 | 对应 SessionRelay | 推荐方式 | 必须改造/核验的边界 | 优先级 |
|---|---|---|---|---|
| semantic_request.ts | handoff | 提取/改造 TypeScript 校验，迁移相应测试思想 | Goal/Agent 身份改为本项目上下文；保留原有 decision；大小限制按实际需求配置 | 高 |
| result_publication.ts | handoff + delivery | 提取不可变结果、稳定更新 ID、回传先后约束 | 依赖 request_control 等模块；不可直接导入整套 Goal 生命周期 | 高 |
| DSH driver.ts | DSH adapter / 可选宿主桥 | 改造事件订阅、来源标记、精确会话检查和暂停边界 | 原生插件接口不是外部 API；不能整体带入 heartbeat、Goal 调度和自动安装 | 高 |
| CodexChatAgentSession | Codex adapter | 协议逻辑移植到 TypeScript；复用测试场景 | Python 实现；独立进程 ownership、认证、sandbox 与桌面旧会话条件不同 | 高 |
| roundtrip.py | delivery + store | 借鉴确定性路由、attempt、readback/unknown 规则 | 不搬入另一套文件状态库和回传 pump；没有全路径 exactly-once 保证 | 中高 |
| native-operation.ts | 本地桥传输工具 | 有界帧、超时、原生身份与拒绝盲重试的参考 | 它不是聊天 API；socket/权限/Windows 支持另验 | 中 |
| Goal/Todo/quota/attention | 可选长期任务扩展 | 在有明确需求后评估子进程集成 | 单一调度所有权；软件 quota 不等于真实账号余额 | 后置 |
| React 看板、协作卡和桌面外壳 | UI | 优先借信息组织；组件只在单独审计依赖后移植 | Tauri/HTTP/自管 Chat 与 Electron IPC 不同；本轮未审计完整前端组件 | 后置 |
| 账户路由、Lark、域扩展、自动更新 | 非当前核心 | 默认不纳入 | 不为“顺便增强”引入新账号、外发通道和权限 | 不进入首轮 |

对应证据 LX04–LX14。这里的“高”表示对需求的相关性，不表示模块已通过本项目生产测试。

## 2. 三条融合路线公平比较

### A. 小模块提取 + SessionRelay 唯一协调核心（当前推荐）

保持 TypeScript/Electron/SQLite，直接复用适合抽离的 TS 代码及测试，移植 Python 的协议处理结构。每个引入文件保留来源、固定 SHA、许可证和修改说明。

优势：不增加第二个任务状态库；可以逐件验证；继续满足不替代客户端、不增加 HTTP 监听和已有三模式。代价：要做上下文类型映射、依赖剥离、错误码适配及后续上游变更跟踪；不能“复制两个文件就算完成”。

首个依赖切片应从 semantic_request.ts 开始。它依赖 runtime_decode、错误类型、digest 和 Goal identity；先画出这几个导入边界，决定沿用小工具还是适配成本项目类型。result_publication.ts 进一步依赖 request_control，不能为省一段校验拉入整个控制面。

### B. SessionRelay 前台/桥接核心 + 可选 LoopX 治理子进程

当用户确实需要长期任务图、证据登记、持续上下文或多项目看板时，可让 LoopX 负责这些域，SessionRelay 仍负责既有会话绑定、投递与用户审批。

采用有界 CLI/stdio JSON 请求响应作为候选边界；不能假定已有稳定的嵌入 SDK。先用不调用模型的操作证明版本探测、超时、结构化错误、只读状态和退出清理，再评估写入。不能把整个 HTTP 服务自动引入，也不能让两个调度器对同一任务各自推进。

建议的所有权划分：SessionRelay 独占 connection、platform session、delivery、receipt 和阶段批准；LoopX 如被采用，只能在明确的独立域拥有 Goal/Todo/evidence。跨域用不可变关联引用和验证结果，不互相直接修改数据库。用户批准仍归 SessionRelay，LoopX 建议不自动成为授权。

优势：可复用更大范围的治理功能。代价：Python+Node 打包、进程生命期、两个版本体系、离线安装、数据迁移、错误映射和恢复对账。当前 pyproject 的 Python dependencies 为空并不等于整个产品无运行依赖。[LX01]

只有隔离试验显示这条路线确实少写大量必要逻辑，并且单一调度权可证明，才建议纳入产品。

### C. 以 LoopX 整体为底座，改造为 SessionRelay

这条路线不是因语言不同而禁止。它已有多 Agent 协作和 UI，可以在需求扩大时成为合理选择。但本轮可见桌面形态是 Tauri + React + 本地 HTTP 服务，并包含自管 Chat/Goal 模型，与本项目的两条原会话、Electron IPC、无监听和免安装目标不同。[LX11]

整套采用的决策门：必须先证明原会话桥接路径可用；明确哪些旧约束用户愿意改变；测量部署与维护负担；完成 Windows 资格验证。若仅换外壳却还需重写核心既有会话适配，不能声称它比 A 更省工。

## 3. 防止出现“两个大脑各自发任务”

融合后只允许一个地方决定是否发起实际平台投递。推荐路径：

用户或已授权管理方提出动作 → coordinator 校验模式、角色、阶段与暂停 → handoff 校验上下文及产物身份 → store 事务建立 delivery → adapter 提交 → receipt/终态回写 → 受控转发。

LoopX 提取模块在该路径里是验证器或事件适配器，不是旁路调度器。可选 LoopX 子进程给出建议时同样要回到该路径。UI、CLI、IPC 共用真实核心操作，不增加测试专用捷径。

尤其不能把 LoopX 注册 Agent 的动作当成给平台创建会话，或把目标选择自动提升成执行授权。额度耗尽后替换绑定要先封住旧投递和旧 revision，再建立明确迁移包。

## 4. 怎样估算节省工作，而不是编一个百分比

本轮没有实现基准、移植试验或工时记录，不给出人日/百分比。可确认的是可省略的重复设计范围：

| 工作包 | 已有参照能减少什么 | 仍必须自己完成什么 |
|---|---|---|
| Codex 协议客户端 | framing、waiter、通知过滤、终态输出处理、部分回归场景探索 | 原桌面附着、queue 对应关系、持久恢复、本机版本验证 |
| DSH 接入 | 原生事件名、活跃 Agent 发现、来源标记、reservation 生命周期摸索 | 外部受支持接口、历史会话、CLI request_id 与事件映射 |
| handoff | 有界语义上下文、相对路径、身份冲突和重放规则 | 本项目 schema/角色/模式/阶段批准映射、Windows 文件边界 |
| delivery | 路由固定、attempt/readback 与结果版本链设计 | SQLite 事务、崩溃窗口、重复事件与未知状态验收 |
| 桌面产品 | 看板/状态分层的信息设计 | Electron IPC、安全配置、托盘、无监听发行和真实用户流程 |

移植试验再记录导入文件数量、传递依赖、修改行数、需要维护的上游接口、单元/集成负例覆盖和打包增量，作为路线 A/B/C 的实测比较依据。

## 5. 额外功能的取舍

优先吸收“产物版本与审查版本一致”“有原因的阻塞状态”“关联到原请求的结果回传”和“上下文约束保留”。quota 可先只做明确往返上限与用户预算，不在没有可靠来源时显示具体订阅剩余额度，不自动换账号或未经批准改模型。

长期规划、学习记忆、跨 Goal 委派和完整管理工作区都可保留在扩展候选，不应变成跑通第一条接力的前置。下一步具体实验见 [验证与交接](06-validation-and-handoff.md)。
