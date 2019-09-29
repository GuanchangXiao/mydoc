# **centos7.6ARM架构安装MySql5.7.22**

mysql下载地址：https://obs.cn-north-4.myhuaweicloud.com/obs-mirror-ftp4/database/mysql-5.7.27-aarch64.tar.gz

1.卸载系统自带的mariaDB

```
rpm -qa | grep mariadb     #查看具体的mariadb信息
rpm -e --nodeps mariadbname  #强制卸载，把mariadbname 换成上面查出的文件名字
```

2.添加mysql用户和用户组，用于隔离mysql进程

```
[root@master ~]#  groupadd -r mysql && useradd -r -g mysql -s /sbin/nologin -M mysql
```

3.安装依赖库

```
[root@master ~]# yum install -y libaio*
```

4.下载解压mysql

```
[root@master ~]# wget https://obs.cn-north-4.myhuaweicloud.com/obs-mirror-ftp4/database/mysql-5.7.27-aarch64.tar.gz
[root@master ~]# tar -zxf mysql-5.7.27-aarch64.tar.gz
```

5.配置mysql

```
[root@master ~]# mv mysql-5.7.27-aarch64 /usr/local/mysql
[root@master ~]# mkdir -p /usr/local/mysql/logs
[root@master ~]# cd /usr/local/mysql/logs
[root@master logs]# touch mysql-error.log
[root@master ~]# chown -R mysql:mysql /usr/local/mysql
[root@master ~]# ln -sf /usr/local/mysql/my.cnf /etc/my.cnf
[root@master ~]# cp -rf /usr/local/mysql/extra/lib* /usr/lib64/
[root@master ~]# mv /usr/lib64/libstdc++.so.6 /usr/lib64/libstdc++.so.6.old
[root@master ~]# ln -s /usr/lib64/libstdc++.so.6.0.24 /usr/lib64/libstdc++.so.6
```

6.设置开机自启

```
[root@master ~]# cp -rf /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
[root@master ~]# chmod +x /etc/init.d/mysqld
[root@master ~]# systemctl enable mysqld
```

7.添加环境变量

```
[root@master ~]# vim /etc/profile
export MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin

[root@master ~]# source /etc/profile
```

8.启动mysql，可以在/etc/my.cnf的[mysqld]下面加入一行skip-grant-tables.用于免密登录。设置完密码后注释这个内容即可。

```
[root@master ~]# vim /etc/my.cnf
[mysqld]
skip-grant-tables
[root@master ~]# service mysqld start
[root@master ~]# mysql -uroot -p
Enter password: 		#直接回车不用输入密码。
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 860
Server version: 5.7.27-log Source distribution

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql> flush privileges;  #更新权限
Query OK, 0 rows affected (0.00 sec)
mysql> set password for root@localhost = password('123456'); 
Query OK, 0 rows affected, 1 warning (0.00 sec)
mysql>flush privileges; #更新权限
mysql>quit; #退出

```

至此，mysql搭建完毕！