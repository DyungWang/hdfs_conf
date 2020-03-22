# hdfs 高可用配置

## 下载并配置hadoop的环境变量
建议新建一个用户用于单独管理hadoop，便于管理环境变量和权限避免混乱
```sh
adduser hadoop # 添加用户名
passwd hadoop # 修改用户密码
```

切换到hadoop用户下，下载需要版本的hadoop，这里下载的是2.9.2
```sh
wget https://mirror.bit.edu.cn/apache/hadoop/common/hadoop-2.9.2/hadoop-2.9.2.tar.gz
```

解压一个到下载的安装包
```sh
tar -xzf hadoop-2.9.2.tar.gz
```

建议建立一个软连接到解压的安装包上，这样在切换hadoop版本时只需要切换软链的对象即可
```sh
ln -s hadoop hadoop-2.9.2
```

设置环境变量，在 ~/.bash_profile 文件中添加如下几句即可
```sh
export HADOOP_HOME=~/hadoop  # 这里修改为hadoop的安装路径即可
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin # 这里将运行hadoop时需要的脚本加入path中
```

修改 hadoop/etc/hadoop/hadoop-env.sh 中的 export JAVA_HOME 修改为
```sh
export JAVA_HOME=$(readlink -f /usr/bin/java | sed "s:bin/java::")
```

## 配置各个机器之间的ssh免密登录
因为hdfs的启动集群脚本会通过ssh的方式去管理其他的机器，各个机器之间能够免密登录能够方便操作。在每台机器上执行
```sh
ssh-keygen 
```
再将~/.ssh/id_rsa.pub复制到其他机器的authorized_keys中，即可完成免密登录。注意这里可能需要在不同的机器之间来回登录一次
让各个机器都能正确加入到known_hosts文件中，这个在搭建高可用环境时是一个小坑。

## 配置并启动高可用集群
配置每个机器上的hadoop/etc/hadoop/目录下的core-site.xml和hdfs-site.xml，参考本项目下的ha目录中的两个文件配置即可。

高可用环境依赖zookeeper，这里不描述zookeeper的搭建。

启动journalnode
```sh
hadoop-daemon.sh start journalnode
```

在其中一个节点上初始化namenode
```sh
hdfs namenode -format
```

在其他节点上将namenode初始化为standby
```sh
hdfs namenode -bootstrapStandby
```

初始化zk
```sh
hdfs zkfc -formatZK
```

启动，因为这个脚本会ssh到其他机器上进行操作，所以只需要在其中一台机器上操作即可
```sh
start-dfs.sh
```

停止
```sh
stop-dfs.sh
```

## 检查是否安装完毕
在 hadoop/logs 路径下会有 dataNode, nameNode, journalNode的启动日志，通过
```sh
tail -f xxx.log
```
可以查看集群启动过程中是否遇到问题。确认集群启动没有问题以后可以通过
```sh
hdfs haadmin -clusterid <cluster> -getServiceState <nameNode>
```
查看启动的各个namenode的状态。确认namenode启动无问题以后可以通过
```sh
hdfs dfs -fs hdfs://<NameNodeAddr> -ls /
```
检查是hdfs是否正常工作

