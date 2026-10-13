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
| `WAITING_SUBMAP_FLOW: waited=Ns` | 子图回访/建图（submapLinkFlow）接管控制期间巡逻挂起，每 5s 心跳；flow 结束后自动重选目标继续 |
| `EXHAUSTED` | 单轮走访完成（全部候选点处理完）；地图过小或无自由点时发起后立即以此终态结束 |
| `TIMEOUT` | 时间预算耗尽 |
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

### 1.3 时间预算（timeout）

| 请求 `timeout` 取值 | 行为 |
|------|------|
| `> 0` | 按指定值执行 |
| `<= 0` | 按候选池规模自适应：`clamp(30 + 60 × N, 180, 900)` 秒（N 为池内候选点数） |

预算耗尽发布终态 `TIMEOUT`，已走访部分不回滚（统计随终态发布）。

### 1.4 与其他任务的互斥

- **与原地旋转（spin）互斥**：spin 运行中发起巡逻被拒绝（响应消息 `spin running, wait spin_status DONE`）。
- **与自主搜索（search）、点导航互斥**：已有任务运行时发起巡逻返回**幂等成功**（`already running`），当前任务继续执行——需要按新参数巡逻时，先 `stop_patrol` 停止当前任务。
- `stop_patrol` 与 `stop_search` 共用同一取消通道，停止的都是"当前运行中的任务"。

## 2. 接口参考

### 2.1 /explore/start_patrol

类型 `tros_frontier_exploration/srv/SearchGoal`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `target_pose` | PoseStamped | 巡逻中心（map 坐标系）。全图巡逻（radius=0）时该坐标不参与选点（按机器人实时位姿就近取点），可填 0,0；区域巡逻时 `pose.position` 为圆心 |
| `search_radius` | float64 | `0`=全图巡逻；`>0`=区域巡逻（候选裁剪到中心半径圆盘内，按池生成序走访）；`<0` 拒绝，响应消息 `patrol requires search_radius >= 0 (0=whole map, >0=area)` |
| `timeout` | float64 | 时间预算（秒），见 1.3 |
| `nav_purpose` | string | 巡逻不使用（自主搜索定向导航专用），留空 |

响应：`success` + `message`（`search started`=已启动；`already running (idempotent, ...)`=幂等成功）。

### 2.2 /explore/stop_patrol

类型 `std_srvs/srv/SetBool`，`data` 字段无语义（取值任意）。停止后机器人取消当前导航目标、原地停止；未在巡逻时返回 `success=true`（message `not running`）。

### 2.3 /explore/get_search_result

类型 `tros_frontier_exploration/srv/SearchResult`：

| 字段 | 类型 | 说明 |
|------|------|------|
| `success` | bool | 已有/曾有结果即为 true |
| `reason` | string | 任务状态：`running`（进行中）/ `exhausted` / `timeout` 等 |
| `goals_completed` | int32 | 已处理目标数 |
| `elapsed_time` | float64 | 任务用时（秒） |

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
