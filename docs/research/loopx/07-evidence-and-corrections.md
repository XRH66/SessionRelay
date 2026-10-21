# 07 · 固定证据、阅读范围与旧结论修正

研究日期：2026-10-22。LoopX 固定提交 `5bf58c1ca3d21919b15c662bc705c3e069bfb496`；SessionRelay 固定基线 `0049af9f47d41c27b8da486107d6d7f4567ba384`。源码存在、测试源文件存在、上游报告测试成功和本轮亲自实测是四种不同证据，不互相替代。

## SessionRelay 基线

| ID | 入口 | 本轮用途 |
|---|---|---|
| SR01 | [README](https://github.com/XRH66/SessionRelay/blob/0049af9f47d41c27b8da486107d6d7f4567ba384/README.md)、[ProjectContext](https://github.com/XRH66/SessionRelay/blob/0049af9f47d41c27b8da486107d6d7f4567ba384/ProjectContext.md) | 产品目标、三模式、原会话、不重造客户端、P0 状态；ProjectContext 按相关范围读取，非逐字全文审计 |
| SR02 | [architecture.md](https://github.com/XRH66/SessionRelay/blob/0049af9f47d41c27b8da486107d6d7f4567ba384/docs/architecture.md#L26-L200) | coordinator/handoff/delivery/store、占用、防重、现有契约和 SQL 草案 |
| SR03 | [phases.md](https://github.com/XRH66/SessionRelay/blob/0049af9f47d41c27b8da486107d6d7f4567ba384/docs/phases.md#L1-L180) | P0/P1、原会话完成链、双向往返与三模式要求；读取返回中的 P1 段落 |
| SR04 | [platform-contracts.md](https://github.com/XRH66/SessionRelay/blob/0049af9f47d41c27b8da486107d6d7f4567ba384/docs/platform-contracts.md#L1-L70)、[P0 摘要入口](https://github.com/XRH66/SessionRelay/blob/0049af9f47d41c27b8da486107d6d7f4567ba384/docs/p0-evidence-summary.md) | 本轮读取契约 1–70 的能力矩阵、统一接口及 Codex 既有证据，并与先前用户提供的契约交叉核对；摘要仅为接续入口，未声称本轮逐行审计 |
| SR05 | [CONTRIBUTING](https://github.com/XRH66/SessionRelay/blob/0049af9f47d41c27b8da486107d6d7f4567ba384/CONTRIBUTING.md) | 共享核心、证据分级、拒绝不确定状态和公开资料隐私规则 |
| SR06 | [旧 LoopX 分析](https://github.com/XRH66/SessionRelay/blob/0049af9f47d41c27b8da486107d6d7f4567ba384/docs/references/loopx-analysis.md)、[旧 references](https://github.com/XRH66/SessionRelay/blob/0049af9f47d41c27b8da486107d6d7f4567ba384/docs/references.md) | 本轮修正对象；保留历史版本可追踪 |

## LoopX 已读源码与文档

| ID | 固定源码/文档 | 已读范围、主要符号及证据等级 |
|---|---|---|
| LX01 | [pyproject.toml](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/pyproject.toml#L1-L180) | 打包元数据；2.0.0、Python/Node 要求、TS 包数据、扩展；配置证据，未安装 |
| LX02 | [DSH package.json](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/packages/dsh-loopx-plugin/package.json) | 全文；beta 版本、exports、Node、DSH peer ranges、Apache；不是兼容性实测 |
| LX03 | [architecture.md](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/docs/architecture.md#L1-L220) | 六层状态、Agent/Provider/Capability/Kernel、effect 边界；设计文档 |
| LX04 | [collaboration/README.md](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/loopx/control_plane/collaboration/README.md) | 全文；请求关系、parent lineage、receiver receipt、result 文件边界；源码配套文档，不等于本轮运行测试 |
| LX05 | [chat_agent.py](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/loopx/chat_agent.py) | 选读 1–240、600–1040、1170–1400、1670–1880；部分大段返回截断，关键启动与事件区间另分段读；CodexChatAgentSession.start/_request/_route_response、turn 过滤和终态输出；未审计整文件 |
| LX06 | [semantic_request.ts](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/loopx/control_plane/collaboration/semantic_request.ts) | 完整返回；normalizeCollaborationBrief/Request、workspaceRef、planCollaborationRequestPublication、sameRequest；源码阅读 |
| LX07 | [result_publication.ts](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/loopx/control_plane/collaboration/result_publication.ts) | 完整返回；planCollaborationResult、resultAttachments、collaborationResultDeliveryReady；源码阅读 |
| LX08 | [DSH driver.ts](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/packages/dsh-loopx-plugin/src/driver.ts) | 1–180、380–620、1000–文件尾；source/reservation、pause 保留、身份检查、ctx.agents.list 与事件注册；未审计全文件/全部发送分支 |
| LX09 | [native-operation.ts](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/packages/dsh-loopx-plugin/src/native-operation.ts)、[插件 README](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/packages/dsh-loopx-plugin/README.md#L1-L135) | TS 源码全文；有界 socket/原生身份/未知效果不重试。README 先读公开页面，再按固定 SHA 读取 1–135，核对自动初始化与安装边界；未执行 |
| LX10 | [test_chat_agent.py](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/tests/test_chat_agent.py#L1-L180) | 选读：静默事件读者与控制回执、slow write/flush、Unicode、fake process；未运行测试，不把合成 peer 当真实 Codex |
| LX11 | [Desktop README](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/apps/desktop/loopx-control-plane/README.md#L1-L110) | Tauri 实验外壳、React/HTTP 服务、Windows 预览限制；文档证据，未审计 Rust/React 全部实现 |
| LX12 | [cross-runtime 用例](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/docs/product/use-cases/cross-runtime/README.md) | 固定 SHA 完整返回：DSH 每阶段新执行会话、显式控制器、Git 产物转移；不是本轮实测或真实跨厂商既有会话验证 |
| LX13 | [roundtrip.py](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/loopx/capabilities/manager_context/roundtrip.py) | 1–220、540–760；原会话路由、attempt locator、回读验证、single-flight；未审计全部重试分支 |
| LX14 | [packages/README.md](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/packages/README.md)、[runtime catalog](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/docs/integrations/runtime-connector-catalog.md) | 包分发边界及宿主连接方式；catalog 读取返回范围，非每个 connector 的运行验证 |
| LX15 | [LICENSE](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/LICENSE)、[NOTICE](https://github.com/loopx-project/loopx/blob/5bf58c1ca3d21919b15c662bc705c3e069bfb496/NOTICE) | LICENSE 1–123、NOTICE 全文；Apache 第 4 节与截至 v0.4.7 的历史 MIT 说明；未做依赖树完整许可审计 |

以上链接固定到研究快照，避免 main 变动后无法追踪结论。代码范围明确记载；完整目录不等于已阅读所有文件。正文没有把文档宣传提升为代码或实测证明。

## 官方资料补充

| ID | 来源 | 本轮用途 |
|---|---|---|
| WEB01 | [OpenAI app-server](https://developers.openai.com/codex/app-server) | 官方协议入口；初始化、thread/turn、完成事件及状态，不证明用户本机已接通 |
| WEB02 | [Antigravity /resume](https://antigravity.google/docs/cli/commands/resume/) | App 导入克隆历史与 CLI resume 的区别，不证明原桌面对话被外部控制 |
| WEB03 | [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0) | 与仓库许可文本交叉核对分发要求 |

在线页面读取日为本研究日，可能随版本更新；优先用固定源码定位和本机版本测试决定实现。

## 对旧报告的具体修正

| 旧表达 | 本轮证据支持的修正 |
|---|---|
| Python 栈，所以不能搬源码 | 存在 TypeScript 控制面和独立 TS DSH 插件；能否抽离取决于依赖/身份边界，不由语言单独决定 |
| 偏单 Agent 的单线滚动 | 当前有对等请求/协调关系与多 Agent 协作层；但不等于满足双旧会话桥接 |
| LoopX 不写文件、不运行 shell | 责任分层不等于整个产品没有执行/IO；已有客户端启动、文件存储和宿主集成实现 |
| Apache-2.0 / MIT 可笼统并列 | 当前根与插件是 Apache-2.0；NOTICE 将 MIT 限定为历史截至 v0.4.7 的发行 |
| 真实信封是四个 proven/verified/open/next 分区 | 已读协作 brief 的真实字段为 purpose/context/constraints/inputs/acceptance/return_requirement；四分区只能是另拟建议 |
| 文件存在、非空、hash 可杜绝虚报成功 | 只能提供部分存在性/版本证据；业务正确性需要任务专属验证，hash 也需可信计算 |
| 应强制某个指定低价模型和 reasoning 参数 | 本轮没有查到这作为 LoopX 通用规则的依据；应使用用户批准的配置/预算与真实 capability，不硬编码品牌模型 |
| 必须绝对排除 Python 运行时 | 这是选型建议，不是用户不可改变的产品目标；保留小模块提取、可选子进程和整套底座三种成本比较 |

## 未覆盖内容与结论失效条件

未审计全仓、所有插件和传递依赖；未核验全部 CI/发行资产；未编译、运行 smoke 或真实模型；未连接用户 Windows 环境。因而不提供性能指标、整体安全认证或准确工时承诺。

上游接口/许可变化、本机版本变化、不同会话存储或权限配置、改变 no-listener/原会话约束，都会使部分结论需重验。研究结论的强度是“具备有依据的复用候选与验证方案”，不是“已经融合完成”。
