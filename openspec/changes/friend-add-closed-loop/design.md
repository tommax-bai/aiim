# Design — friend-add-closed-loop

## 1. 为什么是「状态机」而不是「调完就完」

企微协议的加友是**异步**的，且**没有成功/失败回调**。三个事实必须分开对待（红线「三真相时刻」）：

1. `add_*` 接口回执 `ok` —— 只代表「请求被协议服务受理」，不代表送到对方。
2. 请求真送到对方 —— 需要 `op.result` 里 `is_svr_fail=false` 等本地信号佐证，仍不代表通过。
3. 对方**通过** —— 只能靠 `FriendChange(2131)` 事件 + `sync_contact` 增量确认对方进了好友列表，或轮询好友列表兜底。

因此加友必须建一个**每任务的状态机**，绝不能因回执 `ok` 就宣告「加上了」并触发首触（那会对空气说话）。

## 2. 加友任务状态机

```
[received] ──受理去重(已好友?已pending?黑名单?)──► 无效则 rejected(如实拒,不静默)
   │ 有效
   ▼
[account_selected] ──选号(号龄/当日加友配额/垂类)──► 无可用号则 deferred(排队,不假成功)
   │
   ▼
[risk_gated] ── canDo('add_friend') ──► deny 则 backpressured(不扣预算,不自升风控态)
   │ allow (带 preAddDelayMs)
   ▼
[dispatched] ── gateway 调 add_*，叠拟人间隔 ──► op.result
   │
   ▼
[pending] ── 标 pending, 记 requestId, 启超时定时器 ─────────────────────────┐
   │                                                                        │
   ├─ friend.accepted(2131+sync_contact 确认) ─► [accepted] ─► first_touch.needed
   ├─ friend.rejected / friend.expired ─────────► [failed] (风控信号: light)
   └─ 超时未确认 ───────────────────────────────► [failed:timeout] (可换号/隔日重试)
       连续失败到顶 ─► 升级 + 置风控态 + 告警运营 (绝不无限重发)
```

被动加友：`friend.request_received` → 判自动通过/挂人审 → `friend.accept` → 通过后同样进 `[accepted]` → `first_touch.needed`。

## 3. 「真通过」确认：事件优先、轮询兜底（不锁死提供方）

- **首选**：订阅 `2131` 归一事件 → 触发一次 `sync_contact` 增量 → 确认目标 `wxid` 进入好友列表（`ContactType` 为好友）。
- **兜底**：若提供方 `2131` 不可靠/不上报，则对 `pending` 任务按退避轮询 `sync_contact`。
- spec 只规定「必须靠**对好友列表的实证**确认通过、绝不靠 API 回执」，**不规定**必须用事件还是轮询——由 §先决摸底按提供方能力选，二者皆满足红线。

## 4. 关键决策

- **去重维度**：`(账号, 目标身份)`。目标身份优先 `wxid`；仅有手机号时以手机号为次级键，拿到 `wxid` 后做身份稳定化收敛（先暂存、确认稳定才作主键，防合并/拆分）。
- **选号**：按号龄（冷启动养号档）、当日加友配额余量、垂类/地域匹配打分选一个健康号；无可用号则 `deferred` 排队，绝不硬塞超配额的号。
- **加友风控闸**：`add_friend` 是**头号封号源**，配额档与被动接待完全解耦、取最严档；比例约束（加友通过率骤降 → 降档）实时算。
- **拟人间隔**：`preAddDelayMs` 中心值决策端算（号龄/风控 tempo/时段），执行端叠 lognormal 抖动、保下限——连续加友有自然长尾间隔是最直接的封号相关节奏。
- **失败与升级**：单任务超时判失败可换号/隔日重试；某账号连续加友失败到顶 → 判系统性问题（号被限/提供方故障）→ 停手升级、置风控态、告警，**绝不无限重发**（无限重发本身即高危）。

## 5. 复用 AIDCP 内核的落点

- `RiskController` + 状态机 + 滑窗 + quotas 从 `aidcp-cloud/src/risk/` 复制进 `packages/kernel`，**参数化** `RiskAction`（`add_friend`/`accept_friend`/…）、重定义加友配额三档、删/换 `likeRatioAllowsNextLike` 与 `zeroInteractions` 的 XHS 浅耦合。
- 拟人间隔用 `packages/kernel` 的 `humanize/timing`（对数正态抖动）+ pacing 的 tempo 骨架。
- 多租户：每账号一束上下文/预算/风控账号，`op.result` 与 `2131` 事件按账号定向、不串号。

## 6. 先决依赖（列为 task，不阻塞 spec 定稿）

协议服务商能力矩阵摸底：`2131` 是否上报、`sync_contact` 增量粒度、能否稳定拿 `wxid`、`is_svr_fail`/被限流信号是否可得。结论决定「真通过」走事件还是轮询、以及 `op.result` 能给到多细的本地校验——但都不改变 spec 的行为契约（红线不变）。
