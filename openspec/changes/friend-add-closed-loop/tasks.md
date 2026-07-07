# Tasks — friend-add-closed-loop

> 代码落 `../aiim-service`，进度回写本文件（按模块分节，标 commit-sha）。红线：`AC-ADD-*` / `AC-RISK-*` 必过。

## 0. 先决（不阻塞契约定稿）

- [ ] 0.1 协议服务商能力矩阵摸底：`2131` 是否上报、`sync_contact` 增量粒度、能否稳定拿 `wxid`、`is_svr_fail`/被限流信号是否可得 → 定「真通过」走事件还是轮询

## 1. aiim-service — packages/kernel（复制 AIDCP 内核 + 泛型化/参数化）

- [x] 1.1 从 `aidcp-cloud/src/risk/` 复制 `RiskController` + `risk-state-machine` + `sliding-window-counter` + `quotas`；泛型化 `EventBus<TMap>` / `RiskController<A>` <!-- aiim-service e39e455 registry 暂未复制（单账号先直接实例化，多租户 registry 待 4.7） -->
- [x] 1.1b 泛型化 `EventBus<TMap>`（从 aidcp-cloud/src/event-bus 复制，去掉硬 import 的 XHS AllEventMap） <!-- aiim-service e39e455 -->
- [x] 1.2 参数化 `RiskAction` 为微信动词（`add_friend`/`accept_friend`/…）、重定义加友配额三档（养号/正常/激进）、删/换 `likeRatioAllowsNextLike` 与 `zeroInteractions` 的 XHS 浅耦合 <!-- aiim-service e39e455 三处 XHS 硬编码换成注入式 RiskPolicy（restrictedAllowedActions/warnedPausedActions/ratioGuard）；WECHAT_RISK_POLICY 在 packages/contracts/risk-policy.ts -->
- [ ] 1.3 复制 `humanize/timing`（对数正态抖动）+ pacing `tempoForStatus` 骨架；lint 禁止 kernel 反向依赖 apps/contracts

## 2. aiim-service — packages/contracts

- [x] 2.1 入站事件：`friend.request_received` / `friend.accepted` / `friend.rejected` / `friend.expired` / `op.result`（+ `friend.add_requested` 受理入口 / `first_touch.needed` 交棒） <!-- aiim-service e39e455 events.ts + WechatEventMap -->
- [x] 2.2 出站指令：`friend.add`（wxid/手机号 + 渠道 + 申请语 + `preAddDelayMs`）/ `friend.accept` <!-- aiim-service e39e455 commands.ts + WechatCommandMap -->
- [x] 2.3 加友任务/账号最小类型（状态枚举 received→accepted/failed、身份键、去重键 `targetKey`） <!-- aiim-service e39e455 friend-add.ts -->
- [x] 2.4 版本化信封 `Envelope<TType>`（沿用 AIDCP 信封思想、泛型化，供未来跨进程/回放/审计） <!-- aiim-service e39e455 envelope.ts -->

## 3. aiim-service — apps/gateway

- [ ] 3.1 出站：封装 `add_search_wx_contact` / `add_room_contact` / `agree_contact` / `sync_contact`（回执归一为 `op.result`，`is_svr_fail` 如实回报，绝不假成功）
- [ ] 3.2 入站：`2131`/`2132` 回调归一化为标准事件；`appinfo`/`seq` 去重水位
- [ ] 3.3 「真通过」确认：`2131`→触发 `sync_contact` 增量确认目标进好友列表；提供方不可靠时退避轮询兜底

## 4. aiim-service — apps/brain（加友闭环角色 + 状态机）

- [x] 4.1 受理去重角色：校验/去重(已好友/已 pending/黑名单)，无效如实拒 <!-- aiim-service 8e0de95 coordinator intake（findActiveByTargetKey 跨号去重 + no_target/blacklist/already_friend 如实拒） -->
- [x] 4.2 选号角色：按当日加友配额选号（号龄/垂类打分待接）；无可用号 `deferred` 排队 <!-- aiim-service 8e0de95 selectAccount（canDo 为准），号龄/垂类打分留 TODO -->
- [x] 4.3 加友风控闸：`canDo('add_friend')`，准→带 `preAddDelayMs`（按 tempo 缩放）、拒→背压 deferred（不扣预算、不自升状态） <!-- aiim-service 8e0de95 -->
- [x] 4.4 发起 + 后置校验：记账占额 → 下发 → `op.result` 确认「请求真发出」标 `pending`（回执 ok 绝不判成功/触发首触） <!-- aiim-service 8e0de95；超时改由 sweepTimeouts 巡视驱动，非 setTimeout -->
- [x] 4.5 等待通过角色：`friend.accepted`(实证)→`accepted`+`first_touch.needed`；`rejected`/`expired`/超时→`failed`；连续失败到顶升级+置风控态+告警；**协调器不自动重发** <!-- aiim-service 8e0de95 -->
- [x] 4.6 被动加友受理角色：`friend.request_received` 非可疑→`friend.accept`、可疑关键词→挂人审告警 <!-- aiim-service 8e0de95 -->
- [ ] 4.7 多租户：`op.result`/`2131` 按账号定向、不串号 <!-- 部分：per-account RiskController + accounts() 选号已在；连接绑定/定向回发的完整路由待接 -->
- [ ] 4.8 号龄/垂类选号打分 + 加友通过率 ratioGuard 实接（当前 selectAccount 仅按配额、ratioGuard 占位放行）

## 5. aiim-service — packages/store

- [x] 5.1a 加友任务存储接口 + 内存实现（`FriendAddStore` / `InMemoryFriendAddStore`：CRUD + 按 targetKey/requestId 查 + listPending + 黑名单/已好友 + 连续失败计数） <!-- aiim-service 8e0de95 -->
- [ ] 5.1 PG 持久化（状态机跨重启）+ 身份稳定化（wxid 先暂存后晋升）—— 内存版已就位，PG 实现替换接口即可
- [ ] 5.2 appinfo 去重水位 + 账号风控计数持久化（PG）

## 6. 验收（红线套件，落 apps/*/test）

- [x] 6.1 `AC-ADD-*`：回执 ok 不判成功；无 `friend.accepted` 实证前不触发首触；找不到目标报 `no_target` <!-- aiim-service 8e0de95 apps/brain/test/friend-add.test.ts -->
- [x] 6.2 `AC-ADD-*`：超时判失败、连续失败到顶升级、绝不无限重发 <!-- aiim-service 8e0de95 -->
- [x] 6.4 去重：同一 `(账号,目标)` 重复指令幂等、不重复发起 <!-- aiim-service 8e0de95 -->
- [x] 6.5 被动加友：自动通过 / 可疑挂人审两分支 <!-- aiim-service 8e0de95 -->
- [x] 6.3 `AC-RISK-*`：加友配额耗尽只背压不自升状态；被禁号 `record` 返 false <!-- aiim-service e39e455 packages/kernel/test/risk-controller.test.ts 6/6 过（含状态机迁移/restricted/warned 门控/ratioGuard） -->

## 7. 收口

- [ ] 7.1 `openspec validate friend-add-closed-loop --strict` 通过
- [ ] 7.2 端到端冒烟：1 测试号走 外部指令→加友→(mock 提供方 2131)→确认通过→`first_touch.needed`，人工核对失败如实上报
- [ ] 7.3 archive
