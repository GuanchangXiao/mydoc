# MySQL-MHA集群搭建
## 1.环境说明
|  |   |
| ----- | -----|
|系统版本|centos7
|数据库版本|mysql5.7
|MHA版本|mha0.58-centos
|VIP|192.168.101.220/24


| 角色 | IP  | serverID | 读写权限 |
| ------ | :-----: | :-----: | -----|
| Master | 192.168.101.221/24 | 1| 读写 |
| Slave1(备用master) | 192.168.101.222/24 | 2| 读 |
| Slave2（manager节点） | 192.168.101.223/24 | 3 | 读 |

##2.安装MySQL，配置主从复制

###2.1关闭防火墙
查看防火墙状态：firewall-cmd --state      
停止防火墙：systemctl stop firewalld.service  
禁止防火墙开机启动：systemctl disable firewalld.service  
关闭selinux:打开/etc/selinux/config文件,SELINUX=enforcing改为SELINUX=disabled。 
###2.2.在线安装MySQL5.7
yum install epel*  -y && yum clean all && yum makecache  
rpm -Uvh http://repo.mysql.com/mysql57-community-release-el7.rpm  
yum clean all && yum makecache  
yum install gcc gcc-c++ openssl-devel mysql mysql-server mysql-devel -y  
设置mysql开机自启  
systemctl enable mysqld  
  启动mysql  
	systemctl start mysqld  
  cat /var/log/mysqld.log | grep 'password is generated'	`找到mysql初始密码  `
 
`  登录mysql，设置密码：`  
	mysql> flush privileges;  `#更新权限`  
	set password for root@localhost = password('123456');  
  `如果报错密码策略安全问题，则运行以下命令：`  
	set global validate_password_policy=0;  
    set global validate_password_length=1;  
	`如果是离线安装mysql，可以参考：`[博客地址](https://www.cnblogs.com/Orange42/p/8432185.html)

###2.3配置/etc/my.cnf文件（三个服务器都需要，只有server-id不一样，其他相同）
    Vi /etc/my.cnf
  ```
[client]
	user=root
	password=123456
	[mysqld]
	datadir=/var/lib/mysql
	socket=/var/lib/mysql/mysql.sock
	log-error=/var/log/mysqld.log
	pid-file=/var/run/mysqld/mysqld.pid
	#每个server上不一致，见规划
	server-id = 1
	#read-only=1	#不在配置文件中限定只读，但是要记得在slave上限制只读
	#mysql5.6已上的特性，开启gtid，必须主从全开
	gtid_mode = on	
	enforce_gtid_consistency = 1
	log_slave_updates = 1
	#开启半同步复制  否则自动切换主从的时候会报主键错误
	plugin_load = "rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"
	loose_rpl_semi_sync_master_enabled = 1
	loose_rpl_semi_sync_slave_enabled = 1
	loose_rpl_semi_sync_master_timeout = 5000
	log-bin=mysql-bin
	relay-log = mysql-relay-bin
	replicate-wild-ignore-table=mysql.%
	replicate-wild-ignore-table=test.%
	replicate-wild-ignore-table=information_schema.% 
```
##3.在三个节点上做授权配置（主从复制）
三个节点做以下配置:
```
mysql> grant replication slave on *.* to 'repl_user'@'192.168.101.%' identified by '123456';
mysql> grant all on *.* to 'root'@'192.168.101.%' identified by '123456';
```
只在2个slave节点配置只读
```
mysql> set global read_only=1;
```
在主master上面查看状态：
```
mysql> show master status;
```
注意File和Position的内容，接下来要替换在两个slave节点上做以下操作：
```
mysql> change master to master_host='192.168.101.221',master_user='repl_user',master_password='123456',master_log_file='mysql-bin.000002',master_log_pos=1298;
mysql> start slave;
mysql> show slave status
```
查看slave_IO和slave_SQL是否正常
##4.MHA安装
###4.1三个服务器间配置ssh免密通讯(三台服务器均要设置)
```shell script
ssh-keygen -t rsa
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.101.221
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.101.222
ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.101.223
```
###4.2安装MHA的node软件，三个机器都要装
```shell script
wget http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -ivh epel-release-latest-7.noarch.rpm
yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager -y

```
下载软件（方式任选其一）
```shell script
wget https://qiniu.wsfnk.com/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
wget https://github.com/yoshinorim/mha4mysql-node/releases/download/v0.58/mha4mysql-node-0.58-0.el7.centos.noarch.rpm
rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
```
###4.3 安装mha的manage软件，只在manger节点装（也就是slave2）
两种下载方式：
```shell script
wget https://qiniu.wsfnk.com/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
wget https://github.com/yoshinorim/mha4mysql-manager/releases/download/v0.58/mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
yum install mailx -y //该软件是用来发送邮件的

```
##5.	配置MHA（manager节点）
###5.1 创建目录，存放脚本， 配置全局配置文件
```shell script
mkdir -p /etc/mha/scripts
vi /etc/masterha_default.cnf
```
内容如下：
```shell script
[server default]
user=root
password=123456
ssh_user=root
repl_user=repl_user
repl_password=123456
ping_interval=1
#master_binlog_dir= /var/lib/mysql,/var/log/mysql
secondary_check_script=masterha_secondary_check -s 192.168.101.221 -s 192.168.101.222 -s 192.168.101.223 
master_ip_failover_script="/etc/mha/scripts/master_ip_failover"
master_ip_online_change_script="/etc/mha/scripts/master_ip_online_change"
report_script="/etc/mha/scripts/send_report"
```
###5.2 	配置主配置文件
```shell script
vi /etc/mha/app1.cnf
```
内容如下：
```shell script
[server default]
	manager_workdir=/var/log/mha/app1
	manager_log=/var/log/mha/app1/manager.log
	[server1]
	hostname=192.168.1.57
	candidate_master=1
	master_binlog_dir="/var/lib/mysql"
	#查看方式　find / -name mysql-bin*
	[server2]
	hostname=192.168.1.58
	candidate_master=1
	master_binlog_dir="/var/lib/mysql"
	[server3]
	hostname=192.168.1.56
	master_binlog_dir="/var/lib/mysql"
	#表示没有机会成为master
	no_master=1

```
###5.3配置VIP
为了防止脑裂发生,推荐生产环境采用脚本的方式来管理虚拟 ip,而不是使用 keepalived来完成。
```shell script
vi /etc/mha/scripts/master_ip_failover
```
内容如下：
```shell script
#!/usr/bin/env perl
	use strict;
	use warnings FATAL => 'all';
	use Getopt::Long;
	my (
		$command,	$ssh_user,	$orig_master_host,
		$orig_master_ip,$orig_master_port, $new_master_host, $new_master_ip,$new_master_port
	);
	#定义VIP变量
 #注意这个ens33是你自己网卡信息的名称
	my $vip = '192.168.101.220/24';
	my $key = '1';
	my $ssh_start_vip = "/sbin/ifconfig ens33:$key $vip";
	my $ssh_stop_vip = "/sbin/ifconfig ens33:$key down";
	GetOptions(
		'command=s'		=> \$command,
		'ssh_user=s'		=> \$ssh_user,
		'orig_master_host=s'	=> \$orig_master_host,
		'orig_master_ip=s'	=> \$orig_master_ip,
		'orig_master_port=i'	=> \$orig_master_port,
		'new_master_host=s'	=> \$new_master_host,
		'new_master_ip=s'	=> \$new_master_ip,
		'new_master_port=i'	=> \$new_master_port,
	);
	exit &main();
	sub main {
		print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
		if ( $command eq "stop" || $command eq "stopssh" ) {
			my $exit_code = 1;
			eval {
				print "Disabling the VIP on old master: $orig_master_host \n";
				&stop_vip();
				$exit_code = 0;
			};
			if ($@) {
				warn "Got Error: $@\n";
				exit $exit_code;
			}
			exit $exit_code;
		}

		elsif ( $command eq "start" ) {
		my $exit_code = 10;
		eval {
			print "Enabling the VIP - $vip on the new master - $new_master_host \n";
			&start_vip();
			$exit_code = 0;
		};

		if ($@) {
			warn $@;
			exit $exit_code;
			}
		exit $exit_code;
		}

		elsif ( $command eq "status" ) {
			print "Checking the Status of the script.. OK \n";
			exit 0;
		}
		else {
			&usage();
			exit 1;
		}
	}

	sub start_vip() {
		`ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
	}
	sub stop_vip() {
		return 0 unless ($ssh_user);
		`ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
	}
	sub usage {
		print
		"Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
	}

```
###5.4 配置邮件告警脚本
mail邮件发送程序，需要先配置好发送这信息
```shell script
vi /etc/mail.rc
set from=xingu@163.com
set smtp=smtp.163.com
set smtp-auth-user=xing
	#拿163邮箱来说这个不是密码，而是授权码
set smtp-auth-password=wqwqwq123
set smtp-auth=login

#这是具体的邮件发送脚本
vi /etc/mha/scripts/send_report

```
内容如下：
```shell script
#!/bin/bash
	source /root/.bash_profile
	# 解析变量
	orig_master_host=`echo "$1" | awk -F = '{print $2}'`
	new_master_host=`echo "$2" | awk -F = '{print $2}'`
	new_slave_hosts=`echo "$3" | awk -F = '{print $2}'`
	subject=`echo "$4" | awk -F = '{print $2}'`
	body=`echo "$5" | awk -F = '{print $2}'`
	#定义收件人地址
	email="888118@wsfnk.com"
	tac /var/log/mha/app1/manager.log | sed -n 2p | grep 'successfully' > /dev/null
	if [ $? -eq 0 ]
		then
		messages=`echo -e "MHA $subject 主从切换成功\n master:$orig_master_host --> $new_master_host \n $body \n 当前从库:$new_slave_hosts"` 
		echo "$messages" | mail -s "Mysql 实例宕掉，MHA $subject 切换成功" $email >>/tmp/mailx.log 2>&1 
		else
		messages=`echo -e "MHA $subject 主从切换失败\n master:$orig_master_host --> $new_master_host \n $body" `
		echo "$messages" | mail -s ""Mysql 实例宕掉，MHA $subject 切换失败"" $email >>/tmp/mailx.log 2>&1  
	fi

```
###5.5 	配置编写VIP脚本
```shell script
vi /etc/mha/scripts/master_ip_online_change
```
内容如下：
```shell script
#!/bin/bash
	source /root/.bash_profile
	vip=`echo '192.168.101.220/24'`  #设置VIP
	key=`echo '1'`
	command=`echo "$1" | awk -F = '{print $2}'`
	orig_master_host=`echo "$2" | awk -F = '{print $2}'`
	new_master_host=`echo "$7" | awk -F = '{print $2}'`
	orig_master_ssh_user=`echo "${12}" | awk -F = '{print $2}'`
	new_master_ssh_user=`echo "${13}" | awk -F = '{print $2}'`
	#要求服务的网卡识别名一样，都为ens33(这里是)
	stop_vip=`echo "ssh root@$orig_master_host /usr/sbin/ifconfig ens33:$key down"`
	start_vip=`echo "ssh root@$new_master_host /usr/sbin/ifconfig ens33:$key $vip"`

	if [ $command = 'stop' ]
	  then
	    echo -e "\n\n\n****************************\n"
	    echo -e "Disabled thi VIP - $vip on old master: $orig_master_host \n"
	    $stop_vip
	    if [ $? -eq 0 ]
	      then
		echo "Disabled the VIP successfully"
	      else
		echo "Disabled the VIP failed"
	    fi
	    echo -e "***************************\n\n\n"
	  fi

	if [ $command = 'start' -o $command = 'status' ]
	  then
	    echo -e "\n\n\n*************************\n"
	    echo -e "Enabling the VIP - $vip on new master: $new_master_host \n"
	    $start_vip
	    if [ $? -eq 0 ]
	      then
		echo "Enabled the VIP successfully"
	      else
		echo "Enabled the VIP failed"
	    fi
	    echo -e "***************************\n\n\n"
	fi

```
###5.6.	给脚本赋权可执行
```shell script
chmod +x /etc/mha/scripts/master_ip_failover
chmod +x /etc/mha/scripts/master_ip_online_change
chmod +x /etc/mha/scripts/send_report
```
###5.7.	验证ssh是否成功
```shell script
masterha_check_ssh --conf=/etc/mha/app1.cnf
```
###5.8.	验证主从复制是否正常
```shell script
masterha_check_repl --conf=/etc/mha/app1.cnf
```
##6.	启动MHA（注意：MHA监控脚本切换一次就会退出，需要再次启动）
##6.1 先在master上绑定vip，(只需要在master绑定这一次，以后会自动切换)，注意这里的ens33是网卡名称
```shell script
/usr/sbin/ifconfig ens33:1 192.168.101.220/24
```
###6.2.	然后通过 masterha_manager 启动 MHA 监控(在manager角色上执行)
```shell script
mkdir /var/log/mha/app1 -p
touch /var/log/mha/app1/manager.log
#启动mha监控进程，下面是一条命令
nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &；
#检查MHA的启动状态
tailf /var/log/masterha/app1/manager.log
#如果最后一行是如下，表明启动成功
[info] Ping(SELECT) succeeded, waiting until MySQL doesn’t respond..
#检查集群状态
masterha_check_status --conf=/etc/mha/app1.cnf

```
##7.	切换测试
###第一：要实现自动 Failover,必须先启动 MHA Manager,否则无法自动切换
A、杀掉主库 mysql 进程,模拟主库发生故障,进行自动 failover 操作。  
B、看 MHA 切换日志,了解整个切换过程
```shell script
tailf /var/log/mha/app1.log
```
###第二：从上面的输出可以看出整个 MHA 的切换过程,共包括以下的步骤
1).配置文件检查阶段,这个阶段会检查整个集群配置文件配置  
2).宕机的 master 处理,这个阶段包括虚拟 ip 摘除操作,主机关机操作（由于没有定义power_manager脚本，不会关机）  
3).复制 dead maste 和最新 slave 相差的 relay log,并保存到 MHA Manger 具体的目录下  
4).识别含有最新更新的 slave  
5).应用从 master 保存的二进制日志事件(binlog events)这点信息对于将故障master修复后加入集群很重要  
6).提升一个 slave 为新的 master 进行复制  
7).使其他的 slave 连接新的 master 进行复制  
###第三：切换完成后,关注如下变化
1、vip 自动从原来的 master 切换到新的 master,同时,manager 节点的监控进程自动退出。  
2、在日志目录(/var/log/mha/app1)产生一个 app1.failover.complete 文件  
3、/etc/mha/app1.cnf 配置文件中原来老的 master 配置被删除。
##8.	将故障节点重新加入集群
通常情况下自动切换以后,原 master 可能已经废弃掉;  
待原 master 主机修复后,如果数据完整的情况下,可能想把原来 master 重新作为新主库的 slave;  
这时我们可以借助当时自动切换时刻的 MHA 日志来完成对原 master 的修复。  
(若是下面第二步：出现的是第二种，可以不用借助日志，定位binlog日志点，可以自动定位)

###8.1.修改 manager 配置文件（只针对自动切换的，在线切换不会删除配置）
将如下内容添加到/etc/mha/app1.conf 中
```shell script
[server1]
  hostname=192.168.101.221
  candidate_master=1
  master_binlog_dir="/var/lib/mysql"        

```
###8.2.修复老的 master,然后设置为 slave
```shell script
cat /var/log/mha/app1/manager.log
```
查看信息如下，对应两种修复方式
###8.3.	在老的 master 执行如下命令:(具体执行哪条，根据上面输出来确定，区别是一个有日志的定位，一个是自动定位)
```shell script
mysql> change master to master_host='192.168.101.221,master_user='repl_user',master_password='123456',master_log_file='mysql-bin.0000124',master_log_pos=234;
#或者
mysql> change master to master_host='192.168.101.221,master_user='repl_user',master_password='123456',MASTER_AUTO_POSITION=1
#开启slave：
mysql> start slave;
mysql> show slave status\G;

```
###8.4.	在manager节点上重启监控服务
```shell script
nohup masterha_manager --conf=/etc/mha/app1.cnf --remove_dead_master_conf --ignore_last_failover < /dev/null > /var/log/mha/app1/manager.log 2>&1 &
```