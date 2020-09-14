# hadoop

## 安装

### 准备

#### 安装jkd,mysql

#### 修改主机名hostname 文件，hosts配置

hosts 文件

```shell
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.199.160  hadoop-alone

```

hostname文件 修改主机名

```
hadoop-alone
```

设置本机免密登陆

```shell
ssh-keygen -t rsa
cd .ssh/
cat id_rsa.pub >> authorized_keys
```

### 下载hadoop 2进制文件

```shell
wget https://mirrors.bfsu.edu.cn/apache/hadoop/common/hadoop-2.10.0/hadoop-2.10.0.tar.gz --no-check-certificate
tar -zxvf hadoop-2.10.0.tar.gz -c /usr/local/
cd /usr/local/
mv hadoop-2.10.0/ hadoop
```

### 修改环境变量

 vi /etc/profile

```
export JAVA_HOME=/usr/local/jdk
export HADOOP_HOME=/usr/local/hadoop
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:

```

 source /etc/profile



### 检查环境

```shell
mkdir /usr/test/hadoop/input -p
cp /usr/local/hadoop/*.txt /usr/test/hadoop/input/
hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/sources/hadoop-mapreduce-examples-2.10.0-sources.jar org.apache.hadoop.examples.WordCount /usr/test/hadoop/input/ /usr/test/hadoop/output
cat /usr/test/hadoop/output/part-r-00000
```

有文件生成表示安装OK



## 配置单机伪分布式hadoop

### 修改配置文件

- 编辑core-site.xml

  ```shell
  vim /usr/local/hadoop/etc/hadoop/core-site.xml
  ```

  ```xml
  <configuration>
      <property>
          <name>hadoop.tmp.dir</name>
          <value>/usr/data/hadoop/tmp</value>
          <description>Abase for other temporary dir</description>
      </property>
      <property>
          <name>fs.defaultFS</name>
          <value>hdfs://hadoop-alone:9000</value>
      </property>
  </configuration>
  ```

  

- 编辑hdfs-site.xml

```shell
vim /usr/local/hadoop/etc/hadoop/hdfs-site.xml
```

```xml
<configuration>
    <property>
<!--        保存的副本数量-->
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
<!--        保存文件名称-->
        <name>dfs.namenode.name.dir</name>
        <value>file:///usr/data/hadoop/dfs/name</value>
    </property>
    <property>
<!--        保存具体数据-->
        <name>dfs.datanode.data.dir</name>
        <value>file:///usr/data/hadoop/dfs/data</value>
    </property>
    <property>
<!--        通过浏览器访问-->
        <name>dfs.namenode.http-address</name>
        <value>hadoop-alone:50070</value>
    </property>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop-alone:50090</value>
    </property>
    <property>
<!--        hdfs 操作权限 false表示不验证-->
        <name>dfs.permissions</name>
        <value>false</value>
    </property>
</configuration>
```



- 编辑yarn-site.xml

```shell
vim /usr/local/hadoop/etc/hadoop/yarn-site.xml
```

```xml
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hadoop-alone</value>
    </property>
    <property>
        <name>yarn.namenodemanager.aux-services</name>
        <value>mapreduce-shuffle</value>
    </property>
</configuration>

```

### 提前创建路径

```shell
mkdir -p /usr/data/hadoop/dfs/name
mkdir -p /usr/data/hadoop/dfs/data
mkdir -p /usr/data/hadoop/dfs/tmp
```

### 执行

#####  格式化一个新的分布式文件系统 

```shell
hadoop namenode -format
```

#####  启动Hadoop守护进程 

```shell
./usr/local/hadoop/etc/hadoop/start-all.sh
```

##### hadoop 相关日志

Hadoop守护进程的日志写入到 ${HADOOP_LOG_DIR} 目录 (默认是 ${HADOOP_HOME}/logs) 

##### 浏览器访问

关闭防火墙， selinux

```shell
systemctl stop firewalld
setenforce 0
```

## hadoop 进程分析

start-all.sh 启动了hadoop 相关进程 分为2类：

#### DFS 相关进程（存储数据）

NameNode、SecondaryNameNode 、DataNode

- NameNode：  管理节点（进程）用户读取数据从NameNode节点读，而NameNode节点数据是存储在内存中所以需要大内存
- SecondaryNameNode：管理节点（进程）
- DataNode：执行(可以理解为干活的节点)	所有的hdfs数据保存在DataNode节点 需要大硬盘			

#### YARN相关进程（MR数据分析）

ResourceManager、NodeManager

- ResourceManager：管理
- NodeManager： 执行

### 手动启动hadoop相关进程

start-all.sh 启动了所有hadoop 相关进程实际开发中需要手动启动相关的进程

命令路径：

```
/usr/local/hadoop/sbin
```

| 命令                                     | 描述 |
| ---------------------------------------- | ---- |
| hadoop-daemon.sh start namenode          |      |
| hadoop-daemon.sh stop namenode           |      |
| hadoop-daemon.sh start secondarynamenode |      |
| hadoop-daemon.sh stop secondarynamenode  |      |
| hadoop-daemon.sh start datanode          |      |
| hadoop-daemon.sh stop datanode           |      |
| yarn-daemon.sh start resourcemanager     |      |
| yarn-daemon.sh stop resourcemanager      |      |
| yarn-daemon.sh start nodemanager         |      |
| yarn-daemon.sh stop nodemanager          |      |



### 自动化脚本

- 自动化启动dfs：start-dfs.sh
- 自动化停止dfs：stop-dfs.sh
- 自动化启动yarn：start-yarn.sh
- 自动化停止yarn：stop-yarn.sh



## HDFS 执行分析

### hdfs 读取文件过程

- 1.初始化FileSystem，然后客户端(client)用FileSystem的open()函数打开文件
- 2.FileSystem用RPC调用元数据节点，得到文件的数据块信息，对于每一个数据块，元数据节点返回保存数据块的数据节点的地址。
- 3.FileSystem返回FSDataInputStream给客户端，用来读取数据，客户端调用stream的read()函数开始读取数据。
- 4.DFSInputStream连接保存此文件第一个数据块的最近的数据节点，data从数据节点读到客户端(client)
- 5.当此数据块读取完毕时，DFSInputStream关闭和此数据节点的连接，然后连接此文件下一个数据块的最近的数据节点。
- 6.当客户端读取完毕数据的时候，调用FSDataInputStream的close函数。
- 7.在读取数据的过程中，如果客户端在与数据节点通信出现错误，则尝试连接包含此数据块的下一个数据节点。

 ![img](C:\Users\tangjinxing\Desktop\hadoop.assets\11096896-f642dca06447aa05.webp) 

### hdfs 写入文件过程

- 1.初始化FileSystem，客户端调用create()来创建文件
- 2.FileSystem用RPC调用元数据节点，在文件系统的命名空间中创建一个新的文件，元数据节点首先确定文件原来不存在，并且客户端有创建文件的权限，然后创建新文件。
- 3.FileSystem返回DFSOutputStream，客户端用于写数据，客户端开始写入数据。
- 4.DFSOutputStream将数据分成块，写入data queue。data queue由Data Streamer读取，并通知元数据节点分配数据节点，用来存储数据块(每块默认复制3块)。分配的数据节点放在一个pipeline里。Data Streamer将数据块写入pipeline中的第一个数据节点。第一个数据节点将数据块发送给第二个数据节点。第二个数据节点将数据发送给第三个数据节点。
- 5.DFSOutputStream为发出去的数据块保存了ack queue，等待pipeline中的数据节点告知数据已经写入成功。
- 6.当客户端结束写入数据，则调用stream的close函数。此操作将所有的数据块写入pipeline中的数据节点，并等待ack queue返回成功。最后通知元数据节点写入完毕。
- 7.如果数据节点在写入的过程中失败，关闭pipeline，将ack queue中的数据块放入data queue的开始，当前的数据块在已经写入的数据节点中被元数据节点赋予新的标示，则错误节点重启后能够察觉其数据块是过时的，会被删除。失败的数据节点从pipeline中移除，另外的数据块则写入pipeline中的另外两个数据节点。元数据节点则被通知此数据块是复制块数不足，将来会再创建第三份备份。

## 相关名词

| **FS**  |      |
| ------- | ---- |
| **DFS** |      |
|         |      |
|         |      |
|         |      |
|         |      |

























