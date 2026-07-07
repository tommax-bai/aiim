# Project: AIIM

AIIM（微信操作与运营服务）——承接外部「给客户加微信」指令，自动加好友、加成功后由外部 AI
（异步回调）提供话术、我方负责微信收发与运营、并统一管账号风控。我们是用户 ↔ 外部 AI 的**中转站**，
存对话记忆、管操作与运营、控风控，对话智能外置。设计范式借鉴 AIDCP（事件驱动多角色 + 风控状态机单写 +
绝不静默假成功 + 拟人节奏 + 多租户），删掉浏览器 DOM 定位那一整套。

## 仓库家族（本仓为控制仓）

- `aiim`（本仓，`.`，分支 `main`）：控制仓 —— 契约 / 设计 / openspec change / 部署编排，**无业务代码**。
- `aiim-service`（`../aiim-service`，分支 `main`）：后端 monorepo —— `apps/{gateway,brain,scheduler,panel}` + `packages/{kernel,contracts,store}`。
- `aiim-console`（`../aiim-console`，分支 `main`）：管理后台前端 —— 只读 panel API + 经 `/api` 下发，绝不直连微信网关。

无 edge 仓：微信=HTTP+webhook 服务器对服务器，无边云物理约束。

## 关键约束（写 change 时必须遵守）

- **绝不静默假成功**：加友（回执≠送达≠对方通过，靠 2131+sync_contact 确认）、发消息（回执≠收到≠未撤回，appinfo 去重）、打标/群操作（写后回读）、外部 AI 超时/空/违规（挂起+告警，绝不假发）。
- **风控单写**：账号终态只由 `RiskController` 写（normal→warned→restricted→frozen）；配额背压 ≠ 风控信号。
- **中转站存记忆**：对话历史由我方 `ConversationStore` 权威持久化；外部 AI 只生成，经 `DialogBrainProxy` 唯一收口（异步回调、幂等、超时熔断）。
- **拟人节奏收口决策端 + 多租户 + 行动仲裁器**：内部并发收敛成「一个账号像一个人操作一部手机」。
- **复用 AIDCP 内核**：`packages/kernel` 复制内联 + 泛型化，删 DOM 定位遗产，不抽独立库、不依赖整个 aidcp-edge。
- 不记录任何敏感值（协议 guid / token / PG 密码 / AI key）。

## 工作流

spec-driven：所有跨模块 / 契约 / 风控 / 部署改动走 openspec change（proposal → design → tasks → apply →
`validate --strict` → archive）。代码落子仓，进度回写本控制仓。详见 `AGENTS.md` / `CLAUDE.md`。

## 文档索引

`README.md`、`docs/business-design.md`、`AGENTS.md`、`CLAUDE.md`。
