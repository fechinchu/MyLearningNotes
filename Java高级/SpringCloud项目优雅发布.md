# 1.SpringCloud项目优雅发布

## 1.背景

1.SpringCloud发布式项目,部署在多个节点上.但中间涉及到一个问题,当执行kill命令,服务虽然关闭,但Eureka那里依然保存这台服务的IP,请求依然会跑到这台服务器上.直到持续数十秒后,Eureka将该服务的IP剔除,如果请求量大,会导致大量请求在发版的过程中出现异常,所以要想到一个更优雅的方式来部署服务.

## 2.方案一

调用Eureka的接口,让Eureka自动剔除该服务IP.获取服务的AppID和InstanceID,分别对应响应中的`<app>`和`<instanceId>`;

`GET http://IP:port/eureka/apps/`

![image-20200610203434092](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200610203434092.png)

* 从可用服务列表中剔除该服务

`PUT http://IP:port/eureka/apps/AppID/InstanceID/status?value=DOWN`

* 将该服务加到可用服务列表中

`PUT http://IP:port/eureka/apps/AppID/InstanceID/status?value=UP`

# 2.分布式链路追踪

## 2.1.背景

调用链路追踪最先由google在Dapper这篇论文中提出,主要为了解决的问题:收集一个请求经过多个服务处理的过程中,每一个服务处理的具体的执行情况.目前开源相关实现有Zipkin,Dapper,HTrace,SkyWalking等,也有类似OpenTracing这样被大量支持的统一规范;

Spring Cloud Sleuth为Spring Cloud系统实现了分布式跟踪解决方案

![](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200610210817769.png)

## 2.2.分布式链路追踪核心架构

![image-20200610211113295](https://fechin-leyou.oss-cn-beijing.aliyuncs.com/PicGo/image-20200610211113295.png)

