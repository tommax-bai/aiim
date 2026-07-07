## Why

新建的微信操作与运营服务（AIIM）需要第一个可迭代的**纵切能力**把核心骨架立起来。选「加友闭环」作首个能力，因为：

1. **它是产品入口**——整个服务的触发点就是「外部系统推来『给客户 X 加微信』指令」。没有加友闭环，后续对话与运营都无从谈起。
2. **它把本服务最硬的红线逼到最难处**——「绝不静默假成功」。企微协议**没有「加好友成功/失败」回调**：调 `add_*` 接口返回 `ok` ≠ 请求真送到对方 ≠ 对方通过。真通过只能靠通讯录变更信号 `FriendChange(2131)` + `sync_contact` 增量确认，或轮询好友列表。若不建一套「发起 → pending → 事件/轮询确认 → 超时判失败」的状态机，就会把「回执 ok」误当「加上了」，触发下游对空气对话。

这条能力立住后，后续对话主链、跟进 SOP、群/朋友圈运营都复用同一套「后置校验 + 风控闸 + 拟人节奏 + 多租户」骨架。

## What Changes

- **建立微信域契约的最小子集**（`packages/contracts`）：入站事件 `friend.request_received` / `friend.accepted` / `friend.rejected` / `friend.expired` / `op.result`；出站指令 `friend.add` / `friend.accept`（用稳定 `wxid` 寻址，带可选 `preAddDelayMs`）；以及加友任务/账号的最小类型。**不含** AIDCP 的 DOM 定位契约。
- **gateway 加友原子操作 + 归一化**（`apps/gateway`）：出站封装 `add_search_wx_contact` / `add_room_contact` / `agree_contact` 等；入站把协议 `2131`/`2132` 回调归一化为标准事件；`appinfo`/`seq` 去重；即时本地校验（回执 `ok`/`err`、`is_svr_fail`），**如实回报，绝不假成功**。
- **加友闭环角色 + 状态机**（`apps/brain`）：受理去重 → 选号 → 加友风控闸 → 发起 → 后置校验「请求真发出」→ 标 `pending` → 靠 `2131`+`sync_contact`（事件优先、轮询兜底）确认「对方真通过」→ 超时判失败（可换号/隔日重试，失败到顶升级）→ 加成功发 `first_touch.needed` 交棒对话闭环。
- **加友风控闸 + 加友配额**（复用 `packages/kernel` 的 `RiskController`，参数化 `RiskAction` 为微信动词）：`canDo('add_friend')` 查当日加友配额/通过率/时间窗/账号状态；**配额背压 ≠ 风控信号**（撞自己配额只背压、不自升状态）。
- **加友拟人节奏**：连续加友间隔中心值 `preAddDelayMs` 收口决策端，执行端只叠抖动、保下限。
- **被动加友受理**：`friend.request_received` 判自动通过/挂人审，通过后同样交棒首触。

## Capabilities

### New Capabilities

- `friend-add-closed-loop`：加友全链路行为契约——受理去重、选号归属、风控闸、发起、后置校验、事件/轮询确认真通过、超时判失败与升级、加成功交棒首触、被动加友受理；核心不变量是「三真相时刻（回执 ≠ 送达 ≠ 对方通过）绝不静默假成功」。

## Impact

- **aiim-service `apps/gateway`**：企微协议 HTTP 客户端（add_*/agree_contact/sync_contact）+ webhook 入站归一（2131/2132）+ appinfo/seq 去重。
- **aiim-service `apps/brain`**：加友闭环角色（受理/选号/风控闸/后置校验/等待通过/被动受理）+ 加友任务状态机 + `first_touch.needed` 交棒点。
- **aiim-service `packages/contracts`**：`friend.*` 事件/指令 + 加友任务/账号最小类型。
- **aiim-service `packages/kernel`**：从 `aidcp-cloud` 复制 `RiskController` + 状态机 + 滑窗 + quotas，**参数化** `RiskAction` 为微信动词、重定义加友配额三档、删/换 XHS 浅耦合（likeRatio/zeroInteractions）。
- **aiim-service `packages/store`**：加友任务持久化（状态机跨重启）+ appinfo 去重水位 + 账号风控计数。
- **依赖（先决，列为 task 不阻塞 spec）**：协议服务商在「送达/撤回回执」「`2131` 是否上报」「能否拿 wxid 稳定身份」「被限流信号」上的能力矩阵——直接决定「真通过」靠事件还是靠轮询。spec 用「事件优先、轮询兜底」抽象**不锁死单一提供方**。
- **红线验收**：新增 `AC-ADD-*`（加友三真相时刻）、`AC-RISK-*`（配额背压不自升状态、被禁号 `record` 返 false）。
- **不在本 change（DO NOT TOUCH）**：对话主链、跟进 SOP、群、朋友圈、标签画像、转人工、console/panel UI——后续 change 各自纵切。
