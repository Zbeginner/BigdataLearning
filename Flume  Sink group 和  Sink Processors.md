# Flume容错机制 : Sink Processors

Sink Processors是作用在sink组件上的容错机制。通过调度Sink Groups（就是分了组的Sinks），可以做到负载均衡(**load_balance** Processors)和类似HDFS中Namenode高可用(**Failover** Processors )那样的目的。

### Failover Sink Processor

| Name                              | Default | Description                                    |
| --------------------------------- | ------- | ---------------------------------------------- |
| **sinks**                         | -       | sink的一个集合，用空格分隔，一个sink不建议分组 |
| **processor.type**                | default | Failover处理器就设置为`failover`               |
| **processor.priority.<sinkName>** | -       | 组类某个sink的优先级                           |
| processor.maxpenalty              | 30000   | 失败的sink最大回退时间(毫秒)                   |

优先级大的先被调度，如果优先级大的挂了，会找活着的中最大优先级的sink然后让其工作(一直只有一个sink工作)，如果这个时候原来那个优先级更大的活过来了，不会抢占位置，直到当前sink挂了才会再次选最大优先级的工作。

### load_balance Processors

| Name                          | Default     | Description                                         |
| ----------------------------- | ----------- | --------------------------------------------------- |
| **sinks**                     | -           | sink的一个集合，用空格分隔，一个sink不建议分组      |
| **processor.type**            | default     | Failover处理器就设置为`load_balance`                |
| processor.backoff             | false       | 以指数级回退失败的sink                              |
| **processor.selector**        | round_robin | 调度方式有`round_robin`和`random`两种，或者自定义的 |
| processor.selector.maxTimeOut | 30000       | 限制backoff指数的最大值                             |

round_robin：轮询，如果有三个sink配置在组中，会依次调度三个sink

random：随机调度

#### 例子

```
a1.sources=r1
a1.channels=c1 c2
a1.sinks=k1 k2 k3 k4  #先配置好sinks再为它们分组

# 分配sink组信息
a1.sinkgroups = g1 g2
a1.sinkgroups.g1.sinks= k1 k2 
a1.sinkgroups.g1.processor.type= load_balance
a1.sinkgroups.g1.processor.selector = round_robin

a1.sinkgroups.g2.sinks= k3 k4
a1.sinkgroups.g2.processor.type= failover
a1.sinkgroups.g2.processor.priority.k3 = 5
a1.sinkgroups.g2.processor.priority.k4 = 10
a1.sinkgroups.g2.processor.maxpenalty = 10000

# 再配置你的各个sink就行了
```

