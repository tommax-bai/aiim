# AGENTS.md

本文是 aiim 家族的开发规范（Codex / Claude 通用）。`aiim` 是控制仓，承载契约、设计、openspec
与部署编排，**不含业务代码**；代码落到 `aiim-service`、`aiim-console`。

## 0. 仓库家族与路径

- `.`：`aiim`（控制仓，默认分支 `main`）—— 契约 / 设计 / openspec change / 部署编排。
- `../aiim-service`：后端（默认分支 `main`）—— monorepo：`apps/{gateway,brain,scheduler,panel}` + `packages/{kernel,contracts,store}`。
- `../aiim-console`：管理后台前端（默认分支 `main`）—— 只读 panel API + 经 `/api` 下发，**绝不直连微信网关**。
- 参考仓 `../aidcp`（AIDCP 家族）：内核复用来源；**只复制、不依赖整个仓**。

动子仓代码前先 `ls -d ../aiim-service ../aiim-console` 确认存在。

## 1. 架构不变量（跨文件的上位约束）

权威文档：`docs/business-design.md`。改动前先判断落在哪个 app / package：

- **逻辑三层、物理一体**：原子操作（调企微 HTTP API、收发消息、加友、群、朋友圈）在 `apps/gateway`；
  编排 / 选择 / 风控 / 持久化在 `apps/brain`；时间/状态驱动运营在 `apps/scheduler`；管理后台 API 在 `apps/panel`。
  三者异步事件 / 队列解耦、可独立部署，但**没有跨仓协议**（同 monorepo 共享 `packages/contracts` 类型）。
- **风控单写**：账号健康终态只由 `RiskController` 写（`normal→warned→restricted→frozen`）。其他只提事件 / 读投影。
  **配额背压 ≠ 风控信号**：撞自己配额只降速、绝不自升状态（否则自锁）。
- **红线：绝不静默假成功**（贯穿全部验收）：
  - 加友：`API 回执 ok ≠ 请求送达 ≠ 对方通过`——真通过靠 `FriendChange(2131)+sync_contact` 确认；找不到目标报 `no_target`。
  - 发消息：`回执 ok ≠ 对方收到 ≠ 未被撤回`；用 `appinfo` 去重；失败到顶升级、绝不无限重发。
  - 打标 / 群操作：写后回读校验。
  - 外部 AI 超时 / 空 / 违规：**绝不发占位或编造**，挂起 + 告警运营，如实记「内容未就绪」。
  - 身份稳定化：`wxid/roomid` 先暂存确认稳定才作主键，抓不到如实留空退次级键。
- **中转站存记忆**：对话历史由我方 `ConversationStore` 权威持久化；外部 AI 只负责生成，经 `DialogBrainProxy` 唯一收口（异步回调、幂等、超时熔断）。
- **拟人节奏收口决策端**：中心值（读延迟 / 打字时长 / 条间隔 / 加友间隔）在决策端算，执行端只叠抖动、保下限、断连兜底；绝不零延迟。
- **多租户**：每微信号 = 一束独立上下文 / 预算 / 风控账号，指令定向不串号，账号身份穿透握手。
- **运行时并发**：每 `(账号,客户)` 会话 = 一个 actor（片内串行、片间并发）；入站防抖聚合、AI 在途新消息则代号作废重来；每账号「行动仲裁器」按优先级出队，收敛成「一个账号像一个人操作一部手机」。

## 2. 复用 AIDCP 内核的纪律

- `packages/kernel` 是从 `aidcp-cloud` **复制内联**的领域无关机制（event-bus / risk 状态机+滑窗+quotas / feishu 管道 / llm / pacing 的 tempo+fatigue / soul loader），从 `aidcp-edge` 复制 humanize 的 timing/session-rhythm/reading-time。**切勿依赖整个 aidcp-edge**（会拖进 Electron/CDP/jsdom）。
- 复制时**泛型化开口**：`EventBus<TMap>` / `RiskController<TAction>` / `Envelope<TType>`；微信侧在 `packages/contracts` 提供 `WechatEventMap` / `RoleName` / `MessageType`（wxid/roomid 寻址，**不要** anchor/select/plan 那套 DOM 定位）。
- **复制风控时消除三处 XHS 浅耦合**：`RiskAction` 动词换微信语义（add_friend/send_message/…）、重定义三档 quotas、删/换 likeRatio 护栏、`zeroInteractions` 放行动作 view→read_message。
- lint 禁止 `packages/kernel` 反向依赖 `apps/*` 或 `packages/contracts`（防微信契约漏进通用引擎）。
- **不抽独立共享库**：今天只有 aiim-service 一个消费者；出现第二消费者（或与 AIDCP 统一）再 lift-and-shift 成 `@aidcp/kernel`。

## 3. OpenSpec 工作流（spec-driven）

- 所有跨模块 / 契约 / 风控 / 部署行为改动走 openspec change：proposal → design → tasks → apply → `validate --strict` → archive。
- 不直改 `openspec/specs/`；在 `openspec/changes/<name>/` 写 proposal/tasks/design/spec delta。代码落子仓，进度回写本仓 `tasks.md`（按子仓分节，标 commit-sha）。
- 校验：`openspec list` / `openspec validate <change> --strict`。

## 4. 红线验收套件（固化到 CI）

- `AC-ADD-*`：加友三真相时刻（回执≠送达≠通过，靠事件/轮询确认）。
- `AC-MSG-*`：消息送达校验、appinfo 去重、外部 AI 空返回不假发。
- `AC-RISK-*`：配额背压不自升状态、被禁号 record 返 false、风控单写。
- `AC-TAG-*`：打标/群操作写后回读。

## 5. Git / 沟通 / 安全

- 语言：正文中文；代码 / 注释 / commit / PR / 文件名英文。
- 不记敏感值（协议 guid/token/PG 密码/AI key）：只记路径、用法、配置读取方式。
- commit message 末尾带 `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`。
- 提交 / 推送 / 部署按用户授权执行；force-push、非 fast-forward、推 protected 分支需先确认。
- 每次收尾给一段「说人话」的总结：做了什么、对系统什么影响、下一步。
