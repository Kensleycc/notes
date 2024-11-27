# 架构描述
![kafkacluster](./picture/kafka/cluster.png)
- Producer：生产者，消息入口
- Broker：kafka 实例，可以有多个，可以位于不同的服务器
- Consumer：消费者，消息出口
- Consumer Group：一组消费者，可以同时消费一个 `Topic` 中的不同 `Partition`
- Topic：消息主题，每个 Broker 可以创建多个 Topic
- Partition：同一个 Topic 可以分片为多个 Partition，用于 loadBalance
- Replication：同一个 Partition 可以在多个 Broker 上存在副本，用于高可用
- Leader/Follower：Partition Replication 的主从角色，只有主副本可以提供读写
___
# 消息推送模式
## 推拉模式
![pushpull](./picture/kafka/pushpull.png)
- 基于拉取或者轮询的消息传输模型
- **优点：**
  - 消费者自行控制信息消费速率
- **缺点：**
  - 无法通知消费者有无信息可消费，需要维护一个监听线程不断拉取

## 发布/订阅
![subscribe](./picture/kafka/subscribe.png)
- 基于消息推送
- 优点：
  - 消费者被动接收，不需要监听是否有待消费消息
- 缺点：
  - 推送的速度成为问题，消费者消费能力不一致，很难统一一个兼顾性能和承受能力的值。
___
# 消息流程
![msgProcedure](./picture/kafka/producerSend.png)

## 分区器、分区策略
- 指定消息发送的partition
- 分区策略
  - 轮询：既没有指定partition，也没有设置key
  - hash取模：设置了key，将key按哈希取模
  - 手动指定：指定partition
  - 自定义分区器