### 操作步骤

1. 登录UCloud控制台https://console.ucloud.cn/
2. 进入到UWAN产品页面并点击CPE智能网关，找到想要进行配置的CPE设备点击配置管理。

![img](/images/net01.png)

3. 在CPE网络配置中，WAN口和LAN口都需要进行配置，分别点击对应的“编辑配置”按钮进行配置。

![img](/images/net02.png)

4. 进行WAN口网络配置，配置完成后点击“确定”按钮保存配置。

![img](/images/net03.png)

| 字段名称  | 字段说明                                       |
| --------- | ---------------------------------------------- |
| 设备WAN口 | 选择物理WAN口                                  |
| MTU       | 最大运输单元，范围为 [512,1500]                |
| 接入方式  | 接入云上网络的方式                             |
| 连接类型  | WAN口接入网络的方式                            |
| SNAT      | 开启后可以将UCPE局域网的内网地址转换成公网地址 |

5. 进行LAN口网络配置，配置完成后点击“确定”按钮保存配置。

![img](/images/net04.png)

| 字段名称      | 字段说明                                                     |
| ------------- | ------------------------------------------------------------ |
| MTU           | 最大运输单元，范围为 [512,1500]                              |
| LAN侧网关配置 | UCPE网关的LAN口网络配置，生效于UCPE全部LAN口                 |
| DHCP-Server   | 开启后UCPE下游设备可通过 DHCP 协议动态获取 IP 地址           |
| DHCP地址池    | 期望为UCPE下游设备DHCP分配的IP地址池，该地址池范围应与LAN侧网关配置同段 |
| 地址租期      | DHCP为某一设备分配的IP地址过期时间，超过此时间未续租则需要重新分配IP地址 |