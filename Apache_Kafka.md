### introduction

​		Kafka combines three key capabilities so you can implement your use cases for event streaming end-to-end with a single battle-tested solution:

- To publish (write) and subscribe to (read) streams of events, including continuous import/export of your data from other systems；
- To store streams of events durably and reliably for as long as you want；
- To process streams of events as they occur or retrospectively；

### 1. 集群部署

#### 1.1 下载

```shell
https://kafka.apache.org/downloads
```

#### 1.2 配置

##### 1.2.1 `/etc/hosts`

```shell
10.10.10.001 node-001
10.10.10.002 node-002
10.10.10.003 node-003
```

##### 1.2.2 `/etc/profile`

```shell 
# path of kafka
export KAFKA_HOME=/data/app/kafka
export PATH=$KAFKA_HOME/bin:$PATH

## 配置完 记得
source /etc/profile
```

##### 1.2.3 `/server.properties`

注：现在kafka目录下创建数据与日志目录：`mkdir -p data logs`

```shell
advertised.listeners=PLAINTEXT://10.10.10.001:9092
log.dirs=/data/app/kafka/logs
# 每个brokerid不同，另外两个节点分别为1/2
broker.id=0
# 允许删除topic
delete.topic.enable=true
# 数据保存时长，单位小时
log.retention.hours=168

# zookeeper地址
zookeeper.connect=10.10.10.227:2182,10.10.10.164:2182,10.10.10.20:2182
# timeout in ms for connecting to zookeeper 
zookeeper.connection.timeout.ms=6000
```

#### 1.3 启动

- 三节点上操作`kafka-server-start.sh -daemon config/server.properties`；

### 2. 常用命令

​		若在kafka集群中执行，localhost换为任意broker即可；命令后加 --help   即可查看操作说明；

#### 2.1 增删改查

注：从kafka2.2版本开始，zookeeper参数换为bootstrap-server

```shell
# 创建topic
./kafka-topics.sh --zookeeper local:2181 --create --topic uncleqi --partitions 16  --replication-factor 2

# 查看所有topic列表
./kafka-topics.sh --zookeeper localhost:2181 --list

# 查看指定topic的配置信息，如topic name、partition、replication、Leader（当前起作用的brokerID）、Isr（当前kafka集群中可用的brokerID列表，单节点只有一个）
/kafka-topics.sh --zookeeper localhost:2181 --describe --topic uncleqi

# 生产数据，执行后输入想要生产的消息即可
./kafka-console-producer.sh --broker-list localhost:9092 --topic uncleqi
# 消费数据, 不加from参数会默认从最新的开始消费
./kafka-console-consumer.sh  --bootstrap-server localhost:9092  --topic uncleqi --from-beginning

# 或指定offset，须跟上partition 
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic uncleqi --offset latest --partition 0 --max-messages 2

# 删除topic
./kafka-run-class.sh kafka.admin.DeleteTopicCommand --zookeeper localhost:2181 --topic uncleqi
```

#### 2.2 压测工具

```shell
./kafka-producer-perf-test.sh --topic test_kafka_perf1 --num-records 100000000 --record-size 100 --producer-props bootstrap.servers=10.155.104.164:9092 batch.size=10000 --throughput -1

# --num-records 总共需要发送的消息数
# --record-size 每个记录的字节数
# --throughput 每秒发送的记录数
# batch.size  每次批量发送消息的数量
```

#### 2.3  MirrorMaker

注：很实用且酷炫的工具，用于**kafka跨集群数据迁移**

```shell
./kafka-mirror-maker.sh --new.consumer --consumer.config /opt/octopus-pro/mirror-consumer.properties --num.streams 32 --producer.config /opt/octopus-pro/mirror-producer.properties --whitelist 'octopus-salmon-taskdetail' &

###注意事项
# 1）whitelist和blacklist支持正则表达式。比如需要包含两个topic可以这样写，--whitelist 'A|B' or --whitelist 'A,B' ，或者想迁移所有topic可以这样写 --whitelist '*'；
# 2）注意在迁移之前需要在目标集群中创建好相同topic以及相同partition数量；
# 3）建议将MirrorMaker部署在目标集群内，即使发生网络故障，源数据也不会被消费；
# 4）对于sasl认证的kafka集群，目前的配置中密码是明文写入的，后续可以实现sasl.client.callback.handler.class来从外部读取加密的密码
```

经验证，也可实现多实例部署(一定要保证实例均使用**相同的groupID**)，架构如下:

![image-20201231154327446](C:\Users\f30001582\AppData\Roaming\Typora\typora-user-images\image-20201231154327446.png)

### 3. 用户认证

kafka1.10支持四种协议类型的访问：

| 协议类型       | 说明                          | 备注                 |
| -------------- | ----------------------------- | -------------------- |
| PLAINTEXT      | 无认证的铭文访问              |                      |
| SASL_PLAINTEXT | 支持kerberos认证的铭文访问    |                      |
| SSL            | 支持无认证的SSL加密访问       | ssl.mode.enable=true |
| SASL_SSL       | 支持kerberos认证的SSL加密访问 | ssl.mode.enable=true |

选择sasl鉴权机制时，生产端和消费端，哪端需要配哪端。配置如下：

```shell
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="[USERNAME]" password="[PASSWORD]";
ssl.endpoint.identification.algorithm=
ssl.truststore.location=[jsk_path]
ssl.truststore.password=[jks_password]
```

- kafka认证官方教程：

  https://nifi.apache.org/docs/nifi-docs/components/org.apache.nifi/nifi-kafka-0-11-nar/1.7.1/org.apache.nifi.processors.kafka.pubsub.ConsumeKafkaRecord_0_11/additionalDetails.html

