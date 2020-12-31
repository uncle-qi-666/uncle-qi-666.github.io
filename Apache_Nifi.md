# **Apache NiFi **

### introduction

 		Apache NiFi is a dataflow system based on the concepts of flow-based programming. It supports powerful and scalable directed graphs of data routing, transformation, and system mediation logic. 

content database flowfile provenance 四个repository的作用

核心概念

| nifi术语（基于flowfile编程） | FBP Term（基于流程编程） | 描述                                                         |
| ---------------------------- | ------------------------ | ------------------------------------------------------------ |
| flowfile                     | information packet       | FlowFile表示在系统中移动的每个对象，对于每个FlowFile，NIFI都会记录它一个属性键值对和0个或多个字节内容(FlowFile有attribute和content) |
| processor                    | black box                | 实际上是处理器起主要作用。在eip术语中，处理器就是不同系统间的数据路由，数据转换或者数据中介的组合。处理器可以访问给定FlowFile的属性及其内容。处理器可以对给定工作单元中的零或多个流文件进行操作，并提交该工作或回滚该工作 |
| connection                   | bounded buffer           | Connections用来连接处理器。它们充当队列并允许各种进程以不同的速率进行交互。这些队列可以动态地对进行优先级排序，并且可以在负载上设置上限，从而启用背压 |
| controller                   | scheduler                | 流控制器维护流程如何连接，并管理和分配所有流程使用的线程。流控制器充当代理，促进处理器之间流文件的交换 |
| process group                | subnet                   | 进程组里是一组特定的流程和连接，可以通过输入端口接收数据并通过输出端口发送数据，这样我们在进程组里简单地组合组件，就可以得到一个全新功能的组件 |

四个Repository作用：

- flowfile repository

  ​		对于给定一个流中正在活动的FlowFile,FlowFile Repository就是NIFI保持跟踪这个FlowFIle状态的地方。FlowFile Repository的实现是可插拔的（多种选择，可配置，甚至可以自己实现），默认实现是使用Write-Ahead Log技术(简单普及下，WAL的核心思想是：在数据写入库之前，先写入到日志.再将日志记录变更到存储器中。)写到指定磁盘目录。

- content repository

  ​		Content Repository是给定FlowFile的实际内容字节存储的地方。Content Repository的实现是可插拔的。默认方法是一种相当简单的机制，它将数据块存储在文件系统中。可以指定多个文件系统存储位置，以便获得不同的物理分区以减少任何单个卷上的争用。(所以环境最佳实践时可配置多个目录，挂载不同磁盘，提高IO)。

- database  repository

  ​		由两部分组成 :  nifi-user-keys.h2.db、nifi-flow-audit.h2.db。user-keys中记录了登录过nifi的用户。flow-audit中记录了所有在nifi webUI中所做过的配置更改变动，这些信息可在菜单里的`Flow Configuration History`中查看。

- provenance repository

  ​		Provenance Repository是存储所有事件数据的地方。Provenance Repository的实现是可插拔的，默认实现是使用一个或多个物理磁盘卷。在每个位置内的事件数据都是被索引并可搜索的。

### 1. 集群安装

​		从NiFi 1.0版本开始，NIFI集群采用了Zero-Master Clustering模式。NiFi群集中的每个节点对数据执行相同的任务，但每个节点都在不同的数据集上运行。Apache ZooKeeper选择单个节点作为集群协调器，ZooKeeper自动处理故障转移。所有集群节点都会向集群协调器发送心跳报告和状态信息。集群协调器负责断开和连接节点。此外，每个集群都有一个主节点，主节点也是由ZooKeeper选举产生。我们可以通过任何节点的用户界面（UI）与NiFi群集进行交互，并且我们所做的任何更改都将复制到集群中的所有节点上。

下载路径：

```shell
https://nifi.apache.org/download.html
```

### 2. 配置

#### 2.1 nifi.properties

```shell 
# 关闭内置zookeeper
nifi.state.management.embedded.zookeeper.start=false

# 当前 nifi 节点主机名
nifi.web.http.host=10.10.10.24
#nifi.web.http.host=10.10.10.70
#nifi.web.http.host=10.10.10.113

# 是否是集群中的节点，默认值为false。
nifi.cluster.is.node=true
# 设置为当前节点的主机名
nifi.cluster.node.address=10.10.10.24
nifi.cluster.node.protocol.port=8888
#nifi.cluster.node.address=10.10.10.70
#nifi.cluster.node.address=10.10.10.113

# 指定集群中导致流的早期选择所需的节点数
nifi.cluster.flow.election.max.candidates=1

# cluster 负载均衡配置，其他值默认即可
nifi.cluster.load.balance.host=10.10.10.24

#该属性值应填写外部zk组件的实际地址
nifi.zookeeper.connect.string=10.10.10.227:2182,10.10.10.164:2182,10.10.10.20:2182
```

#### 2.2 state-management.xml

```shell 
<!--该属性值应填写外部zk组件的实际地址-->
<property name="Connect String">
	10.10.10.227:2182,10.10.10.164:2182,10.10.10.20:2182
</property>
```

### 3. 启动

```shell
# 在nifi/bin目录下执行
./nifi.sh start  # 启动
./nifi.sh stop   # 停止
./nifi.sh status # 查看状态

# 或者将bin添加到path中，即可随处启动
echo “export NIFI_HOME=/data/app/nifi/" >> /etc/profile
echo "export PATH=$PATH:$NIFI_HOME/bin" >> /etc/profile
```

### 4. 验证

- 手动停止一个节点，等待若干分钟（集群正在选举新的leader），会发现可用节点数减1

![image-20201228211641964](C:\Users\f30001582\AppData\Roaming\Typora\typora-user-images\image-20201228211641964.png)

- nifi集群运行过程中，各节点同时运作，独立处理数据，若某节点中断，数据将停留在该节点上，恢复时继续处理；

### 5. 配置

#### 5.1 Processors 

​		A NiFi Processor is the basic building block for creating an Apache NiFi dataflow. Processors provide an interface through which NiFi provides access to a flowfile, its attributes and its content. Writing your own custom processor provides a way to perform different operations or to transform flowfile content according to specfic needs.



#### 5.2 Controller Services

​		A NiFi Controller Service provides a shared starting point and functionality across Processors, other ControllerServices, and ReportingTasks within a single JVM. Controllers are used to provide shared resources, such as a database, ssl context, or a server connection to an external server and much more.

- 例子：配置StandardSSLContextService，用途连接密匙认证的kafka集群
- 1) 新建processor：ConsumeKafka_1_0

```shell
Security Protocol ： SASL_SSL
SSL Context Service ：选择配置好的controller Service
Kerberos Service Name：kafka    # name与jaas配置中name相同
# 新建两个变量：
sasl.jaas.config：org.apache.kafka.common.security.plain.PlainLoginModule required username="your_name" password="your_pwd";
sasl.mechanism：PLAIN
```

- 2) 在processor所在group内，新建controller service：StandardSSLContextService

```shell
Truststore Filename：/path/to/client.truststore.jks
Truststore Password:pwd_of_truststore
Truststore Type:JKS
TLS Protocol:TLS
```

- 3) 还需要在nifi/conf/bootstrap.conf内配置

```shell 
java.arg.16=-Djava.security.auth.login.config=/path/to/jass-client.config
```

​		其中，jass-client.config中填写如下内容：

```shell 
KafkaClient {
  org.apache.kafka.common.security.plain.PlainLoginModule required
  username="your_name"
  password="your_pwd";
};
```

### 6.web UI

![img](https://nifichina.gitee.io/image/general/nifi-toolbar-components.png)

#### 6.1 关键工具栏介绍

**Components Toolbar**：构造数据流的主要操作面板（左至右依次介绍，各按钮选中拖至中间画布即可）

processor（处理器）：添加模块；nifi中workflow的基础单元，目前有上百个，也可自己封装。分为两类：功能性模块如putHDFS、ExecuteSQL、ConvertCSVToAvro，通用模块，可以再里面支持第三方操作如ExecuteScript、ExecuteProcess等；

input-port：数据流传入点，并非整体数据流的起点，它是作为组与组之间的数据连接的传入点和输出点；

out-put：与上面相反；

process-group：添加组，组相当于系统中的文件夹，作用是使数据流的各个部分看起来整齐清晰；

remote process-group：添加远程组；

template：拉取已有的文件，配置好完整的workflow后，可储存在本地为xml文件，nifi支持本地的template上传，即模版；

#### 6.2 操作小技巧：

- 删除processor时，若前后连接的管道queued里有数据则删除不掉，需先清空管道才能删除（右键管道，选择Empty queue）；
- 进入group后，空白处右键，选择Leave group即可退出群组；
- 若想多选processor，需按住Shift，再选；
- 每个processor都**必须设置流向**，若没有去处，需选中关闭即可；



附：官方文档（建议看英文版，更易理解概念）

- https://nifi.apache.org/docs/nifi-docs/（英文版）

- https://nifichina.github.io/（中文版）