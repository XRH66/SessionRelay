# LoopX 源码评估与 SessionRelay 融合研究

研究日期：2026-10-22。状态：**静态研究完成；融合建议待采纳；未开始 P1 实现或真实会话测试。**

## 决策摘要

**值得融合，但当前最合适的是按模块融合，而不是直接把两个仓库拼成一个应用。** LoopX 已有可参考的多智能体交接、原生 DSH 会话驱动、Codex app-server 收发与 TypeScript 校验实现。它不是只有 Python 的单智能体记忆工具。另一方面，这些实现并没有替 SessionRelay 证明“安全绑定用户指定的两条现有桌面对话，并完成可靠双向交接”。这是本项目仍需单独验收的产品边界。[证据 LX01–LX08](07-evidence-and-corrections.md)

优先复用的价值在三处：减少平台事件处理的摸索，减少交接身份/版本/重放规则的重新设计，获得可以转写为本项目回归测试的具体反例。**不能把这些收益折算成已经打通接口、已经兼容 Windows，或未经测量的工期节省百分比。**

建议先实施路线 A：以 SessionRelay 作为连接、投递和审批的唯一状态所有者，提取 LoopX 的小范围 TypeScript 校验与协议处理设计。路线 B 保留为后续选项：将 LoopX 作为可选的长期 Goal/Todo 治理子进程。路线 C（以 LoopX 整体为底座重做产品外壳）不排除，但需先证明它比路线 A 更接近用户效果，而不是仅因为功能多就采用。详见 [复用与融合方案](04-reuse-and-fusion.md)。

## 阅读顺序

| 文档 | 回答的问题 |
|---|---|
| [01 产品定位与需求差距](01-product-fit.md) | LoopX 到底是什么；与既有会话桥接器有哪些实质差别 |
| [02 现有会话接入核查](02-session-access.md) | Codex、DSH、Antigravity 的源码能证明到哪一步 |
| [03 消息、身份与恢复](03-protocol-and-recovery.md) | 如何借鉴请求关联、版本链、暂停与未知投递处理 |
| [04 源码复用与融合路线](04-reuse-and-fusion.md) | 哪些模块可提取，哪些要改造，整套融合的代价是什么 |
| [05 许可证与发行边界](05-license-and-distribution.md) | Apache/MIT 的版本区别；Python、Node、桌面外壳与第三方声明 |
| [06 有界验证与执行交接](06-validation-and-handoff.md) | 下一次获批后具体验证什么、怎样判定通过、何时停止 |
| [07 证据清单与旧报告修正](07-evidence-and-corrections.md) | 固定提交、已读源码范围、没有验证的事项、旧结论更正 |

## 固定基线与证据边界

- SessionRelay：`0049af9f47d41c27b8da486107d6d7f4567ba384`。以本轮读取的 GitHub 正式规格为基线，而不是较早的聊天附件。
- LoopX：`5bf58c1ca3d21919b15c662bc705c3e069bfb496`，仓库 `loopx-project/loopx`。该快照的 `pyproject.toml` 标注版本 `2.0.0`；这不等于核验了所有发行资产或最新稳定版本。
- 已做：读取项目规则、现有规格、选定源码与测试片段，核对公开官方协议资料，形成设计建议。
- 未做：安装/构建 LoopX、执行其测试套件、运行真实模型、读取私人聊天、附着用户客户端、修改平台配置、实现适配器。

本文档集不改变既有 `ProjectContext.md`、`docs/architecture.md`、`docs/platform-contracts.md` 或 `docs/phases.md` 的正式状态；不因引用上游代码就把平台能力改成已支持。

## 与现有资料的关系

本轮基线已包含三种模式、会话占用、转发防重和双向接力验收，不把它们再次列成“尚未设计”。旧入口 [loopx-analysis.md](../../references/loopx-analysis.md) 保留为修订说明与导航。修订不抹掉历史：旧内容可在上述 SessionRelay 基线提交中查看。

最先需要解决的不是换框架，而是两个有终点的接口实验：**Codex 原桌面对话的完成链；DSH 宿主原生能力能否通过受支持的窄接口提供给外部程序。** 在这两条证据成立前，不承诺整合 LoopX 就能获得用户想要的效果。
