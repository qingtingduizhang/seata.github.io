---
title: Metrics
keywords: Seata
description: Metrics。
---

### Metrics
#### 设计思路
1. Seata作为一个被集成的数据一致性框架，Metrics模块将尽可能少的使用第三方依赖以降低发生冲突的风险；
2. Metrics模块将竭力争取更高的度量性能和更低的资源开销，尽可能降低开启后带来的副作用；
3. 配置式，Metrics是否激活、数据如何发布，取决于对应的配置；开启配置则自动启用并将度量数据发布到[Prometheus](https://github.com/prometheus)；
4. 不使用Spring，使用SPI(Service Provider Interface)加载扩展；
5. 初始仅发布核心Transaction相关指标，之后结合社区的需求，逐步完善运维方面的所有其他指标。

#### 模块说明
由2个核心API模块`seata-metrics-api`和`seata-metrics-core`，以及N个实现模块例如`seata-metrics-registry-compact`、`seata-metrics-exporter-prometheus`构成：
- seata-metrics-api模块

此模块是Metrics的核心，将作为Seata基础架构的一部分被TC、TM和RM引用，它内部**没有任何具体实现代码**，仅包含接口定义，定义的内容包括：
1. Meter类接口：`Gauge`、`Counter`、`Timer`...
2. 注册容器接口`Registry`
3. Measurement发布接口`Publisher`

>提示：Metrics本身在开源领域也已有很多实现，例如
>1. [Netflix-Spectator](https://github.com/Netflix/spectator)
>2. [Dropwizard-Metrics](https://github.com/dropwizard/metrics)
>3. [Dubbo-Metrics](https://github.com/dubbo/dubbo-metrics)

>它们有的轻而敏捷，有的重而强大，由于也是“实现”，因此不会纳入`seata-metrics-api`中，避免实现绑定。

- seata-metrics-core模块

Metrics核心模块，根据配置组织（加载）1个Registry和N个Exporter；

- seata-metrics-registry-compact模块

这是我们提供的默认（内置）的Registry实现，不使用其它Metrics开源库，轻量级的实现了以下四种Meter：

- seata-metrics-exporter-prometheus模块

这是我们默认提供的Metrics实现，不使用其它Metrics开源实现，并轻量级的实现了以下三个Meter：

| Meter类型  | 描述                                                                                                                         |
| --------- | ------------------------------------------------------------ |
| Gauge     | 单一最新值度量器                                                                                                                |
| Counter   | 单一累加度量器，可增可减                                                                                                         |
| Summary   | 多Measurement输出计数器，将输出`total`(合计)、`count`(计数)和`tps`(合计/时间间隔)，无单位  |
| Timer     | 多Measurement输出计时器，将输出`total`(合计)、`count`(计数)、`max`(最大)和`average`(合计/计数)，支持微秒为单位累计              |

>说明：
>1. 未来可能增加更丰富复杂的度量器例如Histogram，这是一种可以本地统计聚合75th, 90th, 95th, 98th, 99th,99.9th...的度量器，适合某些场合，但需要更多内存。
>2. 所有的计量器都将继承自Meter，所有的计量器执行measure()方法后，都将归一化的生成1或N个Measurement结果。

它也会实现一个内存的Registry和PrometheusExporter，将度量数据同步给Prometheus。

>说明：不同的监控系统，采集度量数据的方式不尽相同，例如Zabbix支持用zabbix-agent推送，Prometheus则推荐使用prometheus-server[拉取](https://prometheus.io/docs/practices/pushing/)的方式；同样数据交换协议也不同，因此往往需要逐一适配。

#### 如何使用
##### 新增配置
如果需要开启TC的Metrics，需要在其配置中增加配置项：

比如file.conf
```text
## metrics settings
metrics {
  enabled = true
  registryType = "compact"
  # multi exporters use comma divided
  exporterList = "prometheus"
  exporterPrometheusPort = 9898
}
```
或者1.5.0+中使用application.yaml
```yaml
seata:
  metrics:
    enabled: true
    registryType: compact
    exporterList: prometheus
    exporterPrometheusPort: 9898
```

或者使用第三方配置中心如nacos,apollo等

[请参考此处](https://github.com/seata/seata/tree/develop/script/config-center)将seata metrics配置项上传到对应配置中心,也可打开对应配置中心控制台进行手动添加

```properties
metrics.enabled=true
metrics.registryType=compact
metrics.exporterList=prometheus
metrics.exporterPrometheusPort=9898
```

之后启动TC，即可在`http://tc-server-ip:9898/metrics`上获取到Metrics的文本格式数据。

>提示：默认使用`9898`端口，Prometheus已登记的端口列表[在此](https://github.com/prometheus/prometheus/wiki/Default-port-allocations)，如果想更换端口，可通过`metrics.exporter.prometheus.port`配置修改。

##### 下载并启动Prometheus
下载完毕后，修改Prometheus的配置文件`prometheus.yml`，在`scrape_configs`中增加一项抓取Seata的度量数据：
```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'seata'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['tc-server-ip:9898']
```

##### 查看数据输出
推荐结合配置[Grafana](https://prometheus.io/docs/visualization/grafana/)获得更好的查询效果，初期Seata导出的Metrics包括：
- TC :

| Metrics    | 描述    |
| ------ | --------- |
| seata.transaction(role=tc,meter=counter,status=active/committed/rollback) | 当前活动中/已提交/已回滚的事务总数  |
| seata.transaction(role=tc,meter=summary,statistic=count,status=committed/rollback) | 当前周期内提交/回滚的事务数  |
| seata.transaction(role=tc,meter=summary,statistic=tps,status=committed/rollback) | 当前周期内提交/回滚的事务TPS(transaction per second) |
| seata.transaction(role=tc,meter=timer,statistic=total,status=committed/rollback) | 当前周期内提交/回滚的事务耗时总和 |
| seata.transaction(role=tc,meter=timer,statistic=count,status=committed/rollback) | 当前周期内提交/回滚的事务数  |
| seata.transaction(role=tc,meter=timer,statistic=average,status=committed/rollback) | 当前周期内提交/回滚的事务平均耗时   |
| seata.transaction(role=tc,meter=timer,statistic=max,status=committed/rollback) | 当前周期内提交/回滚的事务最大耗时 |

>提示：seata.transaction(role=tc,meter=summary,statistic=count,status=committed/rollback)和seata.transaction(role=tc,meter=timer,statistic=count,status=committed/rollback)的值可能相同，但它们来源于两个不同的度量器。

- TM：

稍后实现，包括诸如：
seata.transaction(role=tm,name={GlobalTransactionalName},meter=counter,status=active/committed/rollback) : 以GlobalTransactionalName为维度区分不同Transactional的状态。

- RM：

稍后实现，包括诸如：
seata.transaction(role=rm,name={BranchTransactionalName},mode=at/mt,meter=counter,status=active/committed/rollback)：以BranchTransactionalName为维度以及AT/MT维度区分不同分支Transactional的状态。

#### 如何扩展
如果有下面几种情况：
1. 您不是使用Prometheus作为运维监控系统，但希望能够将Seata的Metrics数据集成进Dashboard中；
2. 您需要更复杂强大的度量器类型，这些度量器在其他Metrics实现库中已有，希望集成这些第三方依赖直接使用；
3. 您需要改变默认Metric的Measurement输出，例如在Timer中增加一个`min`或`sd`(方差)；
4. ...

那么需要自行扩展Metrics的实现，请创建新的模块项目例如`seata-metrics-xxxx`，之后：
- 针对1：您需要实现新的Exporter；
- 针对2：您可以改变默认Registry的实现，返回第三方的Meter计量器实现；
- 针对3：您可以修改对应Meter的实现，包括`measure()`方法返回的Measurement列表。
