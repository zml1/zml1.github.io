---
title: HBase1.2安装配置
date: 2018-03-20 17:23:42
categories: 
- 大数据
tags:
- HBase 
---

## 安装模式

Hbase分为三种安装模式：单机模式，伪分布式模式，完全分布式模式

- 单机模式：单机模式的安装非常简单，几乎不用对安装文件做什么修改就可以使用。单机模式下，HBase并不使用HDFS，因此将安装文件解压后就几乎可以直接运行。
- 伪分布模式是一个运行在单台机器上的分布式模式。此模式下，HBase所有的守护进程将运行在同一个节点之上，而且需要依赖HDFS，因此在此之前必须保证HDFS已经成功运行。
<!--more-->
- 完全分布式模式是运行在每台机器上，本文采取的就是这种方式。

## 准备

1.选择合适版本
选择 Hadoop 版本对HBase部署很关键。下表显示不同HBase支持的Hadoop版本信息。基于HBase版本，应该选择合适的Hadoop版本。
可以从hbase官方文档查询支持版本，链接：<http://hbase.apache.org/book.html#configuration>在浏览器中搜索 Hadoop version support即可
* "S" = supported
* "X" = not supported
* "NT" = Not tested
    
|  |HBase-1.2.x|HBase-1.3.x|HBase-2.0.x|
|:--|-----------|-----------|-----------|
|Hadoop-2.4.x|S|S|X|
|Hadoop-2.5.x|S|S|X|
|Hadoop-2.6.0|X|X|X|
|Hadoop-2.6.1+|S|S|S|
|Hadoop-2.7.0|X|X|X|
|Hadoop-2.7.1+|S|S|S|
|Hadoop-2.8.0|X|X|X|
|Hadoop-2.8.1|X|X|X|
|Hadoop-3.0.0|NT|NT|NT|

HBase从官网下载即可，官方提供很多镜像地址，如http://mirror.jax.hugeserver.com/apache/hbase/

2.环境准备

注意：*需要提前对集群配置SSH免密登陆，时间同步，host主机名，java环境*

集群机器
    
|IP|HostName|Master|RegionServer|
|:-|--------|------|------------|
|192.168.100.111|mini1|yes|no|
|192.168.100.112|mini2|no|yes|
|192.168.100.113|mini3|no|yes|
    
软件环境
* 系统：CentOS 7.2
* Hadoop：2.6.4
* ZooKeeper：3.4.10
* jdk：1.7

## 下载和解压

***整体安装过程务必使用安装hadoop的用户来安装hbase***

> 首先在一台机器上安装配置好，然后分发到集群中其他机器上

```
shell> wget http://mirror.jax.hugeserver.com/apache/hbase/1.2.6/hbase-1.2.6-bin.tar.gz # 下载
shell> tar -xzvf hbase-1.2.6-bin.tar.gz -C /opt/  # 解压
```
## 配置环境变量

修改环境变量需要root用户，在最下面增加HBASE的家目录HBASE_HOME
```
shell> su - root
shell> echo "HBASE_HOME=/opt/hbase-1.2.6" >> /etc/profile
shell> echo "PATH=$PATH:$HBASE_HOME/bin" >> /etc/profile
shell> echo "export HBASE_HOME PATH" >> /etc/profile
```
使环境变量生效
```
shell> source /etc/profile
```
修改资源限制
HBase 和其他的数据库软件一样会同时打开很多文件。Linux 中默认的ulimit 值是1024，这对HBase 来说太小了。当使用诸如bulkload 这种工具批量导入数据的时候会得到这样的异常信息：java.io.IOException:Too many open files。我们需要改变这个值，注意，这是对操作系统的参数调整，而不是通过HBase 配置文件完成的，我们可以大致估算ulimit 值需要配置为多大。
```
shell> ulimit -n 10240
```

## 修改配置文件

hbase 相关的配置主要包括hbase-env.sh、hbase-site.xml、regionservers三个文件（如regionservers没有可以手动创建），都在 $HBASE_HOME/conf 下面

### 配置hbase-env.sh

```
shell> vim $HBASE_HOME/conf/hbase-env.sh  
```

```
# JDK安装目录  
export JAVA_HOME=/opt/jdk1.7.0_80  
export JAVA_CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar  
# hadoop配置文件的位置  
export HBASE_CLASSPATH=/opt/hadoop-2.6.4/etc/hadoop  
# 如果使用独立安装的zookeeper这个地方就是false  
export HBASE_MANAGES_ZK=false  
```

### 配置hbase-site.xml

```
<configuration>  
        <property>  
                <name>hbase.master.port</name>  #hbase master的端口
                <value>60000</value>
        </property>
        <property>
                <name>hbase.master.info.port</name>  #hbase web管理的端口
                <value>60010</value>
        </property>
        <property>
                <name>hbase.rootdir</name>  #hbase共享目录，持久化hbase数据
                <value>hdfs://mini1:9000/hbase</value>
        </property>
        <property>
                <name>hbase.cluster.distributed</name>  #是否分布式运行，false即为单机
                <value>true</value>
        </property>
        <property>
                <name>hbase.zookeeper.quorum</name>  #zookeeper地址
                <value>mini1:2181,mini2:2181,mini3:2181</value>
        </property>
</configuration>
```

### 配置regionservers

regionservers这个文件内容是regionServer的域名，在1.2.6版本中没有，需要用户在$HBASE_HOME/conf下手动创建

```
shell> touch $HBASE_HOME/conf/regionservers
shell> echo "mini2" >> $HBASE_HOME/conf/regionservers
shell> echo "mini3" >> $HBASE_HOME/conf/regionservers
```

### hadoop的hdfs-site.xml和core-site.xml

将集群hadoop的这两个配置文件复制到hbase的conf目录下

```
shell> cp /opt/hadoop-2.6.4/etc/hadoop/hdfs-site.xml $HBASE_HOME/conf/
shell> cp /opt/hadoop-2.6.4/etc/hadoop/core-site.xml $HBASE_HOME/conf/
```

## 分发到其他机器

```
shell> scp -r /opt/hbase-1.2.6 mini2:/opt
shell> scp -r /opt/hbase-1.2.6 mini3:/opt
```

## 启动
HBase强依赖hadoop和zookeeper，所以务必先确保集群的hadoop和zk都正常启动

启动hadoop(mini1)

```
shell> /opt/hadoop-2.6.4/sbin/start-all.sh
```

启动zookeeper（mini, mini2, mini3分别执行）

```
shell> /opt/zookeeper-3.4.10/bin/zkServer.sh start
```

启动HBase（mini1）

> 一般哪台启动hbase，哪台就是HMaster

```
shell> start-hbase.sh
```

### 查看进程

mini1(master)

```
shell> jps

1254 DataNode  
1550 ResourceManager  
3296 Jps  
1988 QuorumPeerMain  
1387 SecondaryNameNode  
1158 NameNode  
2950 HMaster   # hbase master进程
1679 NodeManager  
```

mini2,mini3(salve)

```
shell> jps

1288 QuorumPeerMain  
1148 NodeManager  
1791 Jps  
1571 HRegionServer  # hbase slave进程
```