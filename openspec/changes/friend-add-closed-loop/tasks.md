# Tasks — friend-add-closed-loop

> 代码落 `../aiim-service`，进度回写本文件（按模块分节，标 commit-sha）。红线：`AC-ADD-*` / `AC-RISK-*` 必过。

## 0. 先决（不阻塞契约定稿）

- [ ] 0.1 协议服务商能力矩阵摸底：`2131` 是否上报、`sync_contact` 增量粒度、能否稳定拿 `wxid`、`is_svr_fail`/被限流信号是否可得 → 定「真通过」走事件还是轮询

## 1. aiim-service — packages/kernel（复制 AIDCP 内核 + 泛型化/参数化）

- [ ] 1.1 从 `aidcp-cloud/src/risk/` 复制 `RiskController` + `risk-state-machine` + `sliding-window-counter` + `quotas` + `registry`；泛型化 `RiskController<TAction>`
- [ ] 1.2 参数化 `RiskAction` 为微信动词（`add_friend`/`accept_friend`/…）、重定义加友配额三档（养号/正常/激进）、删/换 `likeRatioAllowsNextLike` 与 `zeroInteractions` 的 XHS 浅耦合
- [ ] 1.3 复制 `humanize/timing`（对数正态抖动）+ pacing `tempoForStatus` 骨架；lint 禁止 kernel 反向依赖 apps/contracts

## 2. aiim-service — packages/contracts

- [ ] 2.1 入站事件：`friend.request_received` / `friend.accepted` / `friend.rejected` / `friend.expired` / `op.result`
- [ ] 2.2 出站指令：`friend.add`（wxid/手机号 + 渠道 + 申请语 + `preAddDelayMs`）/ `friend.accept`
- [ ] 2.3 加友任务/账号最小类型（状态枚举 received→accepted/failed、身份键、去重键）

## 3. aiim-service — apps/gateway

- [ ] 3.1 出站：封装 `add_search_wx_contact` / `add_room_contact` / `agree_contact` / `sync_contact`（回执归一为 `op.result`，`is_svr_fail` 如实回报，绝不假成功）
- [ ] 3.2 入站：`2131`/`2132` 回调归一化为标准事件；`appinfo`/`seq` 去重水位
- [ ] 3.3 「真通过」确认：`2131`→触发 `sync_contact` 增量确认目标进好友列表；提供方不可靠时退避轮询兜底

## 4. aiim-service — apps/brain（加友闭环角色 + 状态机）

- [ ] 4.1 受理去重角色：校验/去重(已好友/已 pending/黑名单)，无效如实拒
- [ ] 4.2 选号角色：按号龄/当日加友配额/垂类打分选号；无可用号 `deferred` 排队
- [ ] 4.3 加友风控闸：`canDo('add_friend')`，准→带 `preAddDelayMs`、拒→背压（不扣预算、不自升状态）
- [ ] 4.4 发起 + 后置校验：`op.result` 确认「请求真发出」，标 `pending`，启超时定时器
- [ ] 4.5 等待通过角色：消费 `friend.accepted`/`rejected`/`expired`/超时 → `accepted` 发 `first_touch.needed`；`failed` 记风控信号、超时可换号/隔日重试；连续失败到顶升级+置风控态+告警
- [ ] 4.6 被动加友受理角色：`friend.request_received` 判自动通过/挂人审 → `friend.accept` → 通过后交棒首触
- [ ] 4.7 多租户：`op.result`/`2131` 按账号定向、不串号

## 5. aiim-service — packages/store

- [ ] 5.1 加友任务持久化（状态机跨重启）+ 身份稳定化（wxid 先暂存后晋升）
- [ ] 5.2 appinfo 去重水位 + 账号风控计数持久化

## 6. 验收（红线套件，落 apps/*/test）

- [ ] 6.1 `AC-ADD-*`：回执 ok 不判成功；无 2131/轮询确认前不触发首触；找不到目标报 `no_target`
- [ ] 6.2 `AC-ADD-*`：超时判失败、连续失败到顶升级、绝不无限重发
- [ ] 6.3 `AC-RISK-*`：加友配额耗尽只背压不自升状态；被禁号 `record` 返 false
- [ ] 6.4 去重：同一 `(账号,目标)` 重复指令幂等、不重复发起

## 7. 收口

- [ ] 7.1 `openspec validate friend-add-closed-loop --strict` 通过
- [ ] 7.2 端到端冒烟：1 测试号走 外部指令→加友→(mock 提供方 2131)→确认通过→`first_touch.needed`，人工核对失败如实上报
- [ ] 7.3 archive
