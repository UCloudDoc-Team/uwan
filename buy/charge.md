# 计费规则

## 计费项
| **计费项** | **描述**                                                     |
| --------- | :----------------------------------------------------------- |
| UWAN实例费用 | 创建UWAN虚拟路由器时根据所选择的UWAN实例型号收取对应的实例费，按实例类型、数量和购买周期计算。 |
| UWAN接入带宽费用 | UWAN实例与外网网络分别独立计费，若外网网络选择带宽计费则会产生接入带宽费用，带宽费用为所配置的线路带宽值 * 带宽单价 * 购买周期。 |
| UWAN接入流量费用   | UWAN实例与外网网络分别独立计费，若外网网络选择流量计费则会产生接入流量费用，流量为后付费，每天按实际使用流量计算。 |



## 计费模式

| **计费模式** |**计费方式**| **计费单位**                                                 | **适用场景**                     |
| ------------ | ------------ | ------------------------------------------------------------ | -------------------------------- |
| 固定带宽     | 预付费       | 元/月，最少购买时长为1个月；                                 | 适用于带宽稳定的场景    |
| 流量计费       | 后付费       | 元/月，实例需预付费，最少购买时长为1个月；<br />元/GB，流量费用根据每天实际使用的流量总量计费； |适用于爆发业务的场景 |



- ### 固定带宽计费

**计费方式** 

计费方式：预付费方式；

**计费周期**

以月/年为单位，购买时立即生成账单；

**计费算法**

    总费用=（实例月单价 + 带宽月单价 * 带宽量）* （有效时间/计费月总时长）； 
    
    若月内带宽包中有带宽的升降级调整，则在调整这个时间节点之前的还是按照原固定带宽进行计费，之后的按照新的固定带宽进行计费，同时产生对应的退费/补缴操作；
    
    计费时间的颗粒度精确到秒；
    
    若用户购买周期不是整数自然月，则当月的最终带宽费用=原带宽费用*有效天数/计费月天数进行折算；

**计费示例**

    例如：客户A，于8月5日上午10点30分0秒，以带宽计费模式购买了配置为300M的UWAN，则客户A本月账单为：
    
    UWAN实例单价：90元/月        
    
    带宽月单价为：110元/M/月；   
    
    带宽值为：300M；
    
    当月有效时间系数为：26天13小时30分0秒/31天=（2246400+46800+1800）/2678400= 0.8569  
    
    （8月份总共31天，用户8月5日上午10点30分0秒购买的，客户的实际使用时长为8月5日这天从10点30分0秒开始计费至24点+8月6日至8月31日全天，总共26天13小时30分0秒，总共2295000秒）
    
    本月实际账单为：（90 + 300 * 110）* 0.8569= 28354.82 元



- ### 流量计费

**计费方式**

计费方式：实例预付费+流量后付费；

**计费周期**
UWAN实例以月/年为单位，预付费；
UWAN流量以天为单位，后付费；

**计费算法**

    总费用= 实例费 + 流量费 ； 实例费按购买周期一次性结算，流量费每日结算一次。
    
    实例费= 实例数量 * 实例单价 * （有效时间/计费月总时长）； 
    
    流量费= 日流量总量 * 流量单价 ；    

**计费示例**

    例如：客户B，于8月5日上午10点30分0秒，以流量计费模式购买了配置为300M的UWAN，则客户A本月账单为：
    
    带宽上限为：300M；        
    
    UWAN实例月单价为：90元/实例/月；  
    
    UWAN流量单价为：0.90元/GB；
    
    月流量用量为：10000 GB；
            
    当月有效时间系数为：26天13小时30分0秒/31天=（2246400+46800+1800）/2678400= 0.8569 ；
    
    （8月份总共31天，用户8月5日上午10点30分0秒购买的，客户的实际使用时长为8月5日这天从10点30分0秒开始计费至24点+8月6日至8月31日全天，总共26天13小时30分0秒，总共2295000秒）
    
    本月实例账单为：90 * 0.8569 + 0.9 * 10000  = 9077.121 元