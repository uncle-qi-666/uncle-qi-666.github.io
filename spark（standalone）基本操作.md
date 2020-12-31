### SPARK（standalone model）基本操作

#### 1、环境准备

安装通用工具

#### 2、安装

2.1配置

1）/etc/hosts文件

```shell
10.10.10.227 node-001
10.10.10.164 node-002
10.10.10.20 node-003
```

2）/etc/profile文件

```shell
export SPARK_HOME=/data/app/spark
export PATH=$PATH:$SPARK_HOME/bin
export PATH=$PATH:$SPARK_HOME/sbin
```

​     配置完，`source /etc/profile`

3)  $SPARK_HOME/conf/spark-env.sh文件

```shell
export JAVA_HOME=/data/app/jdk
export SPARK_MASTER_HOST=node-001

# 以下两个端口号为默认值，若与其他应用冲突，可设置为其他值
export SPARK_MASTER_WEBUI_PORT=8080  
export SPARK_WORKER_WEBUI_PORT=8081
```

4)  $SPARK_HOME/conf/spark-defaults.conf文件（增加历史记录）

```shell
spark.master                     spark://10.10.10.227:7077
spark.eventLog.enabled           true
spark.eventLog.dir               /data/app/spark/logs/eventLog
spark.driver.memory              5g
```

5）$SPARK_HOME/conf/slaves文件（仅master节点配置即可）

```shell
node-001
node-002
node-003
```

#### 3、启动

`$SPARK_HOME/sbin`下操作（粗体为常用启动项）：

- `sbin/start-master.sh` - Starts a master instance on the machine the script is executed on.
- `sbin/start-slaves.sh` - Starts a worker instance on each machine specified in the `conf/slaves` file.
- `sbin/start-slave.sh` - Starts a worker instance on the machine the script is executed on.
- `sbin/start-all.sh` - **Starts both a master and a number of workers as described above.**
- `sbin/stop-master.sh` - Stops the master that was started via the `sbin/start-master.sh` script.
- `sbin/stop-slave.sh` - Stops all worker instances on the machine the script is executed on.
- `sbin/stop-slaves.sh` - Stops all worker instances on the machines specified in the `conf/slaves` file.
- `sbin/stop-all.sh` - **Stops both the master and the workers as described above.**

​        注：master节点也可以同时充当worker节点使用，只需要在服务器用start-slave.sh启动就行，该脚本启动方式为`./start-slave.sh spark://10.10.10.227:7077`

​        启动后验证，8080是监控master的UI port，而8081是监控worker的UI port，且workers的页面可跳转到master上，前提是两者通信port为7077.

打开http://lmaster:8080：

![image-20201218174219940](C:\Users\f30001582\AppData\Roaming\Typora\typora-user-images\image-20201218174219940.png)



#### 4、提交作业

​	  两种提交方式：`spark-shell`与`spark-submit`

​	  区别：`spark-shell`是交互式的，下图为其运行脚本，本质也是调用`spark-submit`，只是设置个应用程序名`Spark shell`，主要调试使用，重点看`spark-submit`

```shell
function main() {
  if $cygwin; then
    stty -icanon min 1 -echo > /dev/null 2>&1
    export SPARK_SUBMIT_OPTS="$SPARK_SUBMIT_OPTS -Djline.terminal=unix"
    "${SPARK_HOME}"/bin/spark-submit --class org.apache.spark.repl.Main --name "Spark shell" "$@"
    stty icanon echo > /dev/null 2>&1
  else
    export SPARK_SUBMIT_OPTS
    "${SPARK_HOME}"/bin/spark-submit --class org.apache.spark.repl.Main --name "Spark shell" "$@"
  fi
}
```