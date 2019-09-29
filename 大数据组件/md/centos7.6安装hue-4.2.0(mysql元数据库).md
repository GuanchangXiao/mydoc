# **centos7.6安装hue-4.2.0(mysql元数据库)**

### 1.安装相关依赖

```shell
[root@master ~]# yum install ant asciidoc cyrus-sasl-devel cyrus-sasl-gssapi cyrus-sasl-plain gcc gcc-c++ krb5-devel libffi-devel libxml2-devel libxslt-devel make mysql mysql-devel openldap-devel python-devel sqlite-devel  gmp-devel openssl-devel
```

### 2.安装maven

maven下载地址：http://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.2/binaries/apache-maven-3.6.2-bin.tar.gz

选择安装目录为/opt/bigdata，下载解压maven

```shell
[root@master ~]# wget http://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.6.2/binaries/apache-maven-3.6.2-bin.tar.gz
[root@master ~]# tar -zxf apache-maven-3.6.2-bin.tar.gz -C /opt/bigdata
[root@master ~]# cd /opt/bigdata
[root@master bigdata]# mv apache-maven-3.6.2-bin maven
```

更改配置文件

```shell
[root@master ~]# cd /opt/bigdata/maven/conf
[root@master conf]# vim settings.xml
<mirrors>
     <mirror>
         <id>alimaven</id>
         <mirrorOf>central</mirrorOf>
         <name>aliyun maven</name>
         <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
     </mirror>

     <mirror>
         <id>ui</id>
         <mirrorOf>central</mirrorOf>
         <name>Human Readable Name for this Mirror.</name>
         <url>http://uk.maven.org/maven2/</url>
     </mirror>

     <mirror>
         <id>jboss-public-repository-group</id>
         <mirrorOf>central</mirrorOf>
         <name>JBoss Public Repository Group</name>
         <url>http://repository.jboss.org/nexus/content/groups/public</url>
     </mirror>
 </mirrors>
```

配置maven环境变量

```shell
[root@master ~]# vim /etc/profile
export MAVEN_HOME=/opt/bigdata/maven
export PATH=$PATH:$MAVEN_HOME/bin
[root@master ~]# source /etc/profile          #重新加载profile
```

### 3.安装hue

hue下载地址：http://gethue.com/downloads/releases/4.2.0/hue-4.2.0.tgz

下载解压hue到/opt/bigdata目录下

```shell
[root@master ~]# wget http://gethue.com/downloads/releases/4.2.0/hue-4.2.0.tgz
[root@master ~]# tar -zxf hue-4.2.0.tgz -C /opt/bigdata
```

编译hue

```shell
[root@master ~]# cd /opt/bigdata/hue-4.2.0
[root@master hue-4.2.0]# make apps
```

编译失败则重试

```shell
[root@master hue-4.2.0]# make clean
[root@master hue-4.2.0]# make apps
```

配置hue

```
[root@master hue-4.2.0]# vim desktop/conf/hue.ine
```

#### HUE

```shell
secret_key=
http_host=master
http_port=8888
time_zone=Asia/Shanghai
server_user=hue
server_group=hue
default_user=root
default_hdfs_superuser=root
```

#### database

```shell
engine=mysql
host=192.168.101.250  #数据库所在的ip
port=3306
user=root
password=123456
name=hue   #元数据库名字
```

## hadoop/hdfs

```shell
fs_defaultfs=hdfs://master:9000
webhdfs_url=http://master:50070/webhdfs/v1
hadoop_conf_dir=/opt/bigdata/hadoop-2.8.5/etc/hadoop #hadoop的安装路径
hadoop_hdfs_home=/opt/bigdata/hadoop-2.8.5
hadoop_bin=/opt/bigdata/hadoop-2.8.5/bin
```

#### YARN

```shell
resourcemanager_host=master
resourcemanager_port=8032
hadoop_conf_dir=/opt/bigdata/hadoop-2.8.5/etc/hadoop #hadoop的安装路径
hadoop_hdfs_home=/opt/bigdata/hadoop-2.8.5
hadoop_bin=/opt/bigdata/hadoop-2.8.5/bin
resourcemanager_api_url=http://master:8088
proxy_api_url=http://master:8088
history_server_api_url=http://master:19888
```

#### Hive

```shell
hive_server_host=master  #hive所在的主机ip
hive_server_port=10000
hive_conf_dir=/opt/bigdata/hive-2.3.4/conf
```

#### mysql

```shell
nice_name="My SQL DB"
name=hue  #元数据库名字
engine=mysql
host=192.168.101.250 #数据库所在ip
port=3306
user=root
password=123456
```

### 4.修改hadoop配置文件

修改hdfs-site.xml文件

```shell
[root@master hadoop]# vim hdfs-site.xml
<property>
  <name>dfs.webhdfs.enabled</name>
  <value>true</value>
</property>
```

修改core-site.xml文件

```shell
[root@master hadoop]# vim core-site.xml
<!-- enable WebHDFS in the NameNode and DataNodes -->
<property> 
　　<name>dfs.webhdfs.enabled</name> 
　　<value>true</value> 
</property>
<property>
     <name>hadoop.proxyuser.root.hosts</name>
     <value>*</value>
</property>
<property>
     <name>hadoop.proxyuser.root.groups</name>
     <value>*</value>
</property>
<property>  
    <name>hadoop.proxyuser.httpfs.hosts</name>  
    <value>*</value>  
</property>  
<property>  
    <name>hadoop.proxyuser.httpfs.groups</name>  
    <value>*</value>  
</property> 
```

### 5.启动hue

初始化数据库

```shell
[root@master ~]# cd /opt/bigdata/hue-4.2.0/build/env/bin/
[root@master bin]# ./hue syncdb
[root@master bin]# ./hue migrate
```

启动hue

```
[root@master bin]# ./supervisor
```

