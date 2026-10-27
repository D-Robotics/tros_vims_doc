---
title: CycloneDDS 配置
layout: default
---

# CycloneDDS 配置

> 解决板载 WiFi 多播发现不可靠导致的 pub-sub 匹配失败问题。
>
> **适用场景**：单机器人独立运行（所有 ROS 2 节点在同一块板上）时的板端 DDS 配置。典型场景：建图、导航、巡逻、搜索、VIO、离线录制/回放——凡是节点全部在本机的用法均适用。多机器人协同或外部设备通过 DDS 直连板端话题的场景**不适用**（见"限制"章节）。

## 1. 板端部署

### 1.1 写配置文件

```bash
cat > /userdata/cyclonedds_config_file.xml << 'XMLEOF'
<?xml version="1.0" encoding="UTF-8"?>
<CycloneDDS xmlns="https://cdds.io/config">
  <Domain Id="any">
    <General>
      <Interfaces>
        <NetworkInterface name="lo" priority="default" multicast="false"/>
      </Interfaces>
      <AllowMulticast>false</AllowMulticast>
    </General>
    <Discovery>
      <Peers>
        <Peer address="127.0.0.1"/>
      </Peers>
    </Discovery>
  </Domain>
</CycloneDDS>
XMLEOF
```

### 1.2 设置环境变量

```bash
echo 'CYCLONEDDS_URI=/userdata/cyclonedds_config_file.xml' >> /etc/environment
echo 'export CYCLONEDDS_URI=/userdata/cyclonedds_config_file.xml' >> ~/.bashrc
```

### 1.3 验证

```bash
source /etc/environment
ros2 topic list   # 有输出 = DDS 正常
```

## 2. 各配置项含义

| 配置项 | 作用 |
|--------|------|
| `NetworkInterface name="lo"` | 将 DDS 通信绑定到回环接口——同机进程间数据不走 WiFi |
| `multicast="false"` | 该接口上禁用多播——绕开 WiFi 多播写失败的故障 |
| `AllowMulticast=false` | 禁用所有多播发现——发现仅通过单播 Peer 进行 |
| `Peer address="127.0.0.1"` | 单播发现的对端地址——同机所有节点通过 lo 回环互相发现 |

## 3. 原理说明

CycloneDDS 默认通过 **多播**（239.255.0.1:18900）发送节点发现公告（SPDP 协议），公告的目的是让同网络中的其他节点知道"我存在、我的通信地址是什么"，双方据此建立通信通道。

板载 WiFi 网卡（aic8800）的**多播发送路径存在间歇性故障**——公告写不出去，其他节点不知道该节点的存在，DDS 匹配无法完成。匹配不完成，后续所有 pub-sub 消息投递都不会发生。

本配置将 DDS 通信绑定到 **lo 回环接口**并指定 **单播 peer 127.0.0.1**：

- 全部 ROS 2 节点在同一块板上（单机系统），节点间通信本质上不需要经过物理网络
- lo 回环接口的数据在内核内存中直接传递，永不丢包
- 所有节点通过 lo 上的 127.0.0.1 互相发现和通信，不依赖 WiFi 的多播路径
- 因此多播写失败、WiFi 抖动、链路质量变化均不影响 DDS 发现和数据投递

## 4. 收益

- 发现匹配秒级完成（lo 回环永不丢包）
- 传感器数据在进程间走 lo 回环，更快更稳
- 不受 WiFi 多播不可靠的影响

## 5. 限制

| 限制 | 说明 |
|------|------|
| **仅限同机通信** | 所有 DDS 节点必须在同一块板上。跨机 DDS 通信（多机器人协同、开发机直连板端 DDS）在此配置下不可用 |
| **外部设备无法通过 DDS 访问** | WSL/MCP 等外部工具如果需要通过 DDS（而非 rosbridge WebSocket）订阅板上话题，此配置会阻断 |
| **未来跨机扩展需改配置** | 多机器人协同等场景需在 Discovery/Peers 中添加对方 IP，并将 NetworkInterface 改回物理网卡 |

**禁止使用的场景**：需要跨机 DDS 通信的任何场合（多机编队、外部开发机直接以 DDS 协议订阅板上话题等）。
