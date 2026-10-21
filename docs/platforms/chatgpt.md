# ChatGPT 普通 Chat 接入评估（候选，默认禁用）

本文只依据用户/GPT 已查的官方资料，评估**独立 SessionRelay 能否选择任意已有 ChatGPT 普通 Chat 并发起自动接力**。没有安装、没有发送探测、没有创建定时任务，也没有运行浏览器自动化。结论是候选评估，不是平台适配完成。

## 1. 结论

**当前官方资料不足以确认完整链路：选择任意已有普通 Chat → 外部发送 → 后台唤醒 → 完成结果回读与可靠关联。**

因此产品处理为：

- 保留 ChatGPT 普通 Chat 作为**候选平台类型**；
- **默认不启用自动连接**，不进入 P1；
- P1 继续优先 Codex↔DSH；
- 官方文档里的工作区/插件/API 能力分别限定，不互相冒充；
- 以后若测试，先单独提交方案并获得用户明确授权。

## 2. 官方证据与边界

| 官方资料 | 能说明什么 | 不能说明什么 |
|---|---|---|
| [ChatGPT Work and Codex](https://help.openai.com/en/articles/20001275-chatgpt-work-and-codex) | Chat、Work、Codex 是分开的产品面；Codex 历史与 ChatGPT 历史分开 | 工具内部能 list/read/send chat，不等于外部程序有同等 API；不能把 Codex thread 当普通 Chat |
| [Conversation state](https://developers.openai.com/api/docs/guides/conversation-state) | Conversations API 与 Responses 可保存 **API 创建**的对象 | 不能接管 ChatGPT 侧已有聊天；API 的 conversation_id 不能替代 ChatGPT 会话 ID |
| [Plugin reference](https://developers.openai.com/plugins/reference)、[ChatGPT UI](https://developers.openai.com/plugins/build/chatgpt-ui) | 组件内可用 **window.openai.sendFollowUpMessage({prompt, scrollToBottom})**，MCP Apps 可用 **ui/message**，请求 ChatGPT 追加组件消息 | 组件生命周期、后台是否可用、能否选择任意旧对话、最终回复能否回读、关联是否可靠均未验证；这不是外部通用的 Chat 发送 HTTP 接口 |
| [Scheduled tasks](https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt) | Work 支持特定 Gmail/Slack/GitHub 事件触发 | 没有查到通用本地回调唤醒任意普通 Chat 的公开接口；但也不能扩大成“完全没有任何事件支持”；定时任务不能替代用户事件驱动要求 |
| [Business data privacy](https://help.openai.com/en/articles/9261474-chatgpt-business-data-privacy-security-and-compliance) | Enterprise/Edu 有合规/审计读取等能力 | 审计读取不是个人账号主动模型推理或普通聊天唤醒的证明 |

以上均为官方公开资料阅读，未在本机验证。链接与页面内容可能随产品更新变化，未来测试前必须重新核对。

## 3. 能力判定（候选）

| 能力 | 当前判定 | 依据/限制 |
|---|---|---|
| 列出现有普通 Chat | 未发现外部入口 | 工具内部历史可见不等于公开 API；Conversations API 只管 API 创建对象 |
| 向用户已选普通 Chat 发送 | 无通用外部接口 | 插件组件桥只覆盖组件会话内的追加消息，候选且未验证 |
| 后台唤醒 | 未验证 | 没有查到通用本地回调；组件消息是否能在后台唤醒未验证 |
| 完成识别 | 未验证 | 无公开完成事件；不能把“组件请求已提交”当回复完成 |
| 投递↔结果关联 | 未验证 | 没有可用的外部 request/turn ID 映射证据 |
| 外部人工输入 | 未验证 | 普通 Chat 没有已证实的来源标记接口 |
| 审批 | 未验证 | 没有外部审批入口证据 |
| 桌面对话 vs API | 明确不等同 | API conversation 与 ChatGPT 已有聊天是不同对象面 |
| 断线恢复 | 未验证 | 无可回读的外部 turn/结果状态 |
| 隐私边界 | 不能整段复制 | 合规日志属于审计能力，不是推理/唤醒接口；正文仍需用户授权 |

能力矩阵中不得把 ChatGPT 标成 native、已支持或可自动连接；最多标 candidate/unsupported，并在 UI 明示。

## 4. 候选路径与所需证明

### 4.1 插件组件桥（唯一相对接近的候选）

官方组件 API 能在组件内请求 ChatGPT 追加消息，但要成为 SessionRelay 的普通 Chat 通道，至少还要证明：

1. 组件是否只能由用户在某个已打开对话中触发；能否后台运行；
2. 能否指定任意一个已有普通 Chat，而不是只作用于当前组件所在对话；
3. 追加的是用户可见消息、组件消息还是新一轮模型调用；是否会真正唤醒；
4. 回复是否可被组件或外部程序读取，是否有完成事件；
5. 发送与回复之间是否有稳定 ID 可关联；
6. 生命周期、权限、失败、取消、重连与速率限制。

在这些都未验证前，只能写成“组件消息桥候选”，不能写成“ChatGPT API 已打通”。

### 4.2 浏览器 UI 自动化（未验证备选）

如果未来用户明确授权测试，浏览器 UI 自动化只能作为未验证备选：它依赖页面结构，易受产品改版影响；本项目不宣称“只能靠截图”，也不使用私有凭据导出或绕过认证。当前不做，也不把它写进能力矩阵的 native 列。

### 4.3 定时任务/事件触发（不能替代）

Scheduled tasks 的 Gmail/Slack/GitHub 触发是 Work 的特定事件能力，不是通用本地回调，也不能保证唤醒任意普通 Chat。不能用“有定时任务”替代本产品的“外部结果到达才继续”的要求，更不能做模型定时轮询。

## 5. 产品决策

- ChatGPT 普通 Chat 保留为候选类型；capabilities 至少把自动会话枚举、自动投递、后台唤醒、完成关联标为 unsupported/unverified。
- 默认关闭自动连接；用户界面若展示，只能作为“未支持/待验证”，不提供“开始接力”按钮。
- P1 不包含 ChatGPT；不因这份文档扩大 P1 范围。
- 任何未来测试都需要单独方案、明确授权和最小只读/单次验证步骤；先证明会话选择与完成关联，再谈后台自动化。
- 不把 Conversations API、插件组件、Scheduled tasks 或 Enterprise 合规日志中的任一能力拼成“完整链路”。

## 6. P0 状态

- 已完成：官方资料阅读与边界整理（本文）。
- 未完成：本机入口、向已有普通 Chat 投递、后台唤醒、完成回读、关联、人工输入/审批、重连。
- 结论：**候选，默认禁用；不阻塞 Codex↔DSH 的 P1，也不得被描述为已支持。**
