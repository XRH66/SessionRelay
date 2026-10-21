# LoopX 开源项目调研与架构借鉴分析

- 调研时间：2026-10-22
- 调研对象：`huangruiteng/loopx`（及相关组织仓库 `loopx-project/loopx`）
- 开源许可：Apache-2.0 / MIT
- 证据来源：公开仓库架构文档、设计规范与源码接口说明（阅读级证据，未在本机安装 Python 包）
- 核心指导原则：**以 SessionRelay 既有功能结构为主，择其善者而用之，不改变产品定位，不引入冗余依赖。**

---

## 1. LoopX 是什么与核心解决的痛点

LoopX 是一个面向长周期（Long-running）AI Agent 任务的**本地状态内核与控制面（Local-first State Kernel & Control Plane）**。它不替代底层 Agent 工具（如 Codex、Claude Code、Cursor），而是位于它们之上进行状态治理。

它针对的核心痛点是：
1. **上下文衰减（Context Decay）**：Agent 在持续执行多轮后，聊天上下文窗口变长，模型开始遗忘原始目标、重复犯错或忽略规则。
2. **假报成功（Self-reporting Hallucination）**：模型在没有真正运行测试或生成产物的情况下，直接在文本中声明“任务已完成，所有测试通过”。
3. **缺乏结构化交接（Context Rollover）**：会话中断或重开时，靠大段聊天记录恢复效率极低且丢失边界。

---

## 2. LoopX 核心架构与设计机制

### 2.1 状态与执行严格分离（State vs. Execution）
- **LoopX 不做执行**：LoopX 自身绝不直接调用 LLM，不直接写文件，也不擅自运行 shell；
- **底层 Runtime 负责执行**：实际的代码修改、测试运行由接入的 Codex CLI 或 Claude Code 执行；
- **LoopX 只管状态持久化**：以本地文件/目录（如 `.loopx/`）记录不可变的任务图（Task Graph）、证据日志（Evidence Logs）、门控（Gates）和配额状态。

### 2.2 “Proof-or-Stop” 证据门（Evidence Gate）
LoopX 强制执行“先拿证据，否则停机”的严格纪律：
- 模型如果声称“功能已完成/已修复”，不能仅凭输出文本中的“Done”或“Fixed”；
- 必须有机器可读的真实产物（如测试执行日志、编译成功输出、真实落盘的文件及 hash）；
- 只有通过证据门验证，任务才被允许标记为完成或推进到下一阶段。

### 2.3 标准化交接信封（Typed Handoffs）
在多轮滚动交接时，LoopX 将上一轮产出序列化为紧凑的交接信封：
- 已证明事实（Proven facts）；
- 交付物及验证凭证（Verifiable artifacts）；
- 待决决策（Open decisions）；
- 给下一轮的明确指令载荷（Next payload）。

### 2.4 配额与调度哨兵（Quota & Cost Guard）
提供类似 `loopx quota should-run` 的机制：
- 在触发下一轮执行前，先评估当前剩余配额、成本上限与健康状态；
- 若预算超标或状态异常，直接熔断拒绝自动调度，避免 Agent 循环死斗烧光 token。

---

## 3. 与 SessionRelay 的异同对比（坚守“以我为主”）

| 维度 | LoopX 做法 | SessionRelay 方案（以我为主） | 取舍结论 |
|---|---|---|---|
| **核心业务定位** | 单 Agent 多天长周期任务的滚动恢复内核 | **Windows 桌面双已有会话跨平台协作台**（Codex ↔ DSH） | **坚持 SessionRelay**：聚焦跨平台双会话协同，而非单 Agent 长期记忆滚动。 |
| **协作拓扑** | 单线递进的任务图 / 树状分解 | **A 侧与 B 侧双角色协同**（如实现方 ↔ 审查方，双向往返） | **坚持 SessionRelay**：保留连接（Connection）与 3 种独立协作模式（自动、干预、管理）。 |
| **技术栈与形态** | Python 3.11+ 命令行工具与后台脚本 | **TypeScript + Electron + React + SQLite WAL** 桌面本地应用 | **代码不直搬**：不引入 Python 环境依赖，在 TS 中吸收其算法与状态思想原生实现。 |
| **人机干预控制** | 依赖 CLI 提示与中断标志 | **桌面 UI 前台 + 系统托盘 + 白名单 IPC + 审批门控** | **坚持 SessionRelay**：通过 UI/托盘提供直观的人工放行、人工就地编辑与阶段审批。 |

---

## 4. 可以搬移与借鉴的现成设计（优化点）

虽然语言栈不同，但 LoopX 的多项核心设计模式完全可以直接吸收到 SessionRelay 的 TypeScript 架构中：

### 优化点 1：将交接校验升级为“Proof-or-Stop 证据门”
- **原设计**：SessionRelay 的 `handoff` 模块检查 `artifacts` 列表非空，但未核验本地真实文件是否存在。
- **借鉴吸收**：在 SessionRelay 的 `handoff.ts` 校验逻辑中引入实体核验：
  - 检查 `artifacts` 引用的文件在本地文件系统是否真实存在；
  - 校验文件大小是否合法（非空文件）；
  - 记录文件的本地 SHA-256 哈希作为不可篡改证据；
  - 若模型“空口谎报交付”，立即拦截并置为 `needs_attention`。

### 优化点 2：建立投递前“配额与模型防踩坑哨兵”（Cost & Model Guard）
- **原设计**：SessionRelay 仅做暂停门与会话占用检查。
- **借鉴吸收**：引入 LoopX 的 `should-run` 成本门禁：
  - 在事务 G（发送前门）中加入平台模型参数校验；
  - 若连接配置为测试环境，强制校验平台参数必须锁定低成本模型（如 Codex 必须是 `gpt-5.6-luna`，且带 `-c model_reasoning_effort="low"`）；
  - 严禁意外下发至高成本模型（如 `gpt-6-astra`），防止自动化测试意外消耗大量配额。

### 优化点 3：规范化交接信封（Handoff Envelope）的标准字段结构
- 借鉴 LoopX 经过实践检验的交接格式，将 SessionRelay 的交接 payload 标准化为 4 个清晰分区：
  1. `proven_facts`：本轮已实测证明的事实（防模型幻觉）；
  2. `verified_artifacts`：真实存在的交付文件指针与哈希；
  3. `open_decisions`：当前暴露的未决问题或需要审查方/用户拍板的事项；
  4. `next_instructions`：针对对侧角色的明确阶段任务提示。

---

## 5. 归档与后续落地步骤

1. 本分析作为正式外部参照归档于 `docs/references/loopx-analysis.md`，并在 [docs/references.md](../references.md) 中建立索引。
2. 在 P1 最小接力开发时，直接在 TypeScript 中实现：
   - 实体文件存在性与 Hash 校验（证据门）；
   - Codex 启动参数的强制低配额守卫（`gpt-5.6-luna` + `low reasoning` 哨兵）；
   - 规范化交接 JSON 信封。
3. 坚持不引入 Python 外部运行时，保持 SessionRelay 轻量、无依赖、独立运行的桌面特性。
