---
title: 自主巡逻（patrol）深度指南
layout: default
---

# 自主巡逻（patrol）深度指南

> 快速上手见主手册 [7.7 自主巡逻](../index.html#77-自主巡逻)。本页是自主巡逻的接口、参数与行为完整参考；主手册只保留跑通所需最小集，本页为唯一权威源。

## 1. 功能与行为

### 1.1 任务生命周期

发起 → 建候选池 → 逐点走访 → 终态：

1. **请求合法性检查**：`search_radius < 0` 直接拒绝（定向导航是自主搜索专属）；原地旋转（spin）任务运行中拒绝；已有任务运行中返回幂等成功（不中断当前任务）。
2. **候选池生成**：地图（包围盒）面积 ≥ `free_pool_map_area_thr` 才建池——方格采样全图自由点，经关键帧约束过滤（只保留真实观察过的区域）和 Nav2 路径验证（规划不达的点剔除，仅可视化）。**图过小或池为空 → 直接进入终态 `EXHAUSTED`**。
3. **走访循环**：全图巡逻按"距机器人实时位姿就近"选点，区域巡逻按池生成序；每个候选点导航前往，到访/失败/取消后处理下一个。循环巡逻（`patrol_loop_count`）时每遍池尽后重采样候选池并清空灰名单续巡，直至巡完指定遍数或被停止。
4. **终态**：见 1.2 状态表。

### 1.2 状态话题 /explore/search_status

消息类型 `tros_frontier_exploration/msg/StringStamped`，QoS 为 transient_local——**新订阅者立即收到最新一条状态**，无需等待下次发布。

| 状态 | 含义 |
|------|------|
| `IDLE` | 空闲（无任务） |
| `PATROL:n/N` | 巡逻进行中：本遍清单已办结 n 个 / 共 N 个去重候选点。办结 = 成功到访或确认失败；等待补巡的取消点未办结（n 暂不计） |
| `SPINNING:scan_blindspot[:n/N]` | 原地旋转扫描盲区（候选点间盲区补扫），旋转结束自动恢复走访；变体 `SPINNING:escape_obstacle`（脱困旋转）、`SPINNING:find_frontier` |
| `WAITING_MAIN_LOOP: waited=Ns` | 主循环 frontier 探索占用导航控制权，巡逻挂起等待（每 5s 心跳）；等待超 60 秒直接终态 `TIMEOUT:main_loop` |
| `WAITING_SUBMAP_FLOW: waited=Ns` | 子图回访/建图（submapLinkFlow）接管控制期间巡逻挂起，每 5s 心跳；flow 结束后自动重选目标继续 |
| `EXHAUSTED` | 走访会话完成（默认单遍即全部候选点处理完；循环巡逻为全部遍数巡完）；启动建池为 0 时也以此终态结束，并附加 `empty_reason` |
| `TIMEOUT` | 达到设置的巡逻时间后结束；若正在前往某个巡逻点，会先处理完当前点，再结束巡逻，不会立即中断当前导航（详见 1.4） |
| `CANCELLED:stop_requested` | 调用 `stop_patrol`（或 `stop_search`）停止 |
| `TRAPPED` | 机器人被困，恢复策略无效 |

附加规则：

- 进行中与终态消息附带时间进度 `t:elapsed/budget`（已用秒 / 预算秒）。末个目标跨越预算边界时 `elapsed` 可能大于 `budget`；超出幅度取决于当前目标剩余导航、结果处理及默认到点旋转时间，不保证只是少量超出。
- 循环巡逻（`patrol_loop_count ≠ 0`）额外附带 `pass:i/M`（当前遍 / 总遍数；不限次显示 `pass:i/-`）。
- `WAITING_SUBMAP_FLOW` 与 `SPINNING:*` 的等待时间**计入**时间预算。
- 同一话题同时服务自主搜索（search）：搜索任务会发布 `SEARCHING:FREE:n/N`、`NAVIGATING`/`NAVIGATED`/`NAV_FAILED` 等额外状态，本页只列巡逻相关。

**终态统计字段**：终态消息附加清单进度与走访统计，实测示例：

```
EXHAUSTED 8/8 t:287/510 ok:6 fail:0 cancel:3
```

| 字段 | 含义 |
|------|------|
| `n/N` | 清单进度：已办结 n 个 / 本遍清单共 N 个。`EXHAUSTED` 时恒 n==N（收尾把未发出点补记办结）；`TIMEOUT` / `TRAPPED` / `CANCELLED` 时 n<N 即未完成量。建池失败的空池终态不带 n/N |
| `ok` | 到访成功的导航次数（含近点到访） |
| `fail` | 导航真失败次数（含被障碍围死按失败记账的取消） |
| `cancel` | 取消次数——含恢复策略（脱困旋转重发、子图回访抢占等）取消后重发的目标，因此**可能大于清单规模** |
| `notsent` | 已选中但未发出（导航服务忙）的点数；仅终态且非零时出现 |
| `empty_reason` | **仅启动建池为 0 时附加**：空池阶段原因；正常巡完没有此字段 |

口径规则：心跳（`PATROL`）与终态字段同序，但 ok/fail/cancel 心跳计**当前遍**、终态计**整个任务**（跨遍累计）。n/N 与 ok/fail/cancel 分工——n/N 按**地点**去重只计最终办结（同点重试只进 1 次 n），ok/fail/cancel 按**导航次数**不去重（同点重试各进 1 次）；成功率看 `ok/(ok+fail+cancel)`，剩余量看 `N−n`。

启动空池示例：

```text
EXHAUSTED ok:0 fail:0 cancel:0 empty_reason:no_cand
```

| `empty_reason` | 含义与排查方向 |
|------|------|
| `map_too_small` | costmap 包围盒面积低于 `free_pool_map_area_thr`；先完成建图或检查地图范围 |
| `costmap_unavailable` | costmap 不可用；检查 Nav2 costmap 是否已发布 |
| `no_grids` | 没有可采样的机器人可达自由格；检查自由空间、机器人位姿与连通性 |
| `no_cand` | 候选点在路径验证前被面积、关键帧、区域半径或探索范围等条件全部过滤；结合建池日志确认具体过滤项 |
| `no_path` | 候选点均进入 Nav2 路径验证，但全部规划失败；检查起点/目标区域连通性及 Nav2 日志 |

任务状态与成败怎么判（状态话题前缀规则 + 终态成败表 + `get_search_result` 收尾）见 1.3。

### 1.3 如何判断任务状态

**怎么做**——发起后三步：

```bash
# ① 随时单次读取状态（无需维护订阅，latch 保证必有返回）
ros2 topic echo /explore/search_status --once

# ② 按消息前缀判定（三选一，见下表）

# ③ 结束后按需取结构化结果（可选）
ros2 service call /explore/get_search_result tros_frontier_exploration/srv/SearchResult
```

| 消息前缀 | 判定 | 动作 |
|------|------|------|
| `PATROL:` / `WAITING_` / `SPINNING:` | 进行中 | 继续等（`t:elapsed/budget` 看进度） |
| `EXHAUSTED` / `TIMEOUT` / `TRAPPED` / `CANCELLED:` | 已结束 | 按下表判成败/取消 |
| 其他（`SEARCHING:` / `NAVIGATING*` / `IDLE`） | 巡逻被其他任务取代 | 视为本次巡逻异常终止，排查调用方 |

终态 → 成败/取消：

| 终态 | 判定 |
|------|------|
| `EXHAUSTED` 且带 `empty_reason` | ⚠️ 未执行走访（启动建池为 0，按原因字段排查） |
| `EXHAUSTED` 且无 `empty_reason` | ✅ **成功**（清单全部办结，n==N 恒成立） |
| `TIMEOUT` | ⚠️ 未完成（预算耗尽，可放大 `timeout` 重试；`TIMEOUT:main_loop` = 等主循环让权超时，非预算问题） |
| `TRAPPED` | ❌ **失败**（被困且脱困无效） |
| `CANCELLED` | ⏹️ **取消**（有人调了 `stop_patrol`，主动停止无成败语义） |

patrol 没有名为 SUCCESS 的状态——正常成功判据是 `EXHAUSTED` 且没有 `empty_reason`（自然巡完时 n==N 恒成立）；发起响应只代表任务被接受，不代表成败。

状态判读规则：

1. **只认 `header.stamp` 晚于本次发起时刻的消息**（防上一任务残留的旧消息），否则丢弃重读；
2. 发起后超过预算仍未终态时，先确认是否仍在执行末个导航目标或到点旋转；**超过预算 + 120 秒只能作为排查提示，不能单独据此判定节点异常**。结合 `/explore/search_status`、当前导航结果与 explore 日志排查，不要无限等待。

> 备注：`get_search_result` 只反映**最近一次**任务且响应无时间戳——判断"结果是不是本次巡逻的"用状态串的 `header.stamp`；清单进度 `n/N` 也只在状态串里（service 响应无此字段）；怀疑卡死时连调两次，`reason=running` 且 `elapsed_time` 在增长即任务活着。

### 1.4 时间预算（timeout）

`timeout` 是**整次巡逻的时间预算**，不是单个巡逻点的导航超时，也不是到点强制停止的定时器。

| 请求 `timeout` 取值 | 行为 |
|------|------|
| `> 0` | 使用指定值（秒） |
| `<= 0` | 按候选点数量自动计算，最少 180 秒、最多 900 秒 |

**超过时间限制后如何结束**

常规走访时，到达 `timeout` 不会立即中断巡逻，而是按以下顺序收尾：

1. 若正在前往某个巡逻点，继续当前导航，不主动取消；
2. 当前点处理完成后，若导航成功且启用默认到点旋转，先完成 180° 旋转；
3. 随后发布 `TIMEOUT`，结束巡逻，不再前往下一个巡逻点。

因此，实际结束时刻可能晚于 `timeout`，状态中的已耗时 `m` 也可能大于时间预算 `M`，超出幅度取决于当前点剩余导航与旋转时间。若期间手动停止或机器人被困，则以对应终态结束，不一定发布 `TIMEOUT`。

子图回访与旋转的耗时也计入预算。`TIMEOUT` 不会撤销已完成的走访结果，统计随终态发布。

`disable_arrival_spin=true` 只能省去到点旋转，不会改变“超时不取消当前导航”的行为。需要立即停止时，调用 [`/explore/stop_patrol`](#22-explorestop_patrol)：取消当前导航并原地停车，终态为 `CANCELLED:stop_requested`。

**循环巡逻**

`timeout` 限制所有遍数的总时间，不是每一遍单独计时。显式 `timeout > 0` 时，预算不随遍数放大；自动预算会随遍数增加。`patrol_loop_count < 0`（不限次）且使用自动预算时不设时间上限，一直巡逻到调用 `stop_patrol`。

**自动预算的计算方式**

```text
clamp(30 + 60 × N, 180, 900) 秒
```

`N` 为候选点数量；基础预算为 30 秒，每个候选点增加 60 秒。`clamp` 表示将计算结果限制在 180～900 秒之间：

| 候选点数量 N | 自动预算 |
|------|------|
| N ≤ 2 | 180 秒 |
| 3 ≤ N ≤ 14 | `30 + 60 × N` 秒 |
| N ≥ 15 | 900 秒 |

例如，8 个候选点的自动预算为 `30 + 60 × 8 = 510` 秒。上述 30/60/180/900 为代码内固定值，不随 `params.yaml` 配置；`N` 的上限由 `free_pool_max_goals` 控制（默认 20）。

### 1.5 与其他任务的互斥

- **与重定位互斥**：重定位中发起巡逻返回 `success=false`，响应消息为 `relocating in progress`。请求不会排队或启动巡逻，也不会改变现有任务状态；调用方需在重定位结束后重新发起。
- **与原地旋转（spin）互斥**：spin 运行中发起巡逻被拒绝（响应消息 `spin running, wait spin_status DONE`）。
- **与点导航互斥**：已有任务运行时发起巡逻返回**幂等成功**（`already running`），当前任务继续执行——需要按新参数巡逻时，先 `stop_patrol` 停止当前任务。

## 2. 接口参考

巡逻对外共 3 个 service（发起/停止/查询）+ 1 个状态话题（`/explore/search_status`，见 1.2）+ marker 可视化（见 2.4）。

### 2.1 /explore/start_patrol

发起一次自主巡逻：按请求参数圈定候选范围（全图或区域）生成候选池，逐点走访，直至走访完成、预算耗尽或被停止。默认单遍；`patrol_loop_count` 可按遍数循环续巡或不限次循环，`patrol_return_to_start` 可在巡完后返回发起起点。

类型 `tros_frontier_exploration/srv/SearchGoal`：

| 字段 | 类型 | 说明 | 应用场景 |
|------|------|------|---------|
| `target_pose` | PoseStamped | 巡逻中心（map 坐标系）。全图巡逻（radius=0）时该坐标不参与选点（按机器人实时位姿就近取点），可填 0,0；区域巡逻时 `pose.position` 为圆心 | 全图巡逻：填 0,0 即可（[全屋移动监控]）；区域巡逻：填重点区域中心坐标（如老人房、宠物活动区）（[重点区域看护]） |
| `search_radius` | float64 | `0`=全图巡逻；`>0`=区域巡逻（候选裁剪到中心半径圆盘内，按池生成序走访）；`<0` 拒绝，响应消息 `patrol requires search_radius >= 0 (0=whole map, >0=area)` | `0` → [全屋移动监控]（全区域走访，消除监控死角）；`>0` → [重点区域看护]（只走访重点区域，单轮短、可重复发起）；[定时/离家值守] 两种取值均可，由上层按计划选择 |
| `timeout` | float64 | 时间预算（秒），软超时语义及生效时机见 1.4 | `0` 自适应适合常规单轮；[定时/离家值守] 可显式设定会话预算，但它不会准点中断当前目标；需要按计划立即停止时另行调用 `stop_patrol`（见 2.2）。预算小于池规模所需时显式放大（见 5 排查） |
| `nav_purpose` | string | 巡逻不使用（自主搜索定向导航专用），留空 | 无 |
| `patrol_loop_count` | int64 | 循环遍数（仅 patrol 生效）：`0`=单遍（默认，既有行为）；`N>0`=总共巡 N 遍，每遍池尽后重采样候选池并清空灰名单续巡；`<0`=不限次一直循环到 `stop_patrol`。`timeout` 为整个循环会话的总预算（见 1.4）；空池早退不循环 | [重点区域看护] 多轮覆盖；值守式持续巡视传负值，由上层按需停止 |
| `disable_arrival_spin` | bool | `true`=到点后不原地旋转 180° 扫盲区（缺省 `false`=旋转，同时失去该点 yaw 变化的到点存帧触发） | 走访点朝向已覆盖关注区域、需缩短单点耗时时关闭 |
| `patrol_skip_cancelled_retry` | bool | `true`=不补巡被 cancel 的候选点（缺省 `false`=每遍主池尽后立刻对 cancel 点精确重导航；死点由灰名单跨遍累计 3 犯拉黑止损） | 被障碍围死的点较多、补巡明显拖长总时长时开启 |
| `patrol_return_to_start` | bool | `true`=`EXHAUSTED` 终态前 best-effort 回到发起起点（完整位姿含朝向）；仅自然巡完触发（`TIMEOUT`/停止/`TRAPPED`/空池早退不回）；回程失败只告警不改变终态 | 巡逻后需回到出发位（充电桩附近、值守点） |

响应：`success` + `message`（`search started`=已启动；`already running (idempotent, ...)`=幂等成功；重定位中返回 `success=false` 和 `relocating in progress`，不会启动巡逻，见 1.5）。

> 场景名称与主手册 [7.7 自主巡逻](../index.html#77-自主巡逻) 场景能力表对应：[全屋移动监控] / [重点区域看护] / [定时/离家值守]。

### 2.2 /explore/stop_patrol

停止当前运行中的巡逻：机器人取消当前导航目标、原地停车。

类型 `std_srvs/srv/SetBool`，`data` 字段无语义（取值任意）。未在巡逻时返回 `success=true`（message `not running`）。

### 2.3 /explore/get_search_result

查询巡逻任务的结果与进度：**可随时重复调用**——任务进行中返回实时中间值（`reason=running` + 当前进度），终态后返回冻结的最终统计；每次调用返回调用时刻的快照，非一次性结果，也不存历史（只反映最近一次任务）。

类型 `tros_frontier_exploration/srv/SearchResult`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `success` | bool | 恒为 true（无判断语义，任务状态看 `reason`） |
| `reason` | string | 任务当前状态：`running`（进行中）/ `exhausted` / `timeout` / `cancelled` 等 |
| `goals_completed` | int32 | 已处理目标数（进行中为实时值） |
| `elapsed_time` | float64 | 任务用时（秒；进行中为实时值，终态后为定值） |

**与状态话题 `/explore/search_status` 的区别与选择**：

| | `/explore/search_status`（话题） | `/explore/get_search_result`（服务） |
|--|--|--|
| 模型 | 推：状态变化/进度心跳时发布，订阅持续接收 | 拉：调用时返回一次快照 |
| 信息量 | 全：状态机各态、时间进度、终态完整分项统计 | 摘要：4 个结构化字段（状态原因/完成数/用时） |
| 数据形式 | 字符串（程序消费需解析） | 结构化字段（程序直接取用） |
| 适用 | 持续监控、调试、可视化 | 上层应用低频查询结果、不便保持订阅的场景 |

两者可并用：上层监听 `search_status` 终态驱动流程，需要结构化数据时再调 `get_search_result`。注意 `get_search_result` 反映的是**最近一次**任务（新任务启动即覆盖旧值），非历史记录。判定流程见 1.3「如何判断任务状态」。

### 2.4 可视化

巡逻候选点与走访结果的 Foxglove marker 通道（颜色/形状/话题）见主手册 [发起和停止巡逻](../index.html#发起和停止巡逻)。

## 3. 参数参考

参数位于 `params.yaml` 的 explore 段，修改后重启 solution 生效（同主手册 4.6）：

| 参数 | 默认值 | 单位/口径 | 说明 |
|------|-------|----------|------|
| `free_pool_map_area_thr` | 20.0 | m²（包围盒估算） | 建池门槛：地图面积低于该值不生成候选池，发起巡逻立即 `EXHAUSTED`，附 `empty_reason:map_too_small` |
| `free_pool_cell_size` | 3.0 | m | 采样方格边长（控制巡逻点密度）：每方格取最大 free 连通域质心作为候选 |
| `free_pool_max_goals` | 20 | 个 | 候选池上限：超限按最远点采样（FPS，保证截断后均匀覆盖全图）截取，直接限制单轮巡逻点数与时长 |
| `free_pool_min_area_ratio` | 2.0 | 系数 | 候选盘最小连通 free 面积 = ratio × π × robot_radius²，过滤碎小 free 区域 |
| `free_pool_kf_radius` | 1.0 | m（经验值待校准） | 关键帧约束半径：候选点该半径内关键帧数（mapGraph 节点位姿 = 机器人真实驶过观察过的位置）≥ `free_pool_kf_min_count` 才合格——防止 costmap 投影外推产生的"从未驶过区域"假 free 点；`<=0` 禁用 |
| `free_pool_kf_min_count` | 1 | 个 | 关键帧约束的最少关键帧数 |

除参数外，候选池生成还会做 **Nav2 路径验证**（从机器人当前位置规划不到的点被剔除，仅以灰色 marker 可视化）。

## 4. 进阶用法

- **多轮巡逻**：默认单遍；需要连续多轮时优先用 `patrol_loop_count`（N 遍或不限次循环，每遍重采样候选池），或仍由上层在终态后重新发起。
- **定点返回**：`patrol_return_to_start=true` 在自然巡完（`EXHAUSTED`）前 best-effort 回到发起起点，适合巡逻后需回到出发位（充电桩、值守点）的场景；超时/手动停止/被困不触发回起点。
- **重点区域看护**：区域巡逻（小半径）池小、单轮快，适合上层按计划高频重复发起。
- **上层集成**：巡逻通过 service 发起/停止、进度经话题发布——定时任务、语音助手等上层应用可按计划调用 `start_patrol` 并监听 `search_status` 终态。巡逻的产出是"走访"行为与状态统计；巡视中的异常发现可结合 solution 已有感知话题，告警推送等由上层应用实现。

## 5. 问题排查

| 现象 | 可能原因 | 处置 |
|------|---------|------|
| 发起立即 `EXHAUSTED` | 启动建池为 0，终态中的 `empty_reason` 标明空池阶段（见 1.2） | 按原因检查地图、自由空间、区域半径或 Nav2 路径；`no_cand` 需结合建池日志确认具体过滤项 |
| `PATROL:n/N` 长时间不推进 | 处于 `WAITING_SUBMAP_FLOW`（子图回访接管）或 `SPINNING:*`；或取消点在等补巡（未办结不计 n，属正常） | 属正常等待，观察状态串即可；长时间卡住再抓 explore 节点日志 |
| 终态 `TIMEOUT` 且 `n < N` | 预算小于清单规模所需（或等待占用过多预算） | 显式传大 `timeout`，或调小 `free_pool_max_goals` |
| 超过 `timeout` 后仍继续前往当前点 | 软超时不取消当前导航；默认到点旋转也会延后终态，见 1.4 | 等当前点结果处理后确认 `TIMEOUT`；需要立即停止时调用 `stop_patrol`，不要只依赖 `timeout` |
| 响应 `success=true` 但机器人不动 | 幂等成功——已有任务在运行 | 先 `stop_patrol`，再按新参数发起 |
| 候选点明显偏少 | 关键帧约束过滤（`free_pool_kf_radius`）或 Nav2 路径验证剔除 | 新建图区域先让机器人驶过；必要时临时调小/禁用 `free_pool_kf_radius` 观察 |
