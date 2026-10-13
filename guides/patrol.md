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
3. **走访循环**：全图巡逻按"距机器人实时位姿就近"选点，区域巡逻按池生成序；每个候选点导航前往，到访/失败/取消后处理下一个。
4. **终态**：见 1.2 状态表。

### 1.2 状态话题 /explore/search_status

消息类型 `tros_frontier_exploration/msg/StringStamped`，QoS 为 transient_local——**新订阅者立即收到最新一条状态**，无需等待下次发布。

| 状态 | 含义 |
|------|------|
| `IDLE` | 空闲（无任务） |
| `PATROL:n/N` | 巡逻进行中：已处理 n 个 / 共 N 个候选点 |
| `SPINNING:scan_blindspot[:n/N]` | 原地旋转扫描盲区（候选点间盲区补扫），旋转结束自动恢复走访；变体 `SPINNING:escape_obstacle`（脱困旋转）、`SPINNING:find_frontier` |
| `WAITING_MAIN_LOOP: waited=Ns` | 主循环 frontier 探索占用导航控制权，巡逻挂起等待（每 5s 心跳）；等待超 60 秒直接终态 `TIMEOUT:main_loop` |
| `WAITING_SUBMAP_FLOW: waited=Ns` | 子图回访/建图（submapLinkFlow）接管控制期间巡逻挂起，每 5s 心跳；flow 结束后自动重选目标继续 |
| `EXHAUSTED` | 单轮走访完成（全部候选点处理完）；地图过小或无自由点时发起后立即以此终态结束 |
| `TIMEOUT` | 时间预算耗尽（变体 `TIMEOUT:main_loop`：等主循环让控制权超 60 秒，非巡逻预算问题） |
| `CANCELLED:agent_stop` | 调用 `stop_patrol`（或 `stop_search`）停止 |
| `TRAPPED` | 机器人被困，恢复策略无效 |

附加规则：

- 进行中与终态消息附带时间进度 `t:elapsed/budget`（已用秒 / 预算秒）。`elapsed` 略超 `budget` 属正常现象（末个目标跨越预算边界时），可作为诊断信号。
- `WAITING_SUBMAP_FLOW` 与 `SPINNING:*` 的等待时间**计入**时间预算。
- 同一话题同时服务自主搜索（search）：搜索任务会发布 `SEARCHING:FREE:n/N`、`NAVIGATING`/`NAVIGATED`/`NAV_FAILED` 等额外状态，本页只列巡逻相关。

**终态统计字段**：终态消息附加走访统计，实测示例：

```
EXHAUSTED t:287/510 ok:6 fail:0 cancel:3 notsent:0 sent:9 pool:8 unvisited:0
```

| 字段 | 含义 |
|------|------|
| `ok` | 到访成功数 |
| `fail` | 导航真失败数 |
| `cancel` | 取消数——含恢复策略（脱困旋转重发、子图回访抢占等）取消后重发的目标，因此**可能大于池规模** |
| `notsent` | 已选中但未发出（导航服务忙） |
| `sent` | 发出且得到结果的目标数，恒等式 `ok + fail + cancel == sent` |
| `pool` | 候选池总数 |
| `unvisited` | 未走访数（= 池内仍合格的候选点） |

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
| `EXHAUSTED` 且 `unvisited:0` | ✅ **成功**（候选点全部走访完成） |
| `EXHAUSTED` 且 `unvisited>0` | ⚠️ 部分完成（候选中途被剔除，按业务容忍度判） |
| `TIMEOUT` | ⚠️ 未完成（预算耗尽，可放大 `timeout` 重试；`TIMEOUT:main_loop` = 等主循环让权超时，非预算问题） |
| `TRAPPED` | ❌ **失败**（被困且脱困无效） |
| `CANCELLED` | ⏹️ **取消**（有人调了 `stop_patrol`，主动停止无成败语义） |

patrol 没有名为 SUCCESS 的状态——成功就是上表判定式（`EXHAUSTED` 且 `unvisited:0`）；发起响应只代表任务被接受，不代表成败。

两条硬规则：

1. **只认 `header.stamp` 晚于本次发起时刻的消息**（防上一任务残留的旧消息），否则丢弃重读；
2. 发起后**预算 + 120 秒**仍无终态 → 判节点异常，抓 explore 日志排查，不要无限等。

> 备注：`get_search_result` 只反映**最近一次**任务且响应无时间戳——判断"结果是不是本次巡逻的"用状态串的 `header.stamp`；成功判据 `unvisited` 也只在状态串里（service 响应无此字段）；怀疑卡死时连调两次，`reason=running` 且 `elapsed_time` 在增长即任务活着。

### 1.4 时间预算（timeout）

| 请求 `timeout` 取值 | 行为 |
|------|------|
| `> 0` | 按指定值执行 |
| `<= 0` | 按候选池规模自适应：`clamp(30 + 60 × N, 180, 900)` 秒（N 为池内候选点数） |

**公式拆解**——`clamp(x, lo, hi)` 表示把 x 限制在 [lo, hi] 区间内：x 低于下限取下限、高于上限取上限、区间内取原值。公式由四部分构成：

| 项 | 值 | 含义 |
|------|------|------|
| `30` | 基础预算 | 与候选点数无关的固定开销：建池、起点定位、任务收尾等 |
| `60 × N` | 逐点预算 | 每个候选点分配 60 秒走访预算（导航前往 + 到访确认 + 可能的恢复重试） |
| `180` | 下限 | 预算最少 3 分钟，防止小池被过早掐断 |
| `900` | 上限 | 预算最多 15 分钟，封顶单轮巡逻的最坏时长 |

按 N 的取值分段：

| N（池内候选点数） | 代入 30+60×N | 实际预算 |
|------|------|------|
| N ≤ 2 | 90~150 秒 | 触下限 → **180 秒** |
| 3 ≤ N ≤ 14 | 210~870 秒 | 公式原值生效（线性增长） |
| N ≥ 15 | 930 秒起 | 触上限 → **900 秒** |

实测对照：候选池 8 点、`timeout` 传 0 → 预算 30+60×8=**510 秒**（状态串显示 `t:147/510`），与公式一致。

> **提示** 
> 30/60/180/900 为代码内固定常量，不随 `params.yaml` 配置。N 的上限即 `free_pool_max_goals`（默认 20，对应预算触 900 秒上限）。预算覆盖全程等待——子图回访（`WAITING_SUBMAP_FLOW`）与旋转（`SPINNING:*`）均计入（见 1.2）。需要更长预算时显式传 `timeout`；预算耗尽发布终态 `TIMEOUT`，已走访部分不回滚（统计随终态发布）。
>

### 1.5 与其他任务的互斥

- **与原地旋转（spin）互斥**：spin 运行中发起巡逻被拒绝（响应消息 `spin running, wait spin_status DONE`）。
- **与点导航互斥**：已有任务运行时发起巡逻返回**幂等成功**（`already running`），当前任务继续执行——需要按新参数巡逻时，先 `stop_patrol` 停止当前任务。

## 2. 接口参考

巡逻对外共 3 个 service（发起/停止/查询）+ 1 个状态话题（`/explore/search_status`，见 1.2）+ marker 可视化（见 2.4）。

### 2.1 /explore/start_patrol

发起一轮自主巡逻：按请求参数圈定候选范围（全图或区域）生成候选池，逐点走访，直至走访完成、预算耗尽或被停止。

类型 `tros_frontier_exploration/srv/SearchGoal`：

| 字段 | 类型 | 说明 | 应用场景 |
|------|------|------|---------|
| `target_pose` | PoseStamped | 巡逻中心（map 坐标系）。全图巡逻（radius=0）时该坐标不参与选点（按机器人实时位姿就近取点），可填 0,0；区域巡逻时 `pose.position` 为圆心 | 全图巡逻：填 0,0 即可（[全屋移动监控]）；区域巡逻：填重点区域中心坐标（如老人房、宠物活动区）（[重点区域看护]） |
| `search_radius` | float64 | `0`=全图巡逻；`>0`=区域巡逻（候选裁剪到中心半径圆盘内，按池生成序走访）；`<0` 拒绝，响应消息 `patrol requires search_radius >= 0 (0=whole map, >0=area)` | `0` → [全屋移动监控]（全区域走访，消除监控死角）；`>0` → [重点区域看护]（只走访重点区域，单轮短、可重复发起）；[定时/离家值守] 两种取值均可，由上层按计划选择 |
| `timeout` | float64 | 时间预算（秒），见 1.4 | `0` 自适应适合常规单轮；[定时/离家值守] 中上层按计划控制单轮时长时显式设定；预算小于池规模所需时显式放大（见 5 排查） |
| `nav_purpose` | string | 巡逻不使用（自主搜索定向导航专用），留空 | 无 |

响应：`success` + `message`（`search started`=已启动；`already running (idempotent, ...)`=幂等成功）。

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

巡逻候选点与走访结果的 Foxglove marker 通道（颜色/形状/话题）见主手册 [Foxglove效果展示](../index.html#foxglove效果展示)。

## 3. 参数参考

参数位于 `params.yaml` 的 explore 段，修改后重启 solution 生效（同主手册 4.6）：

| 参数 | 默认值 | 单位/口径 | 说明 |
|------|-------|----------|------|
| `free_pool_map_area_thr` | 20.0 | m²（包围盒估算） | 建池门槛：地图面积低于该值不生成候选池，发起巡逻立即 `EXHAUSTED` |
| `free_pool_cell_size` | 3.0 | m | 采样方格边长（控制巡逻点密度）：每方格取最大 free 连通域质心作为候选 |
| `free_pool_max_goals` | 20 | 个 | 候选池上限：超限按最远点采样（FPS，保证截断后均匀覆盖全图）截取，直接限制单轮巡逻点数与时长 |
| `free_pool_min_area_ratio` | 2.0 | 系数 | 候选盘最小连通 free 面积 = ratio × π × robot_radius²，过滤碎小 free 区域 |
| `free_pool_kf_radius` | 1.0 | m（经验值待校准） | 关键帧约束半径：候选点该半径内关键帧数（mapGraph 节点位姿 = 机器人真实驶过观察过的位置）≥ `free_pool_kf_min_count` 才合格——防止 costmap 投影外推产生的"从未驶过区域"假 free 点；`<=0` 禁用 |
| `free_pool_kf_min_count` | 1 | 个 | 关键帧约束的最少关键帧数 |

除参数外，候选池生成还会做 **Nav2 路径验证**（从机器人当前位置规划不到的点被剔除，仅以灰色 marker 可视化）。

## 4. 进阶用法

- **多轮巡逻**：本接口为单轮语义，需要再巡一轮时在终态后重新发起。
- **重点区域看护**：区域巡逻（小半径）池小、单轮快，适合上层按计划高频重复发起。
- **上层集成**：巡逻通过 service 发起/停止、进度经话题发布——定时任务、语音助手等上层应用可按计划调用 `start_patrol` 并监听 `search_status` 终态。巡逻的产出是"走访"行为与状态统计；巡视中的异常发现可结合 solution 已有感知话题，告警推送等由上层应用实现。

## 5. 问题排查

| 现象 | 可能原因 | 处置 |
|------|---------|------|
| 发起立即 `EXHAUSTED` | 地图面积 < `free_pool_map_area_thr`，或无自由点可访 | 先按主手册 7.3 完成建图；确认地图范围 |
| `PATROL:n/N` 长时间不推进 | 处于 `WAITING_SUBMAP_FLOW`（子图回访接管）或 `SPINNING:*` | 属正常等待，观察状态串即可；长时间卡住再抓 explore 节点日志 |
| 终态 `TIMEOUT` 且 `unvisited > 0` | 预算小于池规模所需（或等待占用过多预算） | 显式传大 `timeout`，或调小 `free_pool_max_goals` |
| 响应 `success=true` 但机器人不动 | 幂等成功——已有任务在运行 | 先 `stop_patrol`，再按新参数发起 |
| 候选点明显偏少 | 关键帧约束过滤（`free_pool_kf_radius`）或 Nav2 路径验证剔除 | 新建图区域先让机器人驶过；必要时临时调小/禁用 `free_pool_kf_radius` 观察 |
