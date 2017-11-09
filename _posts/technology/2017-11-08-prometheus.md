## 介绍
*Prometheus*是一个开源的监控/告警系统。自2012年起，许多公司和组织都已经在使用*Prometheus*，而且拥有非常活跃的开发者社区。目前*Prometheus*已经成为*Cloud Native Computing Foundation*中排行第二，仅次于*Kubernetes*。

## 主要特性

* 多维度的时间序列数据模型,并且由*metric name*和*key/value*对标识
* 利用灵活的查询语言来提供更多的数据维度
* 不依赖分布式存储。单个服务节点可以做到自治
* 时间序列数据的收集是通过http协议从监控源中*pull*
* 通过一个中间网关，以支持*push*方式推送数据
* 支持自动发现监控源或通过静态配置
* 多种图型和仪表盘组件

## 系统组成
* *Prometheus*服务，负责从监控源中抓取和存储时间序列数据
* 提供多种编程语言的客户端库，简化监控源生成监控数据
* 提供多种第三方服务的*exporter*，如HAProxy,StatsD, Graphite等
* 告警管理器用于处理告警

> 大部分的Prometheus组件是使用Go语言编写的，因为Prometheus利用静态库简化编译和部署。

![architecture]

## 重要概念
### 数据模型
*Prometheus*中存储的都是时间序列的数据，使用*metric name*和多个*label*进行标识。*label*是一个*key/value*对，可以将*label*看作是*metric name*的一些属性吧(个人理解)。

采样数据是由两个字段组成：64位的float类型数据（监控值）及精确到毫秒的时间戳。

采样数据使用下面的表达式来标识：
`<metric name>{<label name>=<label value>, ...}`

**example:**
`api_http_requests_total{method="POST", handler="/messages"}`

以上示例中，`api_http_requests_total`是*metric name*，`method="POST"`和`handler="/messages"`就是两个*label*的*key/value*对。

### 测量类型
#### Counter 
*counter*是一种只会累计递增的测量类型。只适用于监控增量数据，如：请求数量，错误发生数量...不适用于监控会减少的数据。

#### Gauge
*Gauge*与*counter*的区别是，可以既可以监控增加，又可以监控会减少的数据了。

#### Histogram
*Histogram*采样的是观察值(如请求时延或响应大小），并且在配置的bucket中进行计数。它同时提供了所有观察值的总和。

*Histogram*会从一次采样中衍生出多个不同*metric name*（这些*metric name*都有一个相同的*basename*）的时间序列数据进行上报：

* 对观察值所归属的bucket进行计数，*metric name*格式：`<basename>_bucket{le="上限值"}`
* 观察值的总和，*metric name*格式：`<basename>_sum`
* 所有观察值的总计数，*metric name*格式：`<basename>_count`

*Histogram*需要使用`histogram_quantile()`函数去计算*Histogram*分位数(φ-quantiles)或*Histogram*的聚合数。*Histogram*适合用于计算*Apdex(Application Performance Index)*分数。

#### Summary
*Summary*类似于*Histogram*，采样的也是观察值(如请求时延或响应大小）。然而，它提供了观察值的总计数和总和值，而且计算基于一个时间窗口配置的分位数。

*summary*也会从一次采样中衍生出多个不同*metric name*进行上报：

* 符合φ-quantiles (0 ≤ φ ≤ 1)分位数的观察值（如φ-quantiles配置为0.9，意思就是90%的观察值都低于的那个观察值是多少），*metric name*格式：`<basename>{quantile="<φ>"}`。关于φ-quantiles的更多介绍请参考[xgboost之分位点算法](http://datavalley.github.io/2017/09/11/xgboost%E6%BA%90%E7%A0%81%E4%B9%8B%E5%88%86%E4%BD%8D%E7%82%B9)
* 观察值的总和，*metric name*格式：`<basename>_sum`
* 所有观察值的总计数，*metric name*格式：`<basename>_count`

#### Histogram与Summary的区别
两者主要区别是计算φ-quantiles分位数的方式不同，*Summary*是在监控源中进行计算，而*Histogram*是在*Prometheus*服务器中使用`histogram_quantile()`函数进行计算。

如果现在有一个需求，如何通过*Histogram*计算最近5分钟内，95%的请求延时低于一个何值？

首先，需要为*Histogram*添加多个期望的bucket进行监控，如0.1,0.2,0.3,0.4,0.5等。接着使用以下函数计算出quantiles分位数：
`histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`

通过以上表达式计算出来的quantiles分位数就是0.1,0.2,0.3,0.4,0.5之中的一个。

另一个需求，如何监控95%的请求不能超过300毫秒？

通过以下表达式可以计算出低于300毫秒的请求数占比：

	sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m])) by (job)
	/
	sum(rate(http_request_duration_seconds_count[5m])) by (job)

如果以上表达式的结果低于95%就可以进行告警了。

由以上示例可以知道，*Histogram*计算quantiles分位数误差较大，误差的高低由定义的bucket的精细度有关，超精细，则计算quantiles分位数越精准。而*Summary*的误差则低得多，这依赖于客户端类库的实现。

### 任务Job与实例Instance（监控源）

在*Prometheus*中，一个单节点的监控源被称为一个实例(Instance)，多个具有相同目的的实例集合被称为任务(Job)。

*Prometheus*会自动地为时间序列数据添加两个*label*：

1. `job:<jobname>` 
2. `instance:<host>:<port>`

*Prometheus*每次向Instance抓取数据的时候，都会生成以下的时间序列：

* up{job="\<job-name\>", instance="\<instance-id\>"} ： 1 表示实例健康，2 表示实例不可用
* scrape_duration_seconds{job="\<job-name\>", instance="\<instance-id\>"}:抓取数据的延时
* scrape_samples_post_metric_relabeling{job="\<job-name\>", instance="\<instance-id\>"}:the number of samples remaining after metric relabeling was applied.
* scrape_samples_scraped{job="\<job-name\>", instance="\<instance-id\>"}: 采样数目

## Prometheus安装与配置

请参考[官网](https://prometheus.io/docs/prometheus/latest/installation/)

## 规则配置

*Prometheus*包含有两种规则配置。一种称为记录规则(recording rule)，另一种则称为告警规则(alerting rule)。

记录规则可以理解为：对监控源中获取的时间序列数据进行的二次运算而生成新的时间序列数据，并把这些数据记录下来。

配置示例：

	groups:
	  - name: example  #分组名称
	    interval: 1m
	    rules: 
	    - record: job:http_inprogress_requests:sum   #生成的时间序列数据的metric name
	      expr: sum(http_inprogress_requests) by (job)    #Prometheus查询表达式
	      labels: ["sum":"query"]   #添加label

         - alert: HighErrorRate   #告警名称
           expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5  #告警表达式
           for: 10m  #多长时间间隔告警一次
           labels:   #添加label
             severity: page
           annotations: #告警内容
             summary: High request latency


另外，告警可以使用模板，关于告警模板的使用也请查阅[官网](https://prometheus.io/docs/prometheus/latest/configuration/template_examples/)

## Prometheus查询

请查阅[官网](https://prometheus.io/docs/prometheus/latest/querying/basics/)


[architecture]: https://sin90lzc.github.io/images/prometheus/architecture.svg
