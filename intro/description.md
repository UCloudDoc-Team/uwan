# 产品概述

## 什么是UWAN智联？

UWAN智联是UCloud推出的SDWAN接入服务，为企业各门店和分支机构提供就近接入、加密传输、一键互联的智能组网功能。企业可通过「UWAN智联+UGN云联网」整体解决方案实现分支机构/门店网络到云上资源、托管网络和其他私有站点的互联互通。 

## 产品形态
UWAN智联包含以下几种产品形态：
* CE客户网关（Customer Edge） 
适用于大型分支机构、IDC机房之间组网，以及大型分支机构接入上云。 
您可以在现有网络的出口设备上配置公网VPN隧道，连接到就近地域的UWAN虚拟路由器。此种接入方式可以保留现有网络出口设备的性能、安全、流量控制等功能，对现网环境无侵入。
* CPE智能网关（Customer Provider Edge） 
适用于中小型分支机构、门店网络之间组网，以及中小型分支机构、门店网络接入上云。 
您可以在您的节点网络中部署CPE智能网关，通过UWAN控制器对CPE的自动化配置下发，可自动完成节点网络到UWAN虚拟路由器的接入。此种接入方式可以做到简易部署后即开即用，一键接入。

## 产品组件
- UWAN虚拟路由器：UWAN虚拟路由器是地域级别的虚拟网关，为周边节点网络提供站点接入。接入同一个UWAN虚拟路由器的节点网络，可自行实现互联互通。
- UWAN接入带宽包：每一台UWAN虚拟路由器均会绑定一个UWAN接入带宽包，包含UWAN接入带宽和UWAN接入IP。用于CE客户网关/CPE智能网关到UWAN虚拟路由器的数据传输。
 
- 其他组件：UGN云联网
基于Segment Routing技术推出的新一代全球化网络服务产品，实现用户不同地域VPC之间，VPC与本地数据中心、托管网络、分支节点网络之间搭建云上、云下一体化的客户私有网络。帮助用户构建一张企业级的全球化网络。