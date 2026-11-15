# Super-agent 演进计划

把 agent-harness 从“单进程、单会话、同步跑完一轮”的 CLI 演进为支持双层架构、事件驱动、会话与 Loop 分离、Loop 与执行分离（daemon）、Watchdog，并可接入 Codex、Claude Code、DeepSeek Harness 和自定义 Loop 作为 Worker 的 Super-agent 运行时。

- **参考**：DeepSeek《Super-agent 设计探索》（Get 笔记 2026-11-02，[原文](https://mp.weixin.qq.com/s/zoS7GlReWPhfCjENGxXCdg)）；本地原型 `projects/dsh-superagent-prototype/`（领域模型见其 `docs/001-domain.md`、`docs/003-execution.md`、`docs/007-architecture.md`）。
- **原则**：一次只实现一个能力；每一步都可运行、有验收标准、不破坏已有能力。步骤完成后在[进度](#进度)表登记。
- **状态**：设计稿（2026-11-02），尚未开发。各步骤开工前再细化接口和数据结构，并回写本文。

## 1. 现状与差距

| 现状（第一、二讲） | 目标需要 |
| --- | --- |
| `runTurn` 以一条用户输入驱动、同步跑完一轮 | Loop 以收件箱为输入，每次只推进一步，可以在任意进程中继续 |
| Session 是本地 JSONL，单进程写入 | 状态存数据库，多个 Runner 通过租约互斥写入 |
| 工具在宿主进程内执行，沙箱在本机 | 工具通过 daemon 在任意执行机上执行，沙箱和凭据留在执行机 |
| 只有一个会话角色 | Router、Worker、Watchdog 三种角色，Worker 可以是外部 Agent |
| 输入只有 CLI | 飞书、GitHub 等 Source 产生 Trigger |

可以复用的部分：`llm` 契约与适配器、事件 v2 格式与校验、消息投影、中断修复（`tool.result_unknown` 等）、权限策略与 Seatbelt 沙箱。

## 2. 核心抽象

### 2.1 所有输入都是投递到会话收件箱的事件

> **Session = 持久化收件箱（inbox）+ 追加写的事件日志（log）**
> **Loop = 无状态的单步函数：读 Session，取出收件箱全部新消息，推进一步，写回事件**

人的消息、Trigger、异步工具结果、Worker 汇报、Watchdog 提醒，全部是某个 Session 收件箱里的一条消息。由此得到：

- **存算分离**：Loop 不持有状态，任何 Runner 都能领取任意 Session 的下一步。
- **异步 ReAct**：工具 `start` 立即返回 handle，结果作为新的收件箱消息唤醒 Session。
- **Router 批处理**：一步开始时一次取出全部积压消息。
- **订阅直通**：程序直接写入已订阅 Worker 的收件箱。
- **Watchdog 干预**：写提醒消息或回收租约，不需要特殊通道。

### 2.2 三个平面

```
控制面（可上云）：Sources → Engine（领域核心，唯一写者）→ Store
                              Runner × N（无状态，跑托管 Loop）
                              Exec Gateway（WebSocket 服务端）· Outbox · 控制 API
执行面（任意机器）：daemon（出站连接，无会话状态，带沙箱与本地凭据）
                    ├─ 工具任务：read / write / exec ...
                    └─ Agent 任务：托管 Codex / Claude Code / dsh 进程
旁路：Watchdog（独立进程 + 独立 Session，定时驱动）
```

### 2.3 Worker 后端：托管 Loop 与嵌入式 Agent

Worker 统一通过 `WorkerBackend` 接入。不同后端的差别在于 **Loop 在哪里跑、状态在哪里存**：

| 类型 | Loop 运行位置 | 工具执行 | 会话状态 | 例子 |
| --- | --- | --- | --- | --- |
| **托管 Loop**（hosted） | 控制面 Runner | 经 Exec Gateway 派发给 daemon | 控制面 Store（完全存算分离） | 原生 Loop、自定义 Loop |
| **嵌入式 Agent**（embedded） | 执行机上的 daemon 托管其进程 | Agent 自己在本机执行 | 外部 Agent 自己的存储，控制面保存镜像和外部会话 ID | Codex、Claude Code、DeepSeek Harness |

外部 harness 自带 Loop 和工具，而且它们的工具必须在代码所在的机器上执行，因此**整个 Agent 进程跑在 daemon 侧**，控制面只做编排。这类 Worker 的存算分离是**部分的**：执行机离线时 Worker 暂停，重新上线后用外部会话 ID 恢复。这是有意识的取舍，不在控制面重新实现它们的 Loop。

接口草案（实现时细化）：

```js
// 托管与嵌入式共用
WorkerBackend = {
  kind: 'hosted' | 'embedded',
  capabilities: { resume, midTurnInput, interrupt, approvalBridge, mcp },
  start(ticket, { machine, cwd, instructions }) → { externalSessionId? },
  deliver(inputs)     // 收件箱消息 → 后端输入；不支持 midTurnInput 时等当前轮结束
  interrupt(), resume(externalSessionId), close(),
}
// 输出统一成事件流，镜像进 Worker Session 日志：
// worker.message / worker.tool_activity / worker.turn_completed /
// worker.approval_requested / worker.error / worker.report
```

**控制工具通过 MCP 桥接**：嵌入式 Agent 没有 `report`、`subscribe`、`handback`、`recall` 这些 Super-agent 工具。由 daemon 在本地起一个 MCP server，把这些工具代理到控制面。Codex、Claude Code、dsh 都支持 MCP，这样不用改外部 Agent 本身。

**自定义 Loop**：`core` 定义 `Loop` 接口（`step(ctx) → events`），原生 Loop 是其中一个实现；自定义 Loop 按名字注册，以托管方式运行，自动获得存算分离、daemon 执行和 Watchdog 监控。

已知的外部接入面（开工前再核实版本与细节）：

| 后端 | 接入方式 | 依据 |
| --- | --- | --- |
| Codex | `codex app-server`，stdio JSON-RPC：`thread/start`、`thread/resume`、`turn/start`、中断与审批请求 | 本地 `codex-harness/codex-rs/app-server/README.md` |
| DeepSeek Harness | ACP server（JSON-RPC stdio）：创建、恢复、关闭会话，挂 MCP，发 prompt，取消，接收语义更新，应答权限 | 本地 `deepseek-harness/packages/acp/README.md` |
| Claude Code | Claude Agent SDK，或 `claude -p` 的 stream-json 输入输出加 `--resume` | 待核实 |
| 统一协议 | 评估是否统一走 ACP（社区有 Codex 和 Claude Code 的 ACP 适配器），以减少适配器数量 | 待核实 |

### 2.4 不变量

1. Engine 是 Trigger、Thread、Subscription、Ticket、Outbox 的唯一写者，每条命令一个事务（借鉴原型）。
2. 同一 Session 同一时刻只有一个持有租约的 Runner；写回事件带租约校验。
3. 有副作用的工具调用以 `job_id` 幂等，结果未知时不自动重试。
4. 权限在执行侧强制执行，不信任控制面传来的参数；凭据只在执行机注入，不进入模型上下文。
5. Worker 不直接对外发言，对外消息统一走 Outbox。
6. Router 没有执行类工具。

## 3. 实施步骤

每一步一个能力。“验收”是这一步完成的判断标准。

### 阶段 A：内核重构（为后续所有能力打地基）

| 步骤 | 能力 | 验收 |
| --- | --- | --- |
| A1 | **Session 收件箱**：新增 `inbox.posted` / `inbox.consumed` 事件与消费游标；CLI 输入改为投递到收件箱 | 现有 CLI 行为不变；一步之内能一次取出多条积压消息 |
| A2 | **可重入单步**：`runTurn` 拆为 `runStep(sessionState)`，状态从日志重建；“轮”只是事件边界 | 任意一步之后杀掉进程，重新加载 Session 能继续，不重复执行工具 |
| A3 | **Loop 接口**：抽出 `Loop.step(ctx)`，原生 Loop 作为默认实现，支持按名字注册自定义 Loop | 提供一个最小自定义 Loop 示例，并通过同一组测试 |
| A4 | **Store 抽象 + SQLite 后端**：Session、事件、收件箱入库；JSONL 保留为导出格式 | 两个后端跑同一组测试；SQLite 支持并发读和单写者 |

### 阶段 B：会话与 Loop 分离

| 步骤 | 能力 | 验收 |
| --- | --- | --- |
| B1 | **Runner 进程**：领取 `runnable` Session，加租约与心跳，只跑一步就释放；收件箱有新消息即唤醒 | 两个 Runner 并发跑多个 Session，不会同时写同一个 Session |
| B2 | **崩溃恢复**：租约过期后重新领取；处理已开始但没有结果的步骤 | `kill -9` 正在跑的 Runner，另一个接手，结果正确 |
| B3 | **控制 API**：HTTP 或 JSON-RPC，提供创建 Session、投递收件箱、订阅事件流；CLI 改为客户端 | CLI 经 API 完成第一讲的案例；断开重连后能补看事件 |

### 阶段 C：Loop 与执行分离（daemon）

| 步骤 | 能力 | 验收 |
| --- | --- | --- |
| C1 | **执行协议 + Gateway + daemon 骨架**：daemon 出站 WebSocket、注册机器与能力、心跳；`read_file`、`write_file`、`exec` 迁到 daemon，沙箱在 daemon 内执行 | 本机 daemon 下第二讲的六个权限实验结果不变 |
| C2 | **异步工具**：`start` 立即返回 handle，`stop` 可终止，结果回投收件箱；长命令期间 Session 仍能响应新消息 | 跑一个 60 秒命令期间提问能得到回答；`stop` 生效 |
| C3 | **离线与幂等**：daemon 离线时任务排队，上线后派发；重连补发未确认结果，按 `job_id` 去重 | 执行中拔掉 daemon 再接上，结果到达且只到达一次 |
| C4 | **多机路由与本地凭据**：按 `machine_id` 和工作区选择执行机，凭据由 daemon 本地注入 | 两台执行机（本机加一个容器）各自执行；模型日志中看不到凭据 |

### 阶段 D：双层架构（先只用原生 Worker）

| 步骤 | 能力 | 验收 |
| --- | --- | --- |
| D1 | **Ticket 与 WorkerBackend 抽象**：实现原生托管后端，Ticket 绑定 Worker Session 和执行机 | 通过 API 创建 Ticket，Worker 在 daemon 上完成任务 |
| D2 | **Router 会话**：轻量模型，工具为 `accept`、`ignore`、`attach`、`spawn_worker`、`note_worker`、`reply`、`recall`、`ask_status`，没有执行类工具 | 连续提三个任务，Router 立即应答并并行分派 |
| D3 | **汇报与 Outbox**：Worker `report`（进行中、提问、完成、阻塞、失败）回到 Router 收件箱，Router 经 Outbox 统一回复 | Worker 的提问经 Router 转给人，人的回答回到对应 Worker |
| D4 | **Worker 生命周期与并发配额**：空闲、恢复、结束、重试；名额满时排队，并为人的消息预留一个名额 | 名额满时人的追问仍能被处理 |

### 阶段 E：外部 Agent 作为 Worker

| 步骤 | 能力 | 验收 |
| --- | --- | --- |
| E1 | **daemon 托管 Agent 任务 + MCP 桥接**：daemon 启动并看护长期运行的 Agent 进程，把输出转为 `worker.*` 事件；本地 MCP server 代理控制工具 | 用一个假 Agent 进程跑通启动、投递、中断、汇报、恢复 |
| E2 | **Codex 后端**（app-server） | Codex Worker 完成任务、汇报、被中断后能恢复 |
| E3 | **Claude Code 后端**（Agent SDK 或 stream-json） | 同上 |
| E4 | **DeepSeek Harness 后端**（ACP） | 同上；评估 ACP 能否统一 E2、E3 |
| E5 | **审批桥接**：外部 Agent 的审批请求按策略自动处理，或经 Router 转给人 | 敏感命令触发审批，人批准或拒绝后结果回到对应 Agent |

### 阶段 F：事件驱动

| 步骤 | 能力 | 验收 |
| --- | --- | --- |
| F1 | **Source 插件与 Trigger/Thread 模型**：第一个 Source 接飞书（`lark-event` 长连接），Trigger 去重入库并归入 Thread | 群消息按话题归为 Thread，重复事件只入库一次 |
| F2 | **两级过滤**：Fast Path 用配置声明白名单、群范围和来源处理方式；Slow Path 由 Router 按提示词策略判断 | 闲聊被程序滤掉；候选事件由 Router 判断；修改策略不需要改代码 |
| F3 | **订阅直通**：Worker 订阅 Thread（多对多），后续事件由程序直接写入 Worker 收件箱，Router 只收到摘要；支持 `handback` 和取消订阅 | 同一话题的追问不经过 Router 决策，直达 Worker |
| F4 | **Inline & Spill + 链式召回**：对象带 ID，注入时只放摘要和关联 ID，超长内容截断并留下 `recall` 入口 | Router 能顺着消息、Thread、Worker 一跳一跳找回旧任务的上下文 |
| F5 | **多信道输出规范**：IM 每次不超过 100 词；GitHub、文档用结构化交付；按信道套用 Outbox 模板 | 同一结果在 IM 和 GitHub 上呈现两种形态 |
| F6 | **第二个 Source**：GitHub（PR、评论、CI） | 一个任务在飞书与 PR 评论之间连续推进 |

### 阶段 G：Watchdog

| 步骤 | 能力 | 验收 |
| --- | --- | --- |
| G1 | **确定性巡检**：独立进程，定时检查 Runner 和 daemon 心跳、过期租约、积压的收件箱、Outbox 失败、长时间处于 running 的 Session；可回收租约、重派、告警 | 人为制造各类故障，都能被发现并恢复或告警 |
| G2 | **语义巡检 Session**：低频抽样 Worker 最近的步骤，判断有没有实质进展、是否偏离目标；可以提醒、中断或上报人 | 构造一个死循环 Worker，被识别并处理 |
| G3 | **自然语言运维**：可以向 Watchdog 询问故障原因，让它调整提示词或过滤规则（变更需留痕） | 对话式查询一次故障根因 |

### 阶段 H：增强与部署

| 步骤 | 能力 | 验收 |
| --- | --- | --- |
| H1 | **投机执行**：按策略静默尝试，有结果才交付，否则丢弃 | “静默复现并修复”策略跑通一次 |
| H2 | **回放评测**：参照论文的消融实验，用合成消息流对比单 Session 与双层架构（可复用 `agent-evals`） | 输出完成时间分布与 token 开销 |
| H3 | **控制面上云**：Store 换 Postgres，控制面部署到 VPS，本机作为 BYOM 执行机接入 | Mac 合盖后控制面继续运行，重新开机后待执行任务自动派发 |

### 顺序理由

- A、B 是地基：收件箱和可重入单步做好后，双层架构、异步工具和事件驱动都只是在上面增加 Session 角色和消息来源。
- C 在 D 之前：Worker 的执行要依赖 daemon。C1 的协议一开始就区分“工具任务”和“Agent 任务”两类，为 E 预留位置。
- D 先只用原生 Worker，把编排逻辑跑通后再在 E 接外部 Agent，避免两类问题混在一起排查。
- E 和 F 互不依赖，可以按需求调换先后。

## 4. 工程结构演进

```
packages/
  llm/            已有：模型契约与适配器
  session/        已有 → 增加收件箱、租约字段和 worker.* 事件
  core/           已有 → Loop 接口、原生 Loop、工具注册
  store/          新增：Store 接口，SQLite 实现，之后加 Postgres
  exec-protocol/  新增：控制面与 daemon 之间的消息定义
  engine/         新增：Trigger、Thread、Subscription、Ticket、Outbox 领域核心
  workers/        新增：WorkerBackend 及 native、codex、claude-code、dsh 适配器
apps/
  cli/            已有 → 逐步改为控制 API 的客户端
  runner/  gateway/  daemon/  watchdog/   新增
```

依赖方向保持单向：`engine → session/store`，`core → llm/session`，`apps/* → packages/*`；`exec-protocol` 被 gateway 和 daemon 共同依赖，不依赖其他包。

## 5. 待决问题

- 首个场景偏个人助手还是研发协作，决定 F1 之后先接哪些 Source 和 Router 策略。
- 控制面何时上云；在 H3 之前都按本机多进程运行。
- 嵌入式 Agent 的会话镜像粒度：完整镜像还是只保存摘要加外部引用。
- 安全：云端被攻破时如何限制 daemon 的权限（例如 daemon 侧的白名单、按 Ticket 发放临时令牌）。

## 进度

| 步骤 | 状态 | 完成日期 | 记录 |
| --- | --- | --- | --- |
| A1 | 未开始 | | |
