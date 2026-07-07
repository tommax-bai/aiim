## ADDED Requirements

### Requirement: 加友指令受理与幂等去重

系统 SHALL 受理外部推来的「给客户加微信」指令并做校验 + 去重：已是好友、已有进行中（pending）任务、命中黑名单的目标 MUST NOT 重复发起，MUST 如实拒（返回明确原因），MUST NOT 静默吞。去重维度为 `(账号, 目标身份)`，目标身份优先 `wxid`、仅有手机号时以手机号为次级键。同一有效指令重复下发 MUST 幂等（不产生第二个加友任务）。

#### Scenario: 已是好友不重复加
- **WHEN** 指令目标已是该账号好友
- **THEN** 系统如实拒该指令、MUST NOT 再次发起加友

#### Scenario: 重复指令幂等
- **WHEN** 同一 `(账号, 目标)` 在已有 pending 任务时又收到加友指令
- **THEN** 系统 MUST NOT 新建第二个任务，视为同一任务

#### Scenario: 无效指令如实拒
- **WHEN** 指令缺目标身份或命中黑名单
- **THEN** 系统返回明确拒绝原因，MUST NOT 静默丢弃

### Requirement: 选号归属按配额，不硬塞超配额账号

系统 SHALL 为有效加友指令按号龄（冷启动养号档）、当日加友配额余量、垂类/地域匹配选定一个健康微信号承接。当无账号有加友配额余量时，任务 SHALL 进入排队（deferred）等待，MUST NOT 硬塞给已超当日加友配额或非 normal 风控态的账号。

#### Scenario: 无可用配额则排队
- **WHEN** 所有健康账号当日加友配额均已耗尽
- **THEN** 任务进入排队等待，MUST NOT 强行分配给超配额账号

#### Scenario: 选健康号承接
- **WHEN** 存在多个账号且部分有加友配额余量
- **THEN** 从有余量且风控态 normal 的账号中按号龄/垂类选一个承接

### Requirement: 加友风控闸，配额背压不自升风控状态

发起加友前系统 SHALL 经 `RiskController.canDo('add_friend')` 判定（当日加友配额、加友通过率、活跃时段、账号风控态）。判定为拒时任务 SHALL 背压（挂起/排队），MUST NOT 扣减预算、MUST NOT 因「撞自己配额」而自升账号风控状态。加友动作的配额档为最严档，与被动接待完全解耦。

#### Scenario: 撞配额只背压不升级
- **WHEN** `canDo('add_friend')` 因当日配额耗尽返回拒
- **THEN** 任务背压等待，账号风控状态 MUST NOT 由 normal 迁移到 warned/restricted

#### Scenario: 非活跃时段不主动加友
- **WHEN** 当前落在账号作息窗的休眠时段
- **THEN** `canDo('add_friend')` 拒，主动加友 MUST NOT 发起（被动接待不受此闸约束）

### Requirement: 发起后置校验，API 回执不等于成功

发起加友后系统 SHALL 依据 `op.result`（含 `is_svr_fail` 等本地信号）确认「请求是否真的发出」，仅在确认发出后才把任务标记为 `pending`。API 回执 `ok` MUST NOT 被当作「加友成功」，MUST NOT 据此触发首触。找不到目标会话/身份 SHALL 报 `no_target`，MUST NOT 静默当成功。

#### Scenario: 回执 ok 只进 pending 不判成功
- **WHEN** `add_*` 接口回执 `ok`
- **THEN** 任务进入 `pending` 等待对方通过，MUST NOT 标记为已加上、MUST NOT 触发首触

#### Scenario: 服务器失败如实回报
- **WHEN** `op.result` 显示 `is_svr_fail=true` 或找不到目标
- **THEN** 系统如实记失败/`no_target`，MUST NOT 静默假成功

### Requirement: 真通过只靠好友列表实证确认

任务从 `pending` 迁移到 `accepted` SHALL 仅当对好友列表的**实证**确认目标已进入好友列表——通过 `FriendChange(2131)` 事件触发 `sync_contact` 增量确认，或在事件不可靠时退避轮询 `sync_contact` 兜底。系统 MUST NOT 仅凭 API 回执或超时未失败就推断为通过。（本要求规定「必须实证」，不规定事件或轮询，二者皆满足。）

#### Scenario: 2131 + sync_contact 确认通过
- **WHEN** 收到目标相关的 `FriendChange(2131)` 并经 `sync_contact` 确认目标进入好友列表
- **THEN** 任务迁移到 `accepted`

#### Scenario: 无实证不判通过
- **WHEN** 任务在 pending 期间未收到任何好友列表实证
- **THEN** 任务 MUST NOT 迁移到 `accepted`，保持 pending 直至超时判失败

### Requirement: 超时判失败与升级，绝不无限重发

`pending` 任务超过等待时限仍无通过实证 SHALL 判失败（`failed:timeout`），可换号或隔日重试。某账号连续加友失败到达上限 SHALL 判系统性问题（号被限/提供方故障）→ 停手升级、置对应风控态、告警运营。系统 MUST NOT 对同一目标无限重发（无限重发本身即高封号风险）。

#### Scenario: 超时判失败可重试
- **WHEN** pending 任务超时仍无通过实证
- **THEN** 任务判 `failed:timeout`，允许换号或隔日重试（受去重与配额约束）

#### Scenario: 连续失败到顶升级停手
- **WHEN** 某账号连续加友失败达上限
- **THEN** 系统停手升级、置风控态、告警运营，MUST NOT 继续无限重发

### Requirement: 加成功交棒首触

任务迁移到 `accepted`（主动加成功或被动通过）SHALL 发出 `first_touch.needed` 事件，交棒对话闭环做首次打招呼。系统 MUST NOT 在此直接生成或发送话术（对话内容由对话闭环经外部 AI 产出），MUST NOT 秒发（首触节奏由下游拟人化处理）。

#### Scenario: 加成功触发首触事件
- **WHEN** 任务迁移到 `accepted`
- **THEN** 发出 `first_touch.needed`，MUST NOT 在加友闭环内直接发招呼

### Requirement: 被动加友受理

系统 SHALL 受理入站 `friend.request_received`（他人申请加我）：按申请语/黑名单/账号风控档判定自动通过或挂人审。判自动通过时发出 `friend.accept`，通过后同样进入 `accepted` 并交棒首触。被动接待的配额与主动加友完全解耦。

#### Scenario: 自动通过后交棒首触
- **WHEN** 入站好友申请通过自动通过判定
- **THEN** 发出 `friend.accept`，通过后进入 `accepted` 并发 `first_touch.needed`

#### Scenario: 可疑申请挂人审
- **WHEN** 入站好友申请命中需人工判定的条件
- **THEN** 挂起人审，MUST NOT 自动通过

### Requirement: 加友拟人间隔收口决策端

连续加友之间的间隔中心值 `preAddDelayMs` SHALL 由决策端按号龄/风控 tempo/时段算出，随 `friend.add` 指令下发；执行端只叠 lognormal 抖动并保证非零下限。系统 MUST NOT 零延迟连续发起加友、MUST NOT 让多账号在同一秒齐发加友请求。

#### Scenario: 连续加友有拟人间隔
- **WHEN** 一个账号连续处理多个加友任务
- **THEN** 相邻发起之间有 `preAddDelayMs` + 抖动的非零间隔，MUST NOT 零延迟连发
