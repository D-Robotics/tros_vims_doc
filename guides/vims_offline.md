---
title: 离线数据录制与回放（ViMS_offline）深度指南
layout: default
---

# 离线数据录制与回放（ViMS_offline）深度指南

> 快速上手见主手册 [11.1 离线数据录制与回放](../index.html#111-离线数据录制与回放vims_offline)。本页覆盖环境准备、子命令与参数、回放能力profile、录制数据质量检查、单参数调试覆盖以及回放的实现细节，是该工具的完整参考。

## 1. 环境准备

工具已随移动Solution一起编译安装（按[软件配置](../index.html#4-软件配置)章节安装后即包含），无需单独获取和编译。环境要求 python3 + rclpy + PyYAML + ROS 2 CLI（`ros2`）。在机器人端使用：

```bash
source /opt/ros/humble/setup.bash
source /opt/tros/humble/local_setup.bash
source /userdata/vims/install/local_setup.bash
```

source 环境后即可使用安装好的`vims_offline`命令（与`ros2 run vims_offline vims_offline ...`等价）。

> 标定是前置条件：录制依赖机器人已完成的标定（相机内参/外参，`camera_link→imu_link` 等硬件绑定TF来自标定结果）。未标定或标定过期的机器人先参考[外参标定](../index.html#6-外参标定)章节完成标定。

## 2. 子命令与参数

所有功能通过 `vims_offline <子命令>` 调用，共3个子命令：

|子命令|功能|
| :---: | --- |
|`record`|录制数据会话：最小传感器源topic集（rosbag2/mcap）+ 捕获静态TF|
|`replay`|回放录制会话：按profile选定模块组合拉起系统，含空闲检测与报告|
|`list-profiles`|列出可用的回放profile|

**record 参数：**

|参数|默认值|说明|
| :---: | :---: | --- |
|`-o, --output-dir`|当前目录|录制会话的根目录|
|`-n, --session-name`|`session_<时间戳>`|会话名。会话目录名始终拼上录制时间戳（录制开始时刻，格式`YYYYmmdd_HHMMSS`）：指定`-n my_session` → `my_session_<时间戳>`；不指定 → `session_<时间戳>`。同秒重录同名会因目录已存在报错|
|`--topics-config FILE`|`config/record_topics_jpeg.yaml`|录制topic集合（子码流jpeg编码）|
|`--record-switches FILE`|`config/record_modules.yaml`|录制时自动拉起系统的模块开关|
|`--tf-config FILE`|`config/static_tf.yaml`|要捕获的静态TF列表|
|`--max-cache-size MB`|100|录制缓存大小。出现"Message dropped"告警（缓存满丢消息）时调大，500已验证无丢帧，代价是内存|
|`--base-params FILE`|tros_vision_nav安装目录params.yaml|参数基准文件。默认用tros_vision_nav安装目录下的params.yaml；换机器人/标定件后参数不同时，可通过该参数指定自定义params.yaml（如修改了标定参数的副本），原文件不被修改|
|`--launch-arg NAME:=VALUE`|无|单参数调试覆盖，以环境变量方式传给run_launch.sh（优先于配置），可重复传多个|

**replay 参数：**

|参数|默认值|说明|
| :---: | :---: | --- |
|`session_dir`（位置参数）|必填|录制会话目录（需含`bag/` + `tf_static.yaml`），永远不会被修改|
|`--profile`|`full`|回放profile，见[回放能力profile](#6-回放能力profile)|
|`--rate`|1.0|回放倍率（0.5即半速）|
|`--run-dir`|当前目录|回放运行根目录，每次回放生成`<session>_<profile>_<时间戳>/`（含参数副本与回放报告）|
|`--base-params FILE`|tros_vision_nav安装目录params.yaml|参数基准文件，同record：默认用tros_vision_nav安装目录下的params.yaml，可指定自定义params.yaml（如修改了标定参数的副本）；建议与录制时用同一份|
|`--launch-arg NAME:=VALUE`|无|最高优先级调试覆盖（压过profile开关与参数文件），可重复传多个；NAME用launch扁平参数名，不能带点号|
|`--modules-config FILE`|`config/replay_modules.yaml`|回放模块profile配置，可自行编辑或新增profile|

**list-profiles 参数：**

|参数|默认值|说明|
| :---: | :---: | --- |
|`--modules-config FILE`|`config/replay_modules.yaml`|profile配置文件|

另外，录制会话目录永远不会被修改；每次回放写自己的`<session>_<profile>_<时间戳>/`目录，内含`params_offline.yaml`（本次运行的完整参数副本）和`replay_report.yaml`（状态、耗时、开关、空闲检测）。

后文回放/质量检查示例中的`my_session_<时间戳>`请替换为录制结束时打印的实际会话目录名。

> 注意：录制中一旦出现"Message dropped"告警，说明缓存已满、正在丢消息！立即停止录制，加`--max-cache-size 500`重录。丢过帧的bag回放时会有时间空洞，可用[录制数据质量检查](#3-录制数据质量检查)确认。

## 3. 录制数据质量检查

录制完成后用`tros_bag interval`统计各topic的消息间隔（帧率是否稳定、有没有丢帧）：

```bash
ros2 tros_bag interval /userdata/recordings/my_session_<时间戳>/bag
```

输出每个topic的消息数与相邻消息间隔的min/mean/max，并在bag目录旁生成`bag_interval_plots/interval_scatter.png`散点图。

判读基准（正常值）：

|topic|期望间隔|
| :---: | :---: |
|`/imu_data`|mean ≈ 5ms（200Hz）|
|双目子流（sub_image_combine_*）|≈ 125ms（8Hz）|
|主图流（image_combine_*）|≈ 167ms（6Hz）|

丢帧特征：mean接近期望的两倍（如子流mean 250ms = 每隔一帧丢一帧），或min/max异常离散。发现丢帧时按[子命令与参数](#2-子命令与参数)中`--max-cache-size`的说明调大缓存后重录。

## 4. 回放如何保证单一数据源与统一时基

以下强制离线开关写入每次回放的参数副本（代码级固定，不可配置）：

- `robot.robot_base=none`——不加载底盘：回放绝不驱动机器人
- `switch.run_mipi_cam=false`——真实相机保持关闭；bag Player是唯一的图像/IMU发布者
- `switch.run_static_tf=false` + ViMS_offline自己的static_transform_publisher进程回放录制的TF——值与tf_static.yaml一致
- `switch.run_bag_player=true`——rosbag2 Player以组件形式加载进tros_container，与VIO同进程（intra-process零拷贝）
- `switch.use_sim_time=true` + `vio.use_sim_time=true`——Player的50Hz `/clock`（bag时间）驱动整个系统。墙钟/bag时间混用会让VIO的陈旧IMU守卫静默丢弃所有bag消息、rtabmap的TF buffer拒绝bag时间戳的查询
- `vio.sub_from_compressed_image=true`——JPEG bag没有原始双目topic，VIO内部解码jpeg流
- `rtabmap.rtabmap_args=--delete_db_on_start`——每次回放只从bag建图，绝不续建机器人上的live库

另外，`run_slam=true`时bag Player不随容器组一起加载：bringup先等`tros_tf_listener`（map→odom TF，rtabmap初始化即发布），其退出后再加载Player（20秒超时防死锁兜底）。没有这道门控，bag会在组件链还在加载时开播，rtabmap因TF查询早于自身buffer而拒绝每次更新。

## 5. 已知限制与配置文件

已知限制：rtabmap/full回放开头约3秒可能出现TF extrapolation告警（VIO第一条odom→base_link TF落后于bag首帧），属瞬态，不影响建图结果。

所有可调项都是配置文件，随包安装在`$(ros2 pkg prefix vims_offline --share)/config/`下（无需改代码，改配置即可）：

- `record_topics_jpeg.yaml`——录制topic集合（子码流jpeg编码）
- `record_modules.yaml`——录制自动拉起系统的模块开关
- `static_tf.yaml`——要捕获的静态TF列表
- `replay_modules.yaml`——回放模块profile

## 6. 回放能力profile

查看可用回放 profile：

```bash
vims_offline list-profiles
```

回放能力profile（`config/replay_modules.yaml`，可直接编辑或新增）：

|profile|启动模块|已验证输出|
| :---: | --- | --- |
|`depth`|双目 + stereonet|`/StereoNetNode/stereonet_depth`（mono16，约6fps）|
|`vio`|+ VIO|`/odom`（约40Hz）、odom→base_link TF|
|`rtabmap`|+ 分割/深度过滤 + rtabmap|`/map`占据栅格（每次全新建库）|
|`full`|+ 导航/探索|以上全部 + nav2容器|

注意：需要图像的profile里`run_stereo`必须保持true——stereonet链承载图像管线；rtabmap profile要求`run_mask_depth`和`run_perc`同时打开（深度过滤节点把深度图与AI分割结果做精确时间同步，缺一个rtabmap就断粮）。

## 7. 单参数调试覆盖（--launch-arg）

不改任何params文件、临时覆盖个别参数的最高优先级通道，写成`--launch-arg NAME:=VALUE`（或`NAME=VALUE`），可重复传多个。优先级：`--launch-arg` > profile/录制开关 > params.yaml。例如临时开VIO详细日志：

```bash
vims_offline replay /userdata/recordings/my_session_<时间戳> \
    --profile vio --launch-arg vio_log_level:=info
```

要点：NAME用launch扁平参数名（如`vio_log_level`、`rtabmap_args`），不是YAML点路径（带点的键会被直接报错拒绝）；NAME必须是run_launch.sh认识的参数名，否则启动时会提示该覆盖无效果；`YAML_CONFIG_FILE`等受管变量无法覆盖；空值无法表达；布尔值用大写`True`/`False`（与params.yaml拼写一致，launch链按此大小写敏感匹配）。
