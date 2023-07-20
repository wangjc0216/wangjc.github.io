---
layout: post
title:  "监控平台分享"
date:   2023-06-01 20:00:00 +0800
categories: [observability,prometheus]
---
> 之前在公司内部做的一个简单分享，主要是在监控告警的链路中针对关键点发掘潜在需求及痛点。

<br />
# 概述
监控平台的开发与搭建用于 采集、存储、计算、分发、可视化。本文主要针对链路中的每个节点需要注意的问题进行阐述。
![img.png]({{ site.url }}/assets/monitor1.png)

![img.png]({{ site.url }}/assets/prometheus_arch.png)
<br />
<br />
# 采集
- pull 与 push
  pull失败就意味着目标地址(target)出现问题，通过拉取数据就可以判断连通状态。
  
  主动权在服务端，而非客户端。
  
  批处理脚本的指标上报，则需要通过metrics推送到pushgateway组件，prometheus对pushgateway进行拉取。
  <br />
  <br />
- 维度(Dimension)

  数据源的不同维度： 按照主机、容器、中间件、微服务等不同维度进行指标采集。

  数据描述(label)的不同维度：metrics通过label来描述不同维度信息，对没有用到的label进行删除。
  <br />
  <br />
- 采样频率

  15s or 1min？tradeoff
  <br />
  <br />
# 时序数据存储

- 数据的价值

  短期: 监控数据价值在于了解服务健康状态、监控告警。

  长期: 针对long-term的指标数据，对服务的（可用性、变更操作）进行复盘评审。
 <br />
 <br />
- 时序数据的特点

  写多读少。

  数据随着时间推移价值逐渐降低。

  90%以上的查询是在最近26h的数据中。
  
  数据的压缩率高。
  <br />
  <br />
- TSDB
  prometheus的问题：单点故障。

  解决单点问题：联邦 or TSDB集群？

  TSDB： Thanos、Victormetrics、M3DB、Mirir。
  
  M3DB： 分布式TSDB、long term、降采样、analytics。
  <br />
  <br />

# 指标计算
> 用于看板查询与告警

- 告警、可视化响应

  对于告警or统计面板视图，需要对指标进行提前计算，在查询时会有效降低响应延时和内存使用。
  <br />
  <br />
- 查询限制、重查询 OOM

  简单来说，是对prometheus的查询做限流，防止查询过大导致的OOM；通过query日志，找到偶发的慢查询源头并修正。
  <br />
  <br />
# 告警分发

- 丰富prometheus的告警规则

  对prometheus rule可进行与或非等的逻辑判断。
  <br />
  <br />
- receiver： oncall排班 与im通信工具

  通过oncall轮值，有利于团队梯队的建设，使得新人快速融入团队。
  <br />
  <br />
- 告警认领、数据持久化
  <br />
  <br />
# 指标可视化与使用

![img.png]({{ site.url }}/assets/monitor_dashboard.png)

- grafana /  exporter dashboard

- dashboard template。针对微服务，会有些公共library的指标埋点，如在接口访问、DB访问层上。

- 指标的设定

  在prometheus，四种指标类型，分别为counter、gauge、histogram、summary。可以按照不同的规则进行查询。
 <br />
 <br />
- RED&USE方法观测指标

  USE 方法主要着眼于资源内部，RED 方法则关注请求、实际工作以及外部视角（即来自服务消费方的视角）。
  
  其中，USE 原则指的是，按照如下三个维度来规划资源监控指标：
    - 利用率（Utilization），资源被有效利用起来提供服务的平均时间占比；
  
    - 饱和度（Saturation），资源拥挤的程度，比如工作队列的长度；
  
    - 错误率（Errors），错误的数量。

  而 RED 原则指的是，按照如下三个维度来规划服务监控指标：

    - 每秒请求数量（Rate）；
  
    - 每秒错误数量（Errors）；
  
    - 服务响应时间（Duration）；
 <br />
 <br />
- SLA/SLO

  SLA通常指出什么服务被提供，它如何被支持，时间、地点、消耗、性能，对此服务负责的团队等。

  SLO则是SLA具体的衡量指标，如availablity，throughout，frequency，response time，or quality。
  <br />
  <br />
# 观测的更多维度

- 三个基石： metrics、log、trace（sidecar）。

- event，离散的数据，可通过event数据观测到具体服务实例的生命周期、升级变化。
  <br />
  <br />
# 产品化

- 可快速上手的监控产品，夜莺： https://github.com/ccfos/nightingale
  <br />
  <br />
# Reference

oncall服务： [linkedin oncall](https://github.com/linkedin/oncall)

TSDB  M3DB：[m3db](https://m3db.io/)

SLA/SLO: [what-is-the-difference-between-sla-service-level-agreement-and-slo-service-le](https://sqa.stackexchange.com/questions/22213/what-is-the-difference-between-sla-service-level-agreement-and-slo-service-le)

Google SLO： [google SLO](https://sre.google/workbook/implementing-slos/)

Prometheus Sample Lifecycle:  [sample lifecycle](https://docs.google.com/presentation/d/1DNfqByJLnoV2-io1RCOf8hbJ79Fsm3Il9e7n1UZaWAg/edit#slide=id.g1ac46e5f455_0_18)