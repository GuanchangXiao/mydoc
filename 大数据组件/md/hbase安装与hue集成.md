# Hbase2.1.7分布式安装

## 1. 安装前准备

​	hadoop安装<2.8.5>可参考兼容表格。

​	zookeeper安装<3.4.13>。

​	下载hbase安装包 

[下载地址]: http://mirror.bit.edu.cn/apache/hbase/

## 2. 解压安装配置环境变量

解压tar  -zxvf 

配置环境变量

export HBASE_HOME=/opt/dev/hbase-2.1.7
export PATH=$PATH:$HBASE_HOME/bin

## 3.修改配置文件

修改配置文件  `hbase-2.1.7/conf/hbase-env.sh`

设置JAVA_HOME

设置HBASE_MANAGES_ZK = false  //默认true，默认使用自带zk，即单机版

```shell
# The java implementation to use.  Java 1.8+ required.
export JAVA_HOME=/opt/dev/jdk1.8.0_181

# Tell HBase whether it should manage it's own instance of ZooKeeper or not.
export HBASE_MANAGES_ZK=false

```

修改hbase-site.xml

```
<configuration>

  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>master,slave1</value>  //zk管理的机器
    <description>The directory shared by RegionServers.
    </description>
  </property>
 
 <property>
	<name>hbase.rootdir</name>
	<value>hdfs://master:9000/hbase</value>
	<description>The directory shared by RegionServers.
    </description>
 </property>
 <property>
	 <name>hbase.cluster.distributed</name>
	<value>true</value>
	<description>The mode the cluster will be in. Possible values are
      false: standalone and pseudo-distributed setups with managed Zookeeper
      true: fully-distributed with unmanaged Zookeeper Quorum (see hbase-env.sh)
	</description>
 </property>
 <property>
	<name>hbase.master.info.port</name>
	<value>60010</value>
 </property>

</configuration>

```

修改regionservers   

设置slave节点主机名，每行一个

`vi regionservers`

```
slave1   //填写从节点主机名
```

## 4.启动及报错解决

启动  `./start-hbase.sh`

查看进程  jps

```
2371 SecondaryNameNode
3507 HMaster
5432 Jps
4956 ThriftServer
2158 NameNode
1967 QuorumPeerMain
2543 ResourceManager
```

![](C:\Users\Ryan\AppData\Roaming\Typora\typora-user-images\1570705274845.png)



报错：ERROR [main] regionserver.HRegionServer: Failed construction RegionServer java.lang.NoClassDefFoundError: org/apache/htrace/SamplerBuilder

```

```

解决办法:把`/hbase-2.1.7/lib/client-facing-thirdparty`目录下的`htrace-core-3.1.0-incubating.jar` 复制到`/hbase-2.1.7/lib`即可。



## HUE集成Hbase

hue配置连接hbase：

找到hue配置文件目录：

`hue-4.2.0/desktop/conf`

修改配置hue.ini的配置文件

```
[hbase]
  # Comma-separated list of HBase Thrift servers for clusters in the format of '(name|host:port)'.
  # Use full hostname with security.
  # If using Kerberos we assume GSSAPI SASL, not PLAIN.
  hbase_clusters=(Cluster|172.31.40.134:9090)  //必须写IP不然连不上

  # HBase configuration directory, where hbase-site.xml is located.
  hbase_conf_dir=/opt/dev/hbase-2.1.7/conf

  # Hard limit of rows or columns per row fetched before truncating.
  ## truncate_limit = 500

  # 'framed' is used to chunk up responses, which is useful when used in conjunction with the nonblocking server in Thrift.
  # 'buffered' used to be the default of the HBase Thrift Server.
  thrift_transport=buffered

```

hbase 启动thrift  :

`./bin/hbase-daemon.sh start thrift -p 9090`

hue启动 :

找到目录 `/opt/dev/hue-4.2.0/build/env/bin`

提前建nohup.out文件  *不是必须的，可以自行指定日志位置*

 `nohup ./supervisor &`  

