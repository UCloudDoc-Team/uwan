# CE 侧 VPN 配置指南

本文档提供 CE 客户网关侧的 VPN 配置示例，以 strongSwan (Linux) 为例，涵盖感兴趣流模式和 BGP 模式两种 IPSec 路由模式。

## 一、配置前提

- 已在 UCloud 控制台创建 UWAN 虚拟路由器，并获取其公网 IP 地址
- 已创建 CE 客户网关，并完成 VPN 隧道配置
- CE 侧设备已安装 strongSwan（推荐 5.9+ 版本）

## 二、IPSec 参数对照表

以下参数需与 UWAN 控制台上 VPN 隧道的配置保持一致。

### IKE 参数

| 参数 | UWAN 默认值 | 可选值 |
|------|------------|--------|
| IKE 版本 | v2 | v1、v2 |
| 加密算法 | aes128 | aes128、aes192、aes256、3des |
| 认证算法 | sha1 | md5、sha1、sha2-256 |
| DH 组 | 15 | 1、2、5、14、15、16 |
| SA 超时（时间） | 1080 秒 | 600-604800 秒 |
| 协商模式 | 主模式 | 主模式、野蛮模式（仅 IKEv1） |

### IPSec 参数

| 参数 | UWAN 默认值 | 可选值 |
|------|------------|--------|
| 加密算法 | aes128 | aes128、aes192、aes256、3des |
| 认证算法 | sha1 | md5、sha1、sha2-256 |
| 安全协议 | ESP | ESP、AH |
| PFS DH 组 | Disable | Disable、1、2、5、14、15、16 |
| SA 超时（时间） | 3600 秒 | 1200-604800 秒 |
| SA 超时（流量） | — | 8000-2000000 字节 |

### BGP 参数（BGP 模式时需配置）

| 参数 | 说明 |
|------|------|
| 本端 ASN | CE 侧 BGP 自治系统号 |
| 对端 ASN | UWAN 侧 BGP 自治系统号（在控制台创建隧道时指定） |
| BGP 隧道网段 | 建立 BGP 邻居所使用的网段，格式为 `169.254.x.x/30`（在控制台创建隧道时指定） |
| 本端 BGP 地址 | 隧道网段中的本端 IP |
| 对端 BGP 地址 | 隧道网段中的对端 IP |

### DPD 参数

| 参数 | 说明 |
|------|------|
| DPD 开关 | 是否启用 Dead Peer Detection |
| DPD 延迟 | 发送 DPD 探测的间隔时间 |
| DPD 超时 | 判定对端不可达的超时时间（仅 IKEv1） |
| DPD 动作 | 检测到对端不可达后的动作：clear（断开）、trap（等待重连）、restart（重启） |

## 三、strongSwan 配置示例

### 感兴趣流模式

以下示例假设：
- UWAN 侧公网 IP：`203.0.113.1`
- CE 侧公网 IP：`198.51.100.1`
- CE 侧内网网段：`172.16.1.0/24`
- UWAN 侧内网网段：`10.0.0.0/16`
- 预共享密钥：`MySecretKey123`

#### /etc/swanctl/swanctl.conf

```
connections {
   uwan {
      local_addrs  = 198.51.100.1
      remote_addrs = 203.0.113.1

      local {
         auth = psk
         id = 198.51.100.1
      }
      remote {
         auth = psk
         id = 203.0.113.1
      }

      children {
         net-net {
            local_ts  = 172.16.1.0/24
            remote_ts = 10.0.0.0/16

            esp_proposals = aes128-sha1-modp3072
            rekey_time    = 3600s
            dpd_action    = clear

            mode = tunnel
            start_action = trap
         }
      }

      version = 2
      proposals = aes128-sha1-modp3072
      rekey_time = 1080s
   }
}

secrets {
   ike-uwan {
      id-1 = 198.51.100.1
      id-2 = 203.0.113.1
      secret = MySecretKey123
   }
}
```

> proposals 格式为 `加密算法-认证算法-DH组`，DH 组编号对应关系：1=modp768, 2=modp1024, 5=modp1536, 14=modp2048, 15=modp3072, 16=modp4096。

### BGP 模式

以下示例在感兴趣流模式基础上修改，假设：
- BGP 隧道网段：`169.254.1.0/30`
- CE 侧 BGP 地址：`169.254.1.1`（ASN 65001）
- UWAN 侧 BGP 地址：`169.254.1.2`（ASN 65000）

#### /etc/swanctl/swanctl.conf

```
connections {
   uwan {
      local_addrs  = 198.51.100.1
      remote_addrs = 203.0.113.1

      local {
         auth = psk
         id = 198.51.100.1
      }
      remote {
         auth = psk
         id = 203.0.113.1
      }

      children {
         net-net {
            local_ts  = 0.0.0.0/0
            remote_ts = 0.0.0.0/0

            esp_proposals = aes128-sha1-modp3072
            rekey_time    = 3600s
            dpd_action    = clear

            mode = tunnel
            start_action = trap
         }
      }

      version = 2
      proposals = aes128-sha1-modp3072
      rekey_time = 1080s
   }
}

secrets {
   ike-uwan {
      id-1 = 198.51.100.1
      id-2 = 203.0.113.1
      secret = MySecretKey123
   }
}
```

> BGP 模式下 `local_ts` 和 `remote_ts` 均设为 `0.0.0.0/0`，路由信息通过 BGP 自动交换。

#### BGP 邻居配置（FRR 示例）

安装 FRR 后，编辑 `/etc/frr/frr.conf`：

```
router bgp 65001
 bgp router-id 169.254.1.1
 neighbor 169.254.1.2 remote-as 65000
 !
 address-family ipv4 unicast
  neighbor 169.254.1.2 activate
  neighbor 169.254.1.2 soft-reconfiguration inbound
 exit-address-family
```

> 需要先在隧道接口上配置 BGP 地址（如 `ip addr add 169.254.1.1/30 dev <tunnel-if>`），FRR 才能建立邻居关系。

## 四、验证配置

### 检查 IKE SA 状态

```bash
swanctl -l
```

正常输出应包含 `ESTABLISHED` 状态的 IKE SA，以及 `INSTALLED` 状态的 Child SA：

```
uwan: #1, ESTABLISHED, IKEv2, ...
  net-net: #1, INSTALLED, TUNNEL, ...
```

### 检查 BGP 邻居状态（BGP 模式）

```bash
vtysh -c "show bgp summary"
```

正常输出中邻居状态应为 `Established`：

```
Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
169.254.1.2     4      65000      10       8        0    0    0 00:01:23            5
```

### 检查路由学习

```bash
# 查看从 BGP 学习到的路由
vtysh -c "show bgp ipv4"

# 查看系统路由表
ip route show
```

### 常见问题

- **IKE SA 未建立**：检查 PSK 是否一致、加密算法和 DH 组是否匹配
- **Child SA 未安装**：检查 `local_ts` / `remote_ts` 是否正确（感兴趣流模式需手动指定，BGP 模式为 `0.0.0.0/0`）
- **BGP 邻居未建立**：先确认 IPSec 隧道已建立，再检查 ASN、BGP 地址配置是否与控制台一致
- **动态 IP 接入时 IKE 协商失败**：CE 为动态 IP 时，需在 CE 侧配置 `local_id`，UWAN 侧配置对端 ID 类型
