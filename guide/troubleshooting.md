# 故障排查

本文档提供 UWAN 智联使用过程中的常见故障排查方法，包括 VPN 隧道协商失败、BGP 邻居异常、跨地域不通等问题的定位与解决。

---

## VPN 隧道协商失败

### IKE 一阶段协商失败

#### PSK 不匹配

**现象**：VPN 隧道状态一直为"未建立"，CE 侧日志显示认证失败。日志关键字：`Decryption failed! mismatch of preshared secrets`、`mismatch of preshared secrets`、`invalid HASH_V1 payload length`、`could not decrypt payloads`。

**排查步骤**：

1. 在 UCloud 控制台查看 VPN 隧道配置中的预共享密钥
2. 检查 CE 侧配置中的 PSK 是否与控制台完全一致（注意前后空格和大小写）
3. 如果使用 strongSwan，检查 `/etc/swanctl/swanctl.conf` 中 `secret` 字段

**解决方法**：确保两端 PSK 完全一致，修改后重启 IPSec 服务（`systemctl restart strongswan` 或 `swanctl --load-all`）。如需重新触发协商，可在控制台重置 VPN 隧道。

#### 加密算法或认证算法不匹配

**现象**：隧道建立失败，日志提示 `no proposal chosen`、`NO_PROPOSAL_CHOSEN`、`HASH mismatched`、`authentication failure`、`invalid encryption algorithm` 等。

**排查步骤**：

1. 在控制台查看 VPN 隧道的 IKE 加密算法、认证算法配置
2. 检查 CE 侧 `proposals` 配置格式是否正确（如 `aes128-sha1-modp3072`）
3. 确认两端加密算法和认证算法完全匹配
4. 如对端配置了多种算法，建议将对端算法配置修改为与 UWAN 控制台相同

**解决方法**：将 CE 侧的 proposals 调整为与 UWAN 控制台配置一致。

> proposals 格式为 `加密算法-认证算法-DH组`，DH 组编号：1=modp768, 2=modp1024, 5=modp1536, 14=modp2048, 15=modp3072, 16=modp4096。

#### DH 组不匹配

**现象**：隧道建立失败，日志提示 `received KE type 14, expected 2`、`failed to compute dh value`、`rejected dh_group` 等。

**排查步骤**：确认 CE 侧的 DH 组配置与 UWAN 控制台一致。UWAN 默认 DH 组为 15（modp3072）。

**解决方法**：修改 CE 侧 DH 组配置，确保与控制台匹配。注意 IKE 阶段和 IPSec 阶段的 DH 组需分别检查。

#### IKE 版本不匹配

**现象**：隧道建立失败，日志提示 `unknown ikev2 peer`。

**排查步骤**：

1. 确认两端 IKE 版本一致（均为 v1 或均为 v2）
2. 如对端支持自动选择版本，建议指定为与 UWAN 一致的版本

**解决方法**：修改 CE 侧 IKE 版本配置，确保两端一致。推荐使用 IKEv2。

#### 协商模式不匹配

**现象**：隧道建立失败，日志提示 `in Identity not acceptable Aggressive mode`、`not acceptable Identity Protection mode`。

**排查步骤**：确认两端协商模式一致。推荐均使用主模式（main）。

**解决方法**：将两端协商模式修改为一致。如两端均为 main 模式仍协商不成功，可尝试改为野蛮模式（aggressive）。

#### LocalID / RemoteID 不匹配

**现象**：隧道建立失败，日志提示 `does not match peers id`、`Unknow peer id`、`no matching peer config found`、`invalid-id-information` 等。

**排查步骤**：

1. 确认 UWAN 侧 LocalID 与 CE 侧 RemoteID 一致，UWAN 侧 RemoteID 与 CE 侧 LocalID 一致
2. IKEv1 主模式下，LocalID/RemoteID 默认为 IP 地址格式，请确保格式正确
3. IKEv2 下如 ID 配置无误仍失败，需同时检查加密算法、认证算法、DH 组是否一致

**解决方法**：修正两端 ID 配置，确保交叉对应。

!> 若 CE 为动态 IP 接入，CE 侧 ID 类型无法自动配置，需手动填写 `local_id`，UWAN 侧对端 ID 类型选择"域名标识"或"IP 地址标识"并填入 CE 侧对应的 ID 值。

### IPSec 二阶段协商失败

#### 感兴趣流网段不匹配

**现象**：隧道建立失败，日志提示 `traffic selector mismatch`、`traffic selector unacceptable`、`TS_UNACCEPTABLE`、`INVALID_ID_INFORMATION` 等。

**排查步骤**：

1. 确认 UWAN 侧本端网段与 CE 侧对端网段相同，UWAN 侧对端网段与 CE 侧本端网段相同
2. IKEv1 模式下感兴趣流支持多网段，每个网段单独建立一条 IPSec 隧道
3. IKEv2 模式下多网段可能存在兼容性问题（见下方"友商对接——华为策略模式"）

**解决方法**：确保两端感兴趣流网段交叉对应。多网段场景建议使用 BGP 模式或目的路由模式。

#### IPSec DH 组（PFS）不匹配

**现象**：二阶段协商失败，日志提示 `pfs group mismatched`、`message lacks KE payload`。

**排查步骤**：检查 IPSec 阶段的 PFS（完美前向保密）DH 组配置是否一致。UWAN 默认 PFS 为 Disable。

**解决方法**：
- UWAN 侧 PFS 为 Disable 时，CE 侧需关闭 PFS
- UWAN 侧 PFS 开启时，CE 侧需使用相同 DH 组

#### 封装模式不匹配

**现象**：日志提示 `encmode mismatched`。

**解决方法**：UWAN 仅支持隧道模式（tunnel），不支持传输模式。请确保 CE 侧使用隧道模式。

#### 安全协议不匹配

**现象**：日志提示 `proto_id mismatched`。

**解决方法**：UWAN 支持 ESP 和 AH 协议，默认使用 ESP。请确保 CE 侧安全协议配置与 UWAN 一致。

### DPD 问题

#### DPD 超时导致隧道断开

**现象**：VPN 隧道频繁断开并自动重连，日志提示 `DPD check timed out, enforcing DPD action`。

**排查步骤**：

1. 确认两端 DPD 功能是否均已开启，确保两端开启状态相同
2. 检查 DPD 延迟时间是否过短，短暂网络抖动可能触发隧道断开
3. 检查 DPD 动作配置：`clear` 会直接断开隧道，`trap` 会等待重连
4. 检查两端之间的网络质量和路由连通性

**解决方法**：
- 适当增大 DPD 延迟时间（建议 30 秒以上）
- 将 DPD 动作从 `clear` 改为 `trap`，避免短暂抖动导致隧道断开

#### DPD 两端配置不一致导致单向丢包

!> 这是现网高频问题，务必确保两端 DPD 策略一致。

**现象**：一端因 DPD 检测断开隧道后，另一端仍认为隧道正常。表现为：
- 对端 ping 本端失败（本端收到 ESP 包但 `ip xfrm state` 不存在，无法解密，包被丢弃）
- 本端 ping 对端成功（`start_action = trap` 触发隧道重新协商）

**根因**：两端 DPD 配置不一致时，一端因 DPD 超时删除了 SA 状态，另一端未开启 DPD 仍保留旧 SA。此时对端发来的 ESP 数据包在本端无法匹配已删除的 SA，被内核直接丢弃；且本端 `ip xfrm policy` 虽存在但缺少对应 `state`，也无法触发协商。

**解决方法**：
- **确保两端 DPD 配置完全一致**：`dpd_delay`、`dpd_timeout`、`dpd_action` 均需相同
- 推荐配置：`dpd_delay = 30`，`dpd_action = trap`（不建议使用 `clear`，会直接删除 SA 导致数据包丢弃）
- 如果 CE 侧使用 strongSwan，`start_action` 建议设为 `trap`，`close_action` 建议设为 `restart`，保证隧道断开后能自动重建

### SA 超时配置不当

**现象**：隧道在 SA 超时时间到达后无法正常 rekey，导致断开。日志提示 `long lifetime proposed`。

**排查步骤**：

1. 检查 IKE SA 超时时间和 IPSec SA 超时时间配置
2. 确认 CE 侧的 rekey 时间配置与 UWAN 侧匹配

**解决方法**：确保 CE 侧的 IKE rekey 时间和 IPSec rekey 时间与 UWAN 控制台配置一致。两端 SA 生存周期不强制要求相同，但为确保稳定性，推荐配置相同。

### NAT 穿越问题

**现象**：隧道协商失败，日志提示 `ignore the packet, received unexpecting payload type 130`。

**排查步骤**：确认两端 NAT 穿越功能状态是否一致。

**解决方法**：两端 NAT 穿越需同时开启或同时关闭。如 CE 在 NAT 网关之后，两端均需开启 NAT 穿越。

### 对端网关不响应

**现象**：日志提示 `sending retransmit 1 of request message ID 0`、`retransmission count exceeded the limit`、`giving up after 5 retransmits`。

**排查步骤**：

1. 确认对端网关设备可正常收发 IPSec 协议报文
2. 确认 CE 客户网关 IP 地址与对端设备实际 IP 相同
3. 确认对端设备无异常（如故障重启）
4. 在对端设备上使用 `ping`/`mtr`/`traceroute` 访问 UWAN 侧公网 IP，确认网络可达
5. 检查对端设备的访问控制策略：放行 UDP 500 和 4500 端口，放行 UWAN 侧公网 IP

**解决方法**：修复网络连通性或安全策略后，在控制台重置 VPN 隧道触发重新协商。

---

## 友商对接注意事项

跨厂商 IPSec VPN 对接时，因各厂商对协议的实现存在差异，可能遇到兼容性问题。以下为已验证的友商对接注意事项。

### 华为

#### 策略模式下 IKEv2 多子网隧道建立不全

**问题现象**：华为设备与 UWAN 建立 IPSec VPN 后，仅第一条子网隧道可正常建立，其余子网隧道协商失败，导致多子网通信不全。

**根本原因**：双方 IKEv2 流量选择器（TS）协商行为不兼容：
- 华为侧（策略模式）：为每对"本端子网-对端子网"建立精确匹配的独立子 SA，发起 `CHILD_SA_CREATE` 时 TS 载荷仅包含当前特定子网
- UWAN 侧：在响应中返回本地配置的全部子网范围，而非对端请求的特定子网
- 华为设备进行严格 TS 校验，发现响应与请求不匹配后协商失败

**解决方案**：

| 方案 | 说明 | 推荐 |
|------|------|------|
| 华为侧切换为路由模式 | 配置一条指向 UWAN VPC 网段的静态路由，出接口为 IPSec 隧道接口 | **推荐** |
| 降级为 IKEv1 | 在 UWAN 侧将感兴趣流拆分为多条独立隧道，每条对应一对明确子网 | 配置繁琐，管理复杂 |

#### 路由模式下 Rekey 兼容性

华为端通信模式配置为路由模式（0.0.0.0/0）时，默认不接受 UWAN 侧发起的 rekey 请求，必须由华为本端发起协商重新建立才会生效。如遇 rekey 后隧道中断，请确认由华为侧作为协商发起方。

### FortiGate

FortiGate 的 IKE Config 中 **localid-type 必须配置为 address，不能为 auto**，否则 IKE 协商会失败。

日志表现：
```
no matching peer config found
```

**解决方法**：将 FortiGate 的 `localid-type` 从 `auto` 改为 `address`。

### 阿里云

#### NAT 穿越导致入向丢包

**问题现象**：UWAN 侧出向正常，入向 UWAN 网关可抓到包，但客户侧丢包。从 `swanctl -l` 可看到流量走 4500 端口（NAT 穿越已开启）。

**解决方法**：阿里云侧关闭 NAT 穿越。UWAN 侧后续版本将解决此兼容性问题。

---

## BGP 邻居无法建立

### BGP 网段或 ASN 配置错误

**现象**：IPSec 隧道已建立，但 BGP 邻居状态不是 Established。

**排查步骤**：

1. 在控制台查看 VPN 隧道的 BGP 配置（ASN、隧道网段）
2. 检查 CE 侧 BGP 配置中的 `remote-as` 是否与 UWAN 侧 ASN 一致
3. 检查 BGP 隧道网段是否正确（格式 `169.254.x.x/30`）
4. 检查 CE 侧 BGP 地址是否为隧道网段中分配的本端地址

**解决方法**：修正 CE 侧 BGP 配置中的 ASN、邻居地址，使其与控制台配置完全一致。

### IPSec 隧道未先建立

**现象**：BGP 邻居无法建立，IPSec 隧道状态也不正常。

**排查步骤**：BGP 依赖 IPSec 隧道进行通信，需先确保 IPSec 隧道正常建立。

**解决方法**：参照"VPN 隧道协商失败"章节先解决 IPSec 问题，再排查 BGP。

---

## 单地域通了但跨地域不通

### UGN 路由未同步

**现象**：同一地域内的 CE 之间可以互通，但跨地域访问不通。

**排查步骤**：

1. 在 UCloud 控制台检查 UWAN 虚拟路由器是否已关联云联网（UGN）
2. 在虚拟路由器详情页的"路由表"中，检查是否能学习到远端地域的路由
3. 确认云联网中是否已关联两端地域的 UWAN 虚拟路由器
4. 确认各节点网段不重叠（网段冲突会导致路由无法加入）

**解决方法**：

- 确保两端 UWAN 虚拟路由器均已加入同一云联网实例
- 如果路由表中缺少远端路由，等待 1-2 分钟让路由同步完成
- 如果存在网段冲突，需要修改冲突节点的子网划分

---

## 带宽限速不生效

### 带宽包配置与实际限速不一致

**现象**：流量超过带宽包限额但未被限速，或带宽未达到限额就被限速。

**排查步骤**：

1. 在控制台确认 UWAN 接入带宽包的额度配置
2. 注意带宽上限限制的是出入向总量（入向 + 出向之和不超过上限）
3. 检查流量计费模式下是否存在历史欠费导致限速（欠费 3 天后带宽会被限速为 0kbps）

**解决方法**：

- 确认带宽包额度是否已正确调整（调整频率最高 1 分钟 1 次）
- 如有欠费，补缴后等待限速策略恢复
- 如带宽额度需超过 100Mbps，请联系客户经理

---

## CPE 上线后无法联网

### WAN/LAN 配置错误

**现象**：CPE 已上线但下挂设备无法上网。

**排查步骤**：

1. 在控制台"CPE 智能网关 - 配置管理"中检查 WAN 口和 LAN 口配置
2. 确认 WAN 口接入方式和连接类型是否正确
3. 确认 LAN 侧网关配置和 DHCP 地址池是否在同一网段
4. 确认 SNAT 是否已开启（未开启则内网地址无法转换为公网地址）

**解决方法**：修正 WAN/LAN 配置，确保 WAN 口能正常获取 IP，LAN 口 DHCP 地址池与网关同段，SNAT 已开启。

### DHCP 服务异常

**现象**：CPE 下挂设备无法获取 IP 地址。

**排查步骤**：

1. 检查 LAN 口配置中 DHCP-Server 是否已开启
2. 检查 DHCP 地址池范围是否与 LAN 侧网关同段
3. 检查地址租期是否合理

**解决方法**：开启 DHCP-Server，确保地址池范围正确（如 LAN 网关为 192.168.1.1/24，则地址池可为 192.168.1.100-192.168.1.200）。

### VPN 隧道未建立

**现象**：CPE 已上线但无法与 UWAN 虚拟路由器建立 VPN 连接。

**排查步骤**：

1. 检查 CPE 的 WAN 口是否能正常访问公网
2. 检查 CPE 的 TLS 证书是否过期（在控制台监控页面查看）
3. 检查 UWAN 虚拟路由器状态是否正常

**解决方法**：确保 CPE WAN 口网络畅通，证书有效。CPE 的 VPN 连接由 UWAN 控制器自动下发配置，如长时间无法建立请提交工单。

---

## 日志错误速查表

以下表格列出 IPSec VPN 协商过程中常见的日志错误信息及对应排查方法，供快速定位问题。

| 错误场景 | 日志关键字 | 排查方法 |
|----------|-----------|----------|
| CE 网关 IP 不匹配 | `UNSUPPORTED_CRITICAL_PAYLOAD` | 确认 CE 客户网关 IP 与对端设备实际 IP 相同 |
| IKE 加密/认证/DH 不匹配 | `HASH mismatched`、`packet lacks expected payload`、`authentication failure` | 确认 IKE 阶段加密算法、认证算法、DH 组两端一致 |
| IPSec 加密算法不匹配 | `invalid encryption algorithm`、`trns_id mismatched`、`rejected enctype` | 确认 IPSec 阶段加密算法两端一致 |
| IKE 认证算法不匹配 | `authtype mismatched`、`rejected hashtype` | 确认 IKE 阶段认证算法两端一致 |
| IKE DH 组不匹配 | `received KE type 14, expected 2`、`rejected dh_group`、`proposal mismatch, transform type:4` | 确认 IKE 阶段 DH 组两端一致 |
| PSK 不匹配 | `mismatch of preshared secrets`、`Decryption failed`、`could not decrypt payloads` | 确认预共享密钥一致；同时确认 IKE/IPSec 各阶段算法均一致 |
| LocalID/RemoteID 不匹配 | `does not match peers id`、`Unknow peer id`、`no matching peer config found`、`invalid-id-information` | 确认 UWAN 侧 LocalID = CE 侧 RemoteID，UWAN 侧 RemoteID = CE 侧 LocalID |
| DPD 载荷顺序不兼容 | `ignore information because the message has no hash payload` | 确认两端 DPD 载荷顺序均为 hash-notify |
| DPD 超时 | `DPD check timed out, enforcing DPD action` | 确认两端 DPD 功能均已开启且配置一致；检查网络连通性 |
| IKE 版本/协商模式不匹配 | `unknown ikev2 peer`、`not acceptable Identity Protection mode` | 确认 IKE 版本和协商模式两端一致 |
| 感兴趣流网段不匹配 | `traffic selector mismatch`、`TS_UNACCEPTABLE`、`INVALID_ID_INFORMATION` | 确认两端感兴趣流交叉对应；IKEv2 多网段需注意兼容性 |
| IPSec PFS 不匹配 | `pfs group mismatched`、`message lacks KE payload` | 确认 IPSec 阶段 PFS DH 组配置一致 |
| 封装模式不匹配 | `encmode mismatched` | 确认使用隧道模式（tunnel） |
| 安全协议不匹配 | `proto_id mismatched` | 确认使用 ESP 协议 |
| Lifetime 不匹配 | `long lifetime proposed` | 推荐 SA 生存周期两端配置相同 |
| 对端不响应 | `sending retransmit`、`retransmission count exceeded`、`giving up after 5 retransmits` | 检查网络连通性、对端设备状态、安全策略是否放行 UDP 500/4500 |
| 收到 Delete 报文 | `received DELETE IKE_SA`、`received DELETE for ESP CHILD_SA` | 在对端排查发送 Delete 的原因 |
| 协商未开始 | 无明显错误 | 尝试在控制台重置 VPN 隧道触发重新协商 |

---

## 抓包排查指引

如需进一步排查 VPN 隧道问题，可在 UWAN 宿主机网络命名空间（ns）内使用 tcpdump 抓包分析。

### 抓取内层报文

```bash
tcpdump -i $ns内网口 -nne host <内层IP地址>
# 示例：tcpdump -i ns-xxxx-i -nne host 192.168.0.100 and host 10.0.0.100
```

### 抓取外层报文

```bash
tcpdump -i $ns内网口 -nne esp
# 示例：tcpdump -i ns-xxxx-i -nne esp
```

> 抓包需联系 UCloud 技术支持在宿主机 ns 内操作。CE 侧可直接在设备上抓包。
