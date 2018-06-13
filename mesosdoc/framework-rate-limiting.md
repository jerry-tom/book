

## Framework速率限制

&emsp;&emsp; framework速率限制是在mesos 0.20.0引入的一个特性。

### 什么是Framework速率限制

&emsp;&emsp; 在多框架环境中，该特征旨在通过使来自其他(例如，开发，批量)框架的主节制消息来保护高SLA(例如，生产，服务)框架的吞吐量。

&emsp;&emsp; 为了限制来自框架的消息，Mesos集群操作者为其主体标识的每个框架设置一个qps（每秒查询）值（你也可以一同时限制一组框架，但是除非另有说明，否则我们将假定在此文档中使用单独的框架；请参阅下面的RateLimits Protobuf定义和配置注释）。Mesos master承诺不会以高于qps的速率处理来自该框架的消息。未完成的消息被存储在Mesos master内存中。

### 速率限制配置

&emsp;&emsp; 以下是一个简单的配置文件(JSON格式)，可以使用—rate_limits的Mesos master标志进行指定。

``` json
{
  "limits": [
    {
      "principal": "foo",
      "qps": 55.5
      "capacity": 100000
    },
    {
      "principal": "bar",
      "qps": 300
    },
    {
      "principal": "baz",
    }
  ],
  "aggregate_default_qps": 333,
  "aggregate_default_capacity": 1000000
}
```

&emsp;&emsp; 在这个例子中，框架foo通过配置的qps和容量被限制，框架bar被赋予无限能力，框架baz不受任何限制。如果存在第四个框架qux或没有主体连接到mesos master的框架，则会受到aggregate_default_qps和aggregate_default_capacity规则的限制。

#### 配置说明

&emsp;&emsp; 以下是JSON配置中的字段

- **principal**: (必须的) 唯一标识，被限流或明确地给予无限制速率的实体

  - 它应该匹配框架的FrameworkInfo.principal。（参见定义）
  - 你可以多个framework使用相同的prinicipal(比如，一些mesos frameworks每个任务运行一个新的framework实例)，在这种情况下，来自使用相同principal的所有framworks的组合流量将被限制在指定的QPS。

- **qps**: (可选项) 每秒查询次数，即速率

  - 一旦设置，mesos master保证处理来自这个prinicipal的消息不会高于这个速率。然而mesos master可能会慢于这个速度，特别是如果指定的速率太高。
  - 为了明确地给予framwork无限制的速率(即不限制它)，添加一个没有qps的条目到limits中。

- **capacity**：(可选项) 这个principal框架可以传递给mesos master的outstading消息数量。如果没有指定，这个princaipal被赋予无限的capacipty。请注意，如果capacity设置太大或没有设置，消息队列会使用更多内存，从而引起mesos master OOM，是可能的。

  - 注意：如果qps没有设置，capacity将被忽略

- 使用aggregate_default_qps和aggregate_default_capacity来保护mesos master从未指定prinicipal的framework。所有未在limits中指定的framework采用这个默认的qps和capacity。

  - qps和capacity是它们全部的总值，即，它们的组合流量被一起节流。
  - 和上面一样，如果aggregate_default_qps没有指定，aggregate_default_capacity将被忽略。
  - 如果这些字段不存在，未指定的frameworks将不被限制。与上面的显示方法相比，这是一种给予framework无限制速率的隐含方式(仅限于在limits中使用prinicipal的条目)。我们推荐使用明确的选项，特别当mesos master不要求验证时，以防意外的framework压垮mesos master。

  ### 使用Framework速率限制

  #### 监控Framework流量

  &emsp;&emsp; 当一个framework注册到mesos master时，mesos master暴露来之这个framework的所有收到和处理的消息的计数器在它的指标节点:http://<master>/metrics/snapshot。举例，framework `foo` 有两个消息计数器  `frameworks/foo/messages_received` 和 frameworks/foo/messages_processed。如果没有Framework速率限制，两个数字应该相差很小或不想等(因为消息被尽可能快的处理)，但是当一个framework受到限制时，差异表示由于限制而导致的未完成消息。

  &emsp;&emsp; 通过持续监视计数器，你可以推算出消息到达的速率，以及framework的消息队列增长有多快(如果framework被限制)。这应该描述网络流量方面的框架特点。

  #### 配置速率限制

  &emsp;&emsp; 由于Framework速率限制的目标是防止低SLA frameworks使用太多资源，而不是尽可能精确地建模其流量和行为，你开始可以使用大的qps值来限流它们。它们受到限流（不管配置的qps）的事实在提供来自高SLA framework的信息时具有更高的优先级，因为它们被尽快处理。为了计算mesos master有多少处理容量，你必须知道mesos master进程的内存限制，它通常用于服务于类似工作负载而没有速率限制的内存量（例如，使用ps -o rss $MASTER_PID）和framwork message的平均大小(排队的消息存储为具有一些附加字段的串行化协议缓冲区)，你应该在配置中加上所有的容量值。然而由于这种计算不精确，你应该从容忍合理的临时framework突发的小值开始，但是远小于内存限制，以便未来没有容量限制的master和frameworks预留足够的空间。

#### 处理“容量超标”错误

&emsp;&emsp; 当framework超过容量时，一个FrameworkErrorMessage发送给framework, 该framework将终止scheduler driver并调用 error()回调。它不会杀死任何任务或schudler本身。framework开发人员可以选择重启或故障迁移schudler实例，以纠正丢失消息的后果(除非你的framework不假定所有发送给master的消息都被处理)。

&emsp;&emsp;在0.20.0版本之后，我们将通过让master在framework的消息队列开始聚集时发送提前告警来迭代这个特性([MESOS-1664](https://issues.apache.org/jira/browse/MESOS-1664), 认为它是一个软限制)。scheduler可以通过限流自己应对（避免错误信息）或如果这是设计中的临时突发，则忽略这个告警。

&emsp;&emsp;在提前告警实现前，我们不推荐使用速率限制特性来限制产品frameworks，除非你们明确错误消息的后果。当然，使用它可以限制其他framework保护生产framework，如果它没有明确启用，它对mesos master没有任何影响。

