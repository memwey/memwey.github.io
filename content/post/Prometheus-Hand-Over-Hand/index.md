---
title: "Prometheus 手把手"
description: 面向开发, 手把手教还不会的话, 直接打死
date: 2022-05-10T16:36:07+08:00
hidden: false
comments: true
draft: false
toc: false
tags: 
  - Prometheus
  - Database
---

因为 Prometheus 在应用中一共只有四种 metric 类型, 只要根据需要监测的目标选择了 metrics 类型, 后续的告警以及可视化配置就有固定的套路了.

### Counter

Counter 用来监控单调递增的量, 当发生重置时 Prometheus 会自动处理归零的数据. 这是一个非常基本的类型, 一个系统中很多数据都可以通过此类型进行计数, 比如系统接受的请求数, 系统发生的错误数, 再比如对第三方接口的调用数, 第三方接口返回的错误数.

业务代码如下

```python
from prometheus_client import Counter
c = Counter(
    'web_request_total', 'web request total counter',
    ['method', 'endpoint', 'status_code']
)
c.labels('get', '/', '200').inc()
c.labels('post', '/submit', '500').inc()
```

此时在业务暴露出来的端点上会有这样两条记录

```
web_request_total{method="/submit",method="post",status_code="500"} 1
web_request_total{method="/",method="get",status_code="200"} 1
```

对于 Counter 我们一般看增长量和增长速率

* `sum(increase(web_request_total[5m]))` 五分钟内总请求数
* `sum(increase(web_request_total{status_code!~"^2.."}[1m]))` 一分钟内状态码不为2xx的总请求数
* `sum(irate(web_request_total[1m]))` 一分钟内总请求数增长率

告警则一般设置为

* 一分钟内总请求中有有超过 5% 的 5xx 错误, 持续一分钟
```
expr: sum(rate(web_request_total{status_code=~"^5.."}[1m])) / sum(rate(web_request_total[1m])) * 100 > 5
for: 1m
```

* 一分钟内 5xx 错误的请求增长率高于 100%, 持续一分钟
```
expr: sum(irate(web_request_total{status_code=~"^5.."}[1m])) > 100
for: 1m
```

### Gauge

Gauge 是一个瞬时量, 在时间上来看是可增可减的, 用以记录系统在某个时刻的当前状态, 比如当前实例线程数, 当前系统负载, 某个队列的长度, 某个队列的消费延时等.

业务代码如下

```python
from prometheus_client import Gauge
g1 = Gauge(
    'kafka_consumer_lag_seconds', 'kafka consume lag in second',
    ['topic', 'consumer_group']
)
g1.labels('k-test', 'grp1').set(233)
g2 = Gauge(
    'redis_connection_pool_count', 'redis connection poll count',
    ['state']
)
g2.labels('idle').set(10)
g2.labels('active').set(22)
g2.labels('total').set(32)
```

此时在业务暴露出来的端点上会有这样的记录

```
kafka_consumer_lag_seconds{topic="k-test",consumer_group="grp1"} 233
redis_connection_pool_count{state="idle"} 10
redis_connection_pool_count{state="active"} 22
redis_connection_pool_count{state="total"} 32
```

对于 Gauge 我们一般直接看其值或者其值的百分比

* `kafka_consumer_lag_seconds{topic="k-test"}` 查看 k-test 这个 topic 最新的消费延迟
* `avg_over_time(kafka_consumer_lag_seconds[1m])` 查看应用全部 kafka 队列一分钟的平均消费延时
* `max_over_time(kafka_consumer_lag_seconds[1m])` 查看应用全部 kafka 队列每一分钟内的最大消费延时

* `redis_connection_pool_count{state="active"} / ignoring(state) redis_connection_pool_count{state="total"}` 查看应用 redis 连接池的使用百分比

告警则一般设置为

* 一分钟内队列的平均消费延时大于 10min, 持续一分钟
```
expr: avg_over_time(kafka_consumer_lag_seconds[1m]) > 600
for: 1m
```

* Redis 连接池平均占用超过 90%, 持续一分钟
```
expr: avg_over_time(redis_connection_pool_count{state="active"}[1m]) /  ignoring(state) avg_over_time(redis_connection_pool_count{state="total"}[1m]) * 100 > 90
for: 1m
```

### Histogram 和 Summary

Histogram 可以看作是一个复合的 Counter 类型, 它会将数值按区间计数, 常用于记录 P99, P95 类的数据. 可以用来记录比如接口响应耗时情况, 接口返回数据大小情况等.

```python
from prometheus_client import Histogram
h = Histogram(
    'http_request_duration_seconds', 'Api requests response time in seconds',
    ['api', 'method'], (0.1, 0.25, 1, 4, 10)
)
h.labels('/products', 'post').observe(0.0923)
h.labels('/products', 'post').observe(0.3672)
```

此时在业务暴露出来的端点上会有这样的记录, 产生了多条数据, 其中 `_sum` 是值的总和, `_count` 是值的计数, `_bucket` 中有一个命名为 `le` 特殊的 label, 其中的区间值是我们在代码中配置的, 是值在区间内的计数.

```
http_request_duration_seconds_sum{api="/products", "method"="post"} 0.4595
http_request_duration_seconds_count{api="/products", "method"="post"} 2
http_request_duration_seconds_bucket{api="/products", "method"="post", le="0.1"} 1
http_request_duration_seconds_bucket{api="/products", "method"="post", le="0.25"} 1
http_request_duration_seconds_bucket{api="/products", "method"="post", le="1"} 2
http_request_duration_seconds_bucket{api="/products", "method"="post", le="4"} 2
http_request_duration_seconds_bucket{api="/products", "method"="post", le="10"} 2
http_request_duration_seconds_bucket{api="/products", "method"="post", le="+Inf"} 2
```

对于 Histogram 一般可以求均值或者 P95, P99 之类的值, 也可以做普通的 Counter 使用

* `rate(http_request_duration_seconds_sum{api="/products", "method"="post"}[5m]) / rate(http_request_duration_seconds_count{api="/products", "method"="post"}[5m])` 查看 /products 端点上 POST 请求五分钟内的耗时均值
* `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{api="/products", "method"="post"}[5m]))` 查看 /products 端点上 POST 请求五分钟内的耗时 P99
* `increase(http_request_duration_seconds_count{api="/products", "method"="post"}[1m])` 查看 /products 端点上 POST 请求一分钟内的请求数

告警则一般设置为

* 两分钟内某个接口 P95 大于 10s, 持续两分钟
```
expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[2m])) by (api, method)) > 10
for: 2m
```

一些有用的资料
* [awesome-prometheus-alerts](https://awesome-prometheus-alerts.grep.to/rules)
* [How to visualize Prometheus histograms in Grafana](https://grafana.com/blog/2020/06/23/how-to-visualize-prometheus-histograms-in-grafana/)
