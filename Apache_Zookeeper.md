### introduction

​		ZooKeeper通常被用于实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。

- 分布式锁 ： 通过创建唯一节点获得分布式锁，当获得锁的一方执行完相关代码或者是挂掉之后就释放锁；
- 命名服务 ：可以通过 ZooKeeper 的顺序节点生成全局唯一 ID；
- 数据发布/订阅 ：通过 Watcher 机制 可以很方便地实现数据发布/订阅。当你将数据发布到 ZooKeeper 被监听的节点上，其他机器可通过监听 ZooKeeper 上节点的变化来实现配置的动态更新；

### 1. 集群安装

```shell
# 下载地址
https://zookeeper.apache.org/releases.html
```

### 2. 配置

#### 2.1 `/etc/hosts`文件

```shell
10.10.10.227 node-001
10.10.10.164 node-002
10.10.10.20 node-003
```

#### 2.2 `/etc/profile`文件

```shell
# path of Java
export JAVA_HOME=/data/app/jdk1.8.0_212
export CLASSPATH=$:CLASSPATH:$JAVA_HOME/lib/
export PATH=$PATH:$JAVA_HOME/bin
# path of zookeeper
export ZK_HOME=/data/app/zookeeper
export PATH=$ZK_HOME/bin:$PATH

## 配置完 记得
source /etc/profile
```

#### 2.3 创建myid文件

在每个节点的`$ZK_HOME`下执行：

```shell
mkdir data
mkdir datalog

# 并在data内创建myid文件，输入对应节点的编号1/2/3；
echo 1 > myid
```

#### 2.4 `zoo.cfg`文件

```shell
# 若上述hosts文件未配置，下面填IP同样效果
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/data/app/zookeeper/data
dataLogDir=/data/app/zookeeper/datalog
clientPort=2182
server.1=node-001:2888:3888
server.2=node-002:2888:3888
server.3=node-003:2888:3888
# server.1、server.2、server.3为zk集群中三个节点的信息，定义格式为hostname:port1:port2，其中port1是节点间通信使用的端口，port2是节点选举使用的端口，需确保三台主机的这两个端口都是互通的
```

#### 2.5 参数说明

```shell
tickTime：这个时间是作为 Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。
initLimit：这个配置项是用来配置 Zookeeper 接受客户端（这里所说的客户端不是用户连接 Zookeeper 服务器的客户端，而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。当已经超过 10个心跳的时间（也就是 tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 10*2000=20 秒
syncLimit：这个配置项标识 Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是 5*2000=10秒
dataDir：顾名思义就是 Zookeeper 保存数据的目录，默认情况下，Zookeeper 将写数据的日志文件也保存在这个目录里。
clientPort：这个端口就是客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。
server.A=B：C：D：其中 A 是一个数字，表示这个是第几号服务器；B 是这个服务器的 ip 地址；C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader，而这个端口就是用来执行选举时服务器相互通信的端口。如果是伪集群的配置方式，由于 B 都是一样，所以不同的 Zookeeper 实例通信端口号不能一样，所以要给它们分配不同的端口号。
```

### 3. 启动验证

- 在每个节点上，分别启动zookeeper，执行：`zkServer.sh start`；
- 启动完可用`./zkServer.sh status`查看状态，follower和leader两种角色；

![image-20201231143332251](C:\Users\f30001582\AppData\Roaming\Typora\typora-user-images\image-20201231143332251.png)



### 4. 登录zookeeper

- `zkCli.sh` 登录到zookeeper

