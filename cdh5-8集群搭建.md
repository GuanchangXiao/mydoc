#1.集群规划
    由于资源有限，采用的是一台master，一台slave的搭建方式。
    master的IP地址为：192.168.101.222/24
    slave的IP地址为：192.168.101.111/24
    mysql使用的版本为5.7.25，jdk版本为jdk1.8.0_151，
    mysql驱动包为8.0.13，cloudera-manage包为centos7-5.15.0
    cdh包为5.8.0-el7.
    系统版本为centos7
   [cdh下载地址](http://archive.cloudera.com/cdh5/parcels/5.8.0/)
   
   ![当前资源清单](https://raw.githubusercontent.com/GuanchangXiao/mydoc/master/pictures/cdh/0001.png)
#2.安装JDK
   安装jdk，自选版本，所有的节点都要安装jdk。安装前先卸载系统自带的openjdk。  
   查看当前jdk信息
   ```
   rpm -qa | grep java
   ```
   然后用以下命令进行删除
   ```
   rpm -e --nodeps jdkname
   ```
   建立java目录
   ```
   mkdir -p /usr/local/lib/java
   ```
   解压jdk安装包，配置环境变量  
   ```shell script
   vim /etc/profile

   export JAVA_HOME=/usr/local/lib/java/jdk1.8.0_151
   export PATH=$JAVA_HOME/bin:$PATH
   export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
```
   退出编辑并保存，更新profile  
   ```shell script
   source /etc/profile
```
   查看jdk是否安装成功,看到当前版本信息则成功。
   ```shell script
   java -version
```
#3.修改hostname
   每个节点修改/etc/hosts文件，内容如下：
   ```shell script
   vim /etc/hosts

   192.168.101.222 master
   192.168.101.111 slave
```
   使用hostname命令查看当前节点的名字是否正确，用hostname设置主机名字，在master，slave上面分别执行
   ```shell script
    hostnamectl set-hostname master
    hostnamectl set-hostname slave
```
#4.同步时间
   因为有外网，所以直接两个节点都指向同一个时间服务器，如果没有，slave可以指向master。
   ```shell script
    ntpdate cn.pool.ntp.org
```
#5.关闭防火墙和selinux
   查看防火墙状态
   ```shell script
    firewall-cmd --state
```
   停止防火墙
   ```shell script
    systemctl stop firewalld
```
   禁止开机启动防火墙
   ```shell script
    systemctl disable firewalld
```
   关闭selinux，重启生效
   ```shell script
    sed -i 's/SELINUX=.*/SELINUX=disabled/' /etc/selinux/config  #重启机器生效
```
#6.配置ssh免密登录
   每个节点都要进行如下操作。  
   生成公钥
   ```shell script
    ssh-keygen -t rsa #根据提示敲回车即可，共计三个回车。
```
   推送认证信息
   ```shell script
    ssh-copy-id 192.168.101.111
    ssh-copy-id 192.168.101.222
```
   配置完成后，可以用ssh master/slave 测试配置是否成功。
#7.安装mysql
   在主节点master上安装mysql，可选用离线和在线安装两种方式，当前采用离线安装，参照博客[mysql离线安装](https://www.cnblogs.com/Orange42/p/8432185.html)  
   解压mysql安装包
   ```shell script
    mkdir -p /root/mysql
    tar -xvf mysql-5.7.25-1.el7.x86_64.rpm-bundle.tar -C /root/mysql
    cd /root/mysql
    rpm -ivh mysql-community-common-5.7.25-1.el7.x86_64.rpm
    rpm -ivh mysql-community-libs-5.7.25-1.el7.x86_64.rpm
    rpm -ivh mysql-community-devel-5.7.25-1.el7.x86_64.rpm
    rpm -ivh mysql-community-libs-compat-5.7.25-1.el7.x86_64.rpm
    rpm -ivh mysql-community-client-5.7.25-1.el7.x86_64.rpm
    rpm -ivh mysql-community-server-5.7.25-1.el7.x86_64.rpm
```  
   修改配置文件my.cnf，在[mysqld]增加内容如下：
   ```shell script
    mkdir -p /data/mysql
    vim /etc/my.cnf

    default-storage-engine = innodb
    innodb_file_per_table
    collation-server = utf8_general_ci
    init-connect = 'SET NAMES utf8'
    character-set-server = utf8
    datadir=/data/mysql #修改mysql的数据目录
```
   初始化数据库
   ```shell script
    /usr/bin/mysql_install_db  --basedir=/usr/share/mysql/ --datadir=/data/mysql/ --user=mysql  >>/dev/null
```
   启动mysql，设置开机自启  
   ```shell script
    service mysqld start
    chkconfig mysqld on
```
   在my.cnf文件中加入一行“skip-grant-tables”,提示输入密码直接回车，可以免密登录  
   ```shell script
    mysql -uroot -p
    flush privileges;
    mysql角色赋权

    GRANT ALL PRIVILEGES ON *.* TO 'scm'@"%" IDENTIFIED BY "scm" with grant option;
    GRANT ALL PRIVILEGES ON *.* TO 'scm'@"master" IDENTIFIED BY "scm" with grant option;
    GRANT ALL PRIVILEGES ON *.* TO 'scm'@"localhost" IDENTIFIED BY "scm" with grant option;
    GRANT ALL PRIVILEGES ON *.* TO 'scm'@"127.0.0.1" IDENTIFIED BY "scm" with grant option;
    GRANT ALL PRIVILEGES ON *.* TO 'root'@"%" IDENTIFIED BY "123456" with grant option;
    GRANT ALL PRIVILEGES ON *.* TO 'root'@"master" IDENTIFIED BY "123456" with grant option;
    GRANT ALL PRIVILEGES ON *.* TO 'root'@"localhost" IDENTIFIED BY "123456" with grant option;
    GRANT ALL PRIVILEGES ON *.* TO 'root'@"127.0.0.1" IDENTIFIED BY "123456" with grant option;

    创建cdh所需要的库
    create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
    create database amon DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
    create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
    create database monitor DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
    create database oozie DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
```
   在主节点上传mysql的驱动，最好用8.0以上版本，我之前用的5.1.13不能连接hive  
   ```shell script
    mkdir -p /usr/share/java
    cp mysql-connector-java-8.0.13.jar /usr/share/java/mysql-connector-java.jar
```
#8.安装第三方依赖
   依赖如果没有下载完整会导致集群启动失败。  
   ```shell script
    yum install chkconfig python bind-utils psmisc libxslt zlib sqlite fuse fuse-libs redhat-lsb cyrus-sasl-plain cyrus-sasl-gssapi -y
```
#9.安装cloudera-manager
   解压cm到指定目录，每个节点都要做。  
   ```shell script
    mkdir -p /opt/cloudera-manager
    tar -axvf cloudera-manager-centos7-cm5.15.0_x86_64.tar.gz -C /opt/cloudera-manager
```
   在每个节点创建cloudera-scm用户。  
   ```shell script
    useradd -r -d /opt/cloudera-manager/cm-5.15.0/run/cloudera-scm-server -M -c "Cloudera SCM User" cloudera-scm
```
   在各个slave节点配置agent。 
   ```shell script
    vim /opt/cloudera-manager/cm-5.15.0/etc/cloudera-scm-agent/config.ini
    将server_host改为CMS所在的主机名即master
    server_host=master
```
   在主节点中创建parcel-repo仓库。  
   ```shell script
    mkdir -p  /opt/cloudera/parcel-repo
    chown cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo
    cd /opt/cloudera/parcel-repo
    mv CDH-5.8.0-1.cdh5.8.0.p0.42-el7.parcel.sha1  CDH-5.8.0-1.cdh5.8.0.p0.42-el7.parcel.sha
    cp CDH-5.8.0-1.cdh5.8.0.p0.42-el7.parcel.sha CDH-5.8.0-1.cdh5.8.0.p0.42-el7.parcel manifest.json /opt/cloudera/parcel-repo
    说明：Clouder-Manager将CDHs从主节点的/opt/cloudera/parcel-repo目录中抽取出来，分发解压激活到各个节点的/opt/cloudera/parcels目录中
```
   初始脚本配置数据库scm_prepare_database.sh(在主节点上)
   ```shell script
    /opt/cloudera-manager/cm-5.15.0/share/cmf/schema/scm_prepare_database.sh  mysql -h master -P 3306 -u root -p 123456 --scm-host master scm scm scm
    成功信息如下：
    /**
    JAVA_HOME=/usr/java/jdk1.8.0_45
    Verifying that we can write to /opt/cloudera-manager/cm-5.8.3/etc/cloudera-scm-server
    Creating SCM configuration file in /opt/cloudera-manager/cm-5.8.3/etc/cloudera-scm-server
    Executing:  /usr/java/jdk1.8.0_45/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/Oracle-connector-java.jar:/opt/cloudera-manager/cm-5.8.3/share/cmf/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /opt/cloudera-manager/cm-5.8.3/etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
    [                          main] DbCommandExecutor              INFO  Successfully connected to database.
    All done, your SCM database is configured correctly!
    **/
    参数说明：这个脚本就是用来创建和配置CMS需要的数据库的脚本。各参数是指：
    mysql：数据库用的是mysql，如果安装过程中用的oracle，那么该参数就应该改为oracle。
    -h master：数据库建立在hadoop1主机上面。也就是主节点上面。
    -u root：root身份运行mysql。-p 123456：mysql的root密码是123456。
    --scm-host master：CMS的主机，一般是和mysql安装的主机是在同一个主机上。
    最后三个参数是：数据库名，数据库用户名，数据库密码。
```
#10.启动CM
   首先启动server，不然会报错连接拒绝。 
   在master上面启动server和agent
   ```shell script
    /opt/cloudera-manager/cm-5.15.0/etc/init.d/cloudera-scm-server start #start:启动服务，stop:停止服务，status:查看状态
    /opt/cloudera-manager/cm-5.15.0/etc/init.d/cloudera-scm-agent start #start:启动服务，stop:停止服务，status:查看状态
```
   在各个从节点启动agent  
   ```shell script
    /opt/cloudera-manager/cm-5.15.0/etc/init.d/cloudera-scm-agent start #start:启动服务，stop:停止服务，status:查看状态
```
   浏览器访问server，进行页面安装，访问地址：http://192.168.101.222:7180.
   按照自己的要求一步一步往下安装即可。
#11.可能遇到的问题
   > 查看agent和server的状态时报错，cloudera-scm-server dead but pid file exists  
  
    进入 /opt/cloudera-manager/cm-5.15.0/run/cloudera-scm-agent/ ,删除对应的pid文件，server的pid文件在run目录下  

   > 在初始化数据库脚本报错，root被拒绝访问没有权限
   
    确认scm和root用户的权限是否配置成功  
    
   > 在安装集群，初次启动hdfs报错，/dfs/nn 目录已经存在，xxxx目录无法创建
   
    进入master，查看对应的文件夹的所属用户，如果是root，把文件夹拥有者改为hdfs即可

   > 在安装集群，初次启动oozie时报错，上载sharelib超时问题
   
    选择oozie配置，把超时时间设置大一点，比如1500

   > 启动集群，hue连接hive失败
   
    确认mysql驱动版本是否与mysql版本相符合，修改hive-site.xml中的驱动名称，同时加入连接名称和密码
   ```shell script

```
    
    