# CE 侧 VPN 配置指南

本文档提供 CE 客户网关侧的 VPN 配置示例，以 strongSwan (Linux)、H3C 防火墙和华为防火墙为例，涵盖感兴趣流模式和 BGP 模式两种 IPSec 路由模式。

## 配置前提

- 已在 UCloud 控制台创建 UWAN 虚拟路由器，并获取其公网 IP 地址
- 已创建 CE 客户网关，并完成 VPN 隧道配置
- CE 侧设备已安装相应的 VPN 软件/固件

## IPSec 参数对照表

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
| 封装模式 | 隧道模式 | 隧道模式（不支持传输模式） |

### BGP 参数（BGP 模式时需配置）

| 参数 | 说明 |
|------|------|
| 本端 ASN | CE 侧 BGP 自治系统号 |
| 对端 ASN | UWAN 侧 BGP 自治系统号（在控制台创建隧道时指定） |
| BGP 隧道网段 | 建立 BGP 邻居所使用的网段，格式为 `169.254.x.x/30`（在控制台创建隧道时指定） |
| 本端 BGP 地址 | 隧道网段中的本端 IP |
| 对端 BGP 地址 | 隧道网段中的对端 IP |

### DPD 参数

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| DPD 开关 | 是否启用 Dead Peer Detection | 两端均开启 |
| DPD 延迟 | 发送 DPD 探测的间隔时间 | 30 秒 |
| DPD 超时 | 判定对端不可达的超时时间（仅 IKEv1） | — |
| DPD 动作 | 检测到对端不可达后的动作 | trap（等待重连） |

!> **两端 DPD 配置必须一致**。如果一端开启 DPD 而另一端未开启，可能导致一端因 DPD 超时删除 SA 状态后，另一端仍保留旧 SA，造成单向丢包。详见[故障排查 - DPD 问题](/uwan/guide/troubleshooting.md)。

## strongSwan 配置示例

以下示例假设：
- UWAN 侧公网 IP：`203.0.113.1`
- CE 侧公网 IP：`198.51.100.1`
- CE 侧内网网段：`172.16.1.0/24`
- UWAN 侧内网网段：`10.0.0.0/16`
- 预共享密钥：`MySecretKey123`

### 感兴趣流模式

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

      dpd_delay = 30

      children {
         net-net {
            local_ts  = 172.16.1.0/24
            remote_ts = 10.0.0.0/16

            esp_proposals = aes128-sha1-modp3072
            rekey_time    = 3600s

            dpd_action    = trap
            start_action  = trap
            close_action  = restart

            mode = tunnel
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

      dpd_delay = 30

      children {
         net-net {
            local_ts  = 0.0.0.0/0
            remote_ts = 0.0.0.0/0

            esp_proposals = aes128-sha1-modp3072
            rekey_time    = 3600s

            dpd_action    = trap
            start_action  = trap
            close_action  = restart

            mode = tunnel
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

## H3C 防火墙配置示例

以下示例假设：
- UWAN 侧公网 IP：`203.0.113.1`
- H3C 设备公网接口：`GigabitEthernet 2/0`，IP 为 `198.51.100.1`，下一跳 `198.51.100.254`
- H3C 设备私网接口：`GigabitEthernet 4/0`，IP 为 `172.16.1.1`
- CE 侧内网网段：`172.16.1.0/24`
- UWAN 侧内网网段：`10.0.0.0/16`
- IKE 版本：IKEv2，加密算法 aes128，认证算法 sha1，DH 组 15（modp3072）
- 预共享密钥：`MySecretKey123`

> 不同型号版本的 H3C 防火墙配置可能存在差异，请根据实际版本参考相应文档或咨询厂商。以下配置示例仅供参考，建议在实际环境中验证后再部署。

### 感兴趣流模式

#### 步骤一：接口网络配置

```
# 公网接口
interface GigabitEthernet 2/0
 ip addr 198.51.100.1 24
 quit

# 私网接口
interface GigabitEthernet 4/0
 ip addr 172.16.1.1 24
 quit

# 接口加入安全域
security-zone name Untrust
 import interface GigabitEthernet 2/0
 quit
security-zone name Trust
 import interface GigabitEthernet 4/0
 quit

# 配置对端 UWAN 公网地址路由
ip route-static 203.0.113.1 32 198.51.100.254
```

#### 步骤二：IPSec 提议和策略配置

```
# 配置 IPSec 安全提议
ipsec transform-set to-uwan-trans
 encapsulation-mode tunnel
 protocol esp
 esp authentication-algorithm sha1
 esp encryption-algorithm aes-cbc-128
 pfs dh-group15
 quit

# 配置 IKEv2 安全提议和策略
ikev2 proposal to-uwan-prop
 dh group15
 encryption aes-cbc-128
 integrity sha1
 prf sha1
 quit
ikev2 policy to-uwan-policy
 priority 1
 proposal to-uwan-prop
 quit

# 配置 IKE keychain
ikev2 keychain to_uwan_key
 peer to-uwan-peer
  address 203.0.113.1 32
  identity address 203.0.113.1
  pre-shared-key plaintext MySecretKey123
  quit
 quit

# 配置 IKE profile
ikev2 profile to-uwan-profile
 authentication-method local pre-share
 authentication-method remote pre-share
 keychain to_uwan_key
 identity local address 198.51.100.1
 match remote identity address 203.0.113.1 32
 sa duration 1080
 dpd interval 30 periodic
 quit

# 配置 IPSec profile
ipsec profile to-uwan-profile isakmp
 transform-set to-uwan-trans
 ikev2-profile to-uwan-profile
 sa duration time-based 3600
 quit
```

#### 步骤三：隧道接口配置

```
# 配置 Tunnel 接口
interface tunnel 1 mode ipsec
 ip address unnumbered interface GigabitEthernet 2/0
 tunnel protection ipsec profile to-uwan-profile
 source 198.51.100.1
 destination 203.0.113.1
 quit

# Tunnel 接口加入安全域
security-zone name Untrust
 import interface Tunnel 1
 quit

# 配置对端内网网段路由指向 Tunnel 接口
ip route-static 10.0.0.0 16 Tunnel 1
```

#### 步骤四：安全策略配置

```
# 放通 IKE 协商报文和 IPSec 数据报文
acl advanced 3001
 rule 0 permit ip
 quit
zone-pair security source any destination any
 packet-filter 3001
 quit
```

> 以上为简化配置，请根据实际需要补充细粒度的安全策略规则。

### BGP 模式

在感兴趣流模式基础上，修改以下配置：

假设 BGP 隧道网段为 `169.254.1.0/30`，H3C 侧 BGP 地址 `169.254.1.1`（ASN 65001），UWAN 侧 BGP 地址 `169.254.1.2`（ASN 65000）。

#### 修改 Tunnel 接口 IP 和路由

```
# 配置 Tunnel 口 BGP 地址
interface tunnel 1 mode ipsec
 ip address 169.254.1.1 30
 tunnel protection ipsec profile to-uwan-profile
 source 198.51.100.1
 destination 203.0.113.1
 quit

# 删除感兴趣流模式的静态路由
undo ip route-static 10.0.0.0 16 Tunnel 1

# 配置 BGP
bgp 65001
 peer 169.254.1.2 as-number 65000
 address-family ipv4 unicast
  peer 169.254.1.2 enable
  network 172.16.1.0 24
 exit-address-family
```

> IPSec 提议中感兴趣流对应的 `local_ts`/`remote_ts` 在 H3C 设备上通过 ACL 引用实现，BGP 模式下 ACL 应匹配 `0.0.0.0/0`。

## 华为防火墙配置示例

以下示例假设：
- UWAN 侧公网 IP：`203.0.113.1`
- 华为设备公网接口：`GigabitEthernet 1/0/1`，IP 为 `198.51.100.1`
- CE 侧内网网段：`172.16.1.0/24`
- UWAN 侧内网网段：`10.0.0.0/16`
- IKE 版本：IKEv2，加密算法 aes128，认证算法 sha1，DH 组 15

!> 华为防火墙与 UWAN 对接时，**必须使用路由模式而非策略模式**。策略模式下 IKEv2 多子网存在流量选择器（TS）协商不兼容问题，会导致仅第一条子网隧道建立成功。详见[故障排查 - 友商对接注意事项](/uwan/guide/troubleshooting.md)。

> 不同型号版本的华为防火墙配置可能存在差异，请根据实际版本参考相应文档或咨询厂商。以下配置示例仅供参考，建议在实际环境中验证后再部署。

### 路由模式（推荐）

#### 步骤一：接口和路由配置

```
# 公网接口
interface GigabitEthernet 1/0/1
 ip address 198.51.100.1 255.255.255.0
 quit

# 私网接口
interface GigabitEthernet 1/0/2
 ip address 172.16.1.1 255.255.255.0
 quit

# 安全区域
firewall zone untrust
 add interface GigabitEthernet 1/0/1
 quit
firewall zone trust
 add interface GigabitEthernet 1/0/2
 quit

# 配置到 UWAN 侧的路由
ip route-static 203.0.113.1 255.255.255.255 198.51.100.254
```

#### 步骤二：IPSec 配置

```
# 配置 IKE 提议
ike proposal 1
 encryption-algorithm aes-128
 dh group15
 authentication-algorithm sha1
 authentication-method pre-share
 sa duration 1080
 quit

# 配置 IKE 对等体
ike peer uwan
 pre-shared-key MySecretKey123
 ike-proposal 1
 remote-address 203.0.113.1
 dpd type periodic
 dpd idle-time 30
 quit

# 配置 IPSec 提议
ipsec proposal 1
 encapsulation-mode tunnel
 transform esp
 esp authentication-algorithm sha1
 esp encryption-algorithm aes-128
 pfs dh-group15
 quit

# 配置 IPSec 策略（路由模式，感兴趣流为 0.0.0.0/0）
ipsec policy uwan 1 isakmp
 ike-peer uwan
 proposal 1
 sa duration time-based 3600
 quit
```

#### 步骤三：隧道接口和路由

```
# 创建 Tunnel 接口
interface Tunnel 1
 ip address 169.254.1.1 255.255.255.252
 tunnel-protocol ipsec
 ipsec policy uwan
 source 198.51.100.1
 destination 203.0.113.1
 quit

# Tunnel 接口加入安全区域
firewall zone untrust
 add interface Tunnel 1
 quit

# 配置到 UWAN 侧内网的路由指向 Tunnel 接口
ip route-static 10.0.0.0 255.255.0.0 Tunnel 1
```

#### 步骤四：安全策略

```
# 放通 IKE 和 IPSec 流量
security-policy
 rule name ipsec_ike
  source-zone local
  destination-zone untrust
  source-address 198.51.100.1 32
  destination-address 203.0.113.1 32
  action permit
  service udp 500
  service udp 4500
 quit
 rule name ipsec_data
  source-zone trust
  destination-zone untrust
  source-address 172.16.1.0 24
  destination-address 10.0.0.0 16
  action permit
 quit
 rule name ipsec_data_return
  source-zone untrust
  destination-zone trust
  source-address 10.0.0.0 16
  destination-address 172.16.1.0 24
  action permit
 quit
```

### BGP 模式

在路由模式基础上，增加 BGP 配置：

```
# 配置 BGP
bgp 65001
 peer 169.254.1.2 as-number 65000
 ipv4-family unicast
  peer 169.254.1.2 enable
  network 172.16.1.0 255.255.255.0
 quit
```

> 华为防火墙路由模式下 rekey 注意事项：默认不接受对端发起的 rekey 请求，需由华为侧作为协商发起方。如遇 rekey 后隧道中断，请检查协商发起方配置。

## 验证配置

### strongSwan

```bash
# 检查 IKE SA 状态
swanctl -l
# 正常输出应包含 ESTABLISHED 状态的 IKE SA，以及 INSTALLED 状态的 Child SA

# 检查 BGP 邻居状态（BGP 模式）
vtysh -c "show bgp summary"
# 正常输出中邻居状态应为 Established

# 查看从 BGP 学习到的路由
vtysh -c "show bgp ipv4"

# 查看系统路由表
ip route show
```

### H3C 防火墙

```
# 查看 IKE SA
display ike sa

# 查看 IPSec SA
display ipsec sa

# 查看 BGP 邻居（BGP 模式）
display bgp peer

# 查看路由表
display ip routing-table
```

### 华为防火墙

```
# 查看 IKE SA
display ike sa

# 查看 IPSec SA
display ipsec sa

# 查看 BGP 邻居（BGP 模式）
display bgp peer

# 查看路由表
display ip routing-table
```

## 常见问题

- **IKE SA 未建立**：检查 PSK 是否一致、加密算法和 DH 组是否匹配
- **Child SA 未安装**：检查 `local_ts` / `remote_ts` 是否正确（感兴趣流模式需手动指定，BGP 模式为 `0.0.0.0/0`）
- **BGP 邻居未建立**：先确认 IPSec 隧道已建立，再检查 ASN、BGP 地址配置是否与控制台一致
- **动态 IP 接入时 IKE 协商失败**：CE 为动态 IP 时，需在 CE 侧配置 `local_id`，UWAN 侧配置对端 ID 类型
- **华为策略模式多子网不通**：华为设备与 UWAN 对接时必须使用路由模式，详见[故障排查 - 友商对接注意事项](/uwan/guide/troubleshooting.md)
- **DPD 配置不一致导致单向丢包**：确保两端 DPD 配置完全一致，详见[故障排查 - DPD 问题](/uwan/guide/troubleshooting.md)
