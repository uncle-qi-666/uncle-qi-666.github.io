注：在通用配置的基础上进行集群搭建

### 1. 环境变量及hosts配置

#### 1.1 /etc/hosts文件

```shell
10.10.10.227 node-001
10.10.10.164 node-002
10.10.10.20 node-003
10.10.10.24 node-004
10.10.10.70 node-005
10.10.10.113 node-006
10.10.10.213 node-007
```

#### 1.2 /etc/profile文件

```shell
export JAVA_HOME=/path/jdk1.8.0_212
export CLASSPATH=$:CLASSPATH:$JAVA_HOME/lib/
export PATH=$PATH:$JAVA_HOME/bin

export ZK_HOME=/data/app/zookeeper
export PATH=$ZK_HOME/bin:$PATH

export SPARK_HOME=/data/app/spark
export PATH=$PATH:$SPARK_HOME/bin
export PATH=$PATH:$SPARK_HOME/sbin

export HADOOP_HOME=/data/app/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin
```

### 2. 搭建zookeeper集群

​	将节点1的文件发给两个节点

### 3. Hadoop配置文件

#### 2.1 hadoop-env.sh、yarn-env.sh、slaves

​	默认配置即可，从节点的ip需要存进slaves中；

#### 2.2 core-site.xml

```xml
<configuration>
    <!-- 指定hdfs的nameservice为uncleqi -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://uncleqi/</value>
    </property>

    <!-- 指定hadoop临时目录 -->
    <!-- nodemanager 作为容器提供者，会提供容器，运行用户提交的程序，这个过程会产生一些临时的数据，若不配置默认存放在temp目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/data/app/hadoop/tmpLog</value>
    </property>

    <!-- 指定zookeeper地址 -->
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>node-001:2182,node-002:2182,node-003:2182</value>
    </property>
</configuration>
```

#### 2.3 hdfs-site.xml

```xml
<configuration>
    <!-- name service -->
    <!--指定hdfs的nameservice为uncleqi(不能有下划线)，需要和core-site.xml中的保持一致 -->
    <property>
        <name>dfs.nameservices</name>
        <value>uncleqi</value>
    </property>

    <!-- uncleqi下面有两个NameNode，分别是q1，q2 -->
    <property>
        <name>dfs.ha.namenodes.uncleqi</name>
        <!-- 这是逻辑代号，名字随便起 -->
        <value>q1,q2</value>
    </property>

    <!-- q1的RPC通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.uncleqi.q1</name>
        <value>node-001:9000</value>
    </property>
    <!-- q1的http通信地址 -->
    <property>
        <name>dfs.namenode.http-address.uncleqi.q1</name>
        <value>node-001:50070</value>
    </property>

    <!-- q2的RPC通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.uncleqi.q2</name>
        <value>node-002:9000</value>
    </property>
    <!-- q2的http通信地址 -->
    <property>
        <name>dfs.namenode.http-address.uncleqi.q2</name>
        <value>node-002:50070</value>
    </property>

    <!-- 指定NameNode的edits元数据在机器本地磁盘的存放位置：namenode本地磁盘工作目录 -->
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/data/app/hadoop/dfs/name</value>
    </property>

    <property>
        <!-- datanode的工作目录 -->
        <name>dfs.datanode.data.dir</name>
        <value>/data/app/hadoop/dfs/data</value>
    </property>

    <!-- JournalNode -->
    <!-- 指定NameNode的共享edits元数据在JournalNode上的存放位置,目录名最好和名称服务一致，当然这一定是一个虚拟目录 -->
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
        <value>qjournal://node-001:8485;node-002:8485;node-003:8485;node-004:8485;node-005:8485;node-006:8485;node-007:8485;/uncleqi
        </value>
    </property>

    <!-- 指定JournalNode在本地磁盘存放数据的位置 -->
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/data/app/hadoop/dfs/journaldata</value>
    </property>

    <!-- zkfc -->
    <!-- 开启NameNode失败自动切换 -->
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>

    <!-- 配置失败自动切换实现方式: 用什么控制器 -->
    <!-- 大型集群里可能有多对namenode，也就是多对nameservice -->
    <property>
        <name>dfs.client.failover.proxy.provider.uncleqi</name>
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>

    <!-- 配置隔离机制方法，多个机制用换行分割，即每个机制暂用一行：防止脑裂的2种策略--><!-- 模拟脚本 -->
    <property>
        <name>dfs.ha.fencing.methods</name>
        <value>shell(/bin/true)</value>
    </property>

    <!-- 使用sshfence隔离机制时需要ssh免登陆：告知秘钥位置 -->
    <property>
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/home/omm/.ssh/id_rsa</value>
    </property>

    <!-- 配置sshfence隔离机制超时时间，若不配置会一直发脚本，超时后就会自动调用上面配置的脚本 -->
    <property>
        <name>dfs.ha.fencing.ssh.connect-timeout</name>
        <value>10000</value>
    </property>
    
    <!-- 权限设置，允许其他用户操作hdfs -->
    <property>
		<name>dfs.permissions.enabled</name>
		<value>false</value>
	</property>
    
</configuration>
```

#### 2.4 maprd-site.xml

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

#### 2.5 yarn-site.xml

```xml
<configuration>
    <!-- 开启RM高可用 -->
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
    <!-- 指定RM的cluster id：因为会有多个RM -->
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <!-- 随便起 -->
        <value>qishushu</value>
    </property>
    <!-- 指定RM的逻辑名字 -->
    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>
    <!-- 分别指定RM的地址 -->
    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>node-001</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>node-002</value>
    </property>
    <!-- 指定zk集群地址：yarn集群也需要依赖Zookeeper -->
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <value>node-001:2182,node-002:2182,node-003:2182</value>
    </property>
    <property>
        <!-- nodemanager 给mapreduce提供的辅助服务：mapTask把生成的结果以文件的形式写在了机器上，然后mapTask退出，reduceTask要去下载这些数据，靠nodemanager上的web服务器提供下载功能 -->
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>10.10.10.227:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>10.10.10.227:8030</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>10.10.10.227:8035</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address</name>
        <value>10.10.10.227:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>10.10.10.227:8088</value>
    </property>
</configuration>
```

### 4. 启动集群

#### 4.1 启动journalnode

- 在每个节点上分别启动： `hadoop-daemon.sh start journalnode`

#### 4.2 格式化NameNode

- 首先在主NameNode节点上格式化，`hdfs namenode -format`，然后启动该节点NameNode，`hadoop-daemon.sh  start namenode`；
- 其次在备用NameNode节点上同步fsimage，`hdfs namenode -bootstrapStandby`；

#### 4.3 格式化ZKFC

- 在主节点上操作即可，`hdfs zkfc -formatZK`;

#### 4.4 启动hdfs

- 启动备用NameNode，`hadoop-daemon.sh  start namenode`；
- 在各节点上启动DataNode， `hadoop-daemon.sh start datanode`；

#### 4.5 启动YARN

- 主备resourcemanager节点上都需要手动启动， `yarn-daemon.sh start resourcemanager`；
- 所有节点上启动nodemanager， `yarn-daemon.sh start nodemanager`；

#### 4.6 启动historyServer

- 仅限主节点操作，`mr-jobhistory-daemon.sh start historyserver`；

### 5. 验证状态

- webUI查看，8088端口是查看YARN集群状态，50070是查看HDFS集群状态，19888是历史记录情况；

- 脚本查看

  ```shell
  # 查看hdfs的各节点状态信息
  hdfs dfsadmin -report 
  
  #切换NameNode(强制切换主备)，前提是目前在用的NameNode是standby状态，否则失败
  hdfs haadmin -transitionToActive --forcemanual q2
  
  #查看NameNode运行状态
  hdfs haadmin -getServiceState q1
  ```

### 6. 疑难杂症

#### 6.1 安装过程的坑

1. 机器需要配置免密，因为`hdfs-site.xm`l文件中配置了免密登录进行操作；

2. 配置文件`hdfs-site.xml`中`hdfs`命名不允许下划线，且和`core-site.xml`保持一致；

3. 在主`NameNode`节点格式化后，要先启动才能在备用`NameNode`节点再同步；

4. 在启动`namenode`时，命令需要小写字母，否则找不到主类；

5. `NameNode`处于`standby`时是无法读取的，因为该节点连接不通，可强制将节点转换为`active，hdfs haadmin -transitionToActive --forcemanual q2`；

6. 若启动一半又停止，再启动又格式化一遍NameNode，此时DataNode保留的是第一次格式化时保存的VERSION信息，与新格式化的不一样，因此需要将所有节点的`dfs/data/current/VERSION`文件删除，重新执行格式化；

7. 启动后若还有配置文件修改，不必重启集群，刷新节点即可，`hdfs dfsadmin  -refreshNodes`；


#### 6.2 hdfs权限问题

​	hdfs的文件夹拥有者不是root，是用户`hdfs`所有（755权限，超级用户），只有hdfs能对文件夹进行写操作，当其他用户想要对文件夹进行写操作时，需要hdfs用户赋权，两种方式：

- 1. 修改配置，为所有用户赋权，在`hdfs-site.xml`中增加：

```xml
    <!-- 权限设置，允许其他用户操作hdfs -->
    <property>
		<name>dfs.permissions.enabled</name>
		<value>false</value>
	</property>
```

- 2. 单独赋权，赋予某个用户对某个文件夹写操作的权限

```shell
	hdfs dfs -chown salmon /octopus/jobdetail
```

