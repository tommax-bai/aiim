# CLAUDE.md

This file guides Claude Code when working in the `aiim` family. It mirrors [AGENTS.md](AGENTS.md);
prefer AGENTS.md for detail, this file for the fast orientation.

> `aiim`（本仓，控制仓，分支 `main`）只承载**文档与契约**（`docs/` + `openspec/` + `README.md`），
> 无业务代码、无构建链。代码落 `../aiim-service`（后端 monorepo）与 `../aiim-console`（管理后台前端）。

## 速记

- **定位**：微信操作与运营服务；我们是用户 ↔ 外部 AI 的**中转站**，存对话记忆、管微信操作与运营、控风控，对话智能外置。
- **仓库家族（3 仓 = AIDCP 去掉 edge）**：`aiim`(控制) + `aiim-service`(后端 monorepo：apps gateway/brain/scheduler/panel + packages kernel/contracts/store) + `aiim-console`(前端)。无 edge——微信是 HTTP+webhook 无边云物理约束。
- **权威设计**：[`docs/business-design.md`](docs/business-design.md)。

## 三条铁律（改任何东西先记住）

1. **绝不静默假成功**：加友/发消息「回执 ≠ 真送达 ≠ 对方真收到/通过」，都要后置校验；外部 AI 超时/空绝不假发。
2. **风控单写 + 配额背压 ≠ 风控信号**：账号终态只由 `RiskController` 写；撞自己配额只降速、不自升状态。
3. **复用 AIDCP 内核靠复制+泛型化**（`packages/kernel`），删浏览器 DOM 遗产；不抽独立库、不依赖整个 aidcp-edge。

## 工作流

- spec-driven：跨模块/契约/风控/部署改动走 openspec change（proposal→tasks→apply→`validate --strict`→archive），代码落子仓、进度回写本仓。
- 语言：正文中文，代码/commit 英文。commit 末尾 `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`。不记敏感值。
