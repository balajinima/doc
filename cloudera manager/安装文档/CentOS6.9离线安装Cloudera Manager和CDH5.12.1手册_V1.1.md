


#安装 Cloudera Manager、CDH 手册 #

>verson 1.1



####一、关于CDH和Cloudera Manager

----------

CDH (Cloudera's Distribution, including Apache Hadoop)，是Hadoop众多分支中的一种，由Cloudera维护，基于稳定版本的Apache Hadoop构建，并集成了很多补丁，可直接用于生产环境。

Cloudera Manager则是为了便于在集群中进行Hadoop等大数据处理相关的服务安装和监控管理的组件，对集群中主机、Hadoop、Hive、Spark等服务的安装配置管理做了极大简化。

####二、系统环境

----------

	操作系统：CentOS release 6.9 (Final) x64
	Cloudera Manager：5.12.1
	CDH: 5.12.1
	

####三、安装包准备

----------

##### 1.Cloudera Manager仓库镜像包下载地址：

https://archive.cloudera.com/cm5/repo-as-tarball/5.12.1/cm5.12.1-centos6.tar.gz

##### 2.CDH parcel安装包地址：

https://archive.cloudera.com/cdh5/parcels/5/CDH-5.12.1-1.cdh5.12.1.p0.3-el6.parcel

https://archive.cloudera.com/cdh5/parcels/5/CDH-5.12.1-1.cdh5.12.1.p0.3-el6.parcel.sha1

https://archive.cloudera.com/cdh5/parcels/5/manifest.json


<font face="微软雅黑" color="red">注意：
通过 Cloudera Manager 安装parcel时sha1格式的文件需要提前修改为sha。</font>
 
##### 3.mariaDB 离线安装包下载地址：

http://yum.mariadb.org/10.1/centos6-amd64/rpms/

	下载对应的安装包：
	galera-25.3.19-1.rhel6.el6.x86_64.rpm
	jemalloc-3.6.0-1.el6.x86_64.rpm
	MariaDB-10.1.22-centos6-x86_64-client.rpm
	MariaDB-10.1.22-centos6-x86_64-common.rpm
	MariaDB-10.1.22-centos6-x86_64-compat.rpm
	MariaDB-10.1.22-centos6-x86_64-devel.rpm
	MariaDB-10.1.22-centos6-x86_64-server.rpm


##### 4.java8 安装包下载地址：

http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

#### 四、准备工作 

----------
##### 1.更新系统（所有节点）

	yum -y update

##### 2.网络配置（所有节点）

修改hostname：

	vi /etc/sysconfig/network
编辑

	NETWORKING=yes
	HOSTNAME= hadoop22.test.com

通过重启验证生效。

修改ip与主机名的对应关系

	vi /etc/hosts,
编辑

	192.168.2.22 hadoop22.test.com
	192.168.2.23 hadoop23.test.com
	192.168.2.24 hadoop24.test.com
	192.168.2.25 hadoop25.test.com
	192.168.2.26 hadoop26.test.com

注意：这里需要将每台机器的ip及主机名对应关系都写进去，本机的也要写进去，否则启动Agent的时候会提示hostname解析错误。


##### 3.配置公钥认证(用于免密登录)

在管理节点（hadoop22.test.com）上执行

	ssh-keygen -t rsa

一路回车，生成无密码的密钥对。

将公钥添加到认证文件中：

	cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

设置authorized_keys的访问权限：

	chmod 600 ~/.ssh/authorized_keys

scp文件到所有受管节点（  192.168.2.23~26）的~/.ssh目录：

	scp ~/.ssh/authorized_keys root@hadoop23.test.com:~/.ssh/

测试：在管理节点上ssh  hadoop23.test.com，正常情况下，不需要密码就能直接登陆进去了。

<font face="微软雅黑" color="red">注意：
.ssh目录访问权限是700，如果收管理节点不存在目录时需要建立此目录，authorized_keys的文件访问权限是600</font>

##### 4.关闭防火墙和SELinux（所有节点）

注意：需要在所有的节点上执行，因为涉及到的端口太多了，临时关闭防火墙是为了安装起来更方便，安装完毕后可以根据需要设置防火墙策略，保证集群安全。

关闭防火墙：

	service iptables stop （临时关闭）  
	chkconfig iptables off （重启后生效）
关闭SELINUX（实际安装过程中发现没有关闭也是可以的，不知道会不会有问题，还需进一步进行验证）:

	setenforce 0 （临时生效）  
	修改 /etc/selinux/config 下的 SELINUX=disabled （重启后永久生效）

##### 5.配置NTP服务

按照Cloudera 的官方建议，所有的CDH节点和Cloudea Manager节点都需要启动ntpd服务。要不然会报如下错误： 

1）此角色的主机的运行状况为不良。 以下运行状况测试不良： 时钟偏差. 

2）The host’s NTP service is not synchronized to any remote server.

在配置之前，先使用ntpdate手动同步一下时间，免得本机与对时中心时间差距太大，使得ntpd不能正常同步。这里选用us.pool.ntp.org作为对时中心,

	ntpdate us.pool.ntp.org

yum安装ntp(所有节点):

	yum -y install ntp

启动服务，执行如下命令：

	service ntpd start

设置ntp服务开机自启动：

	chkconfig ntpd on

客户端校验配置

	ntpq -p查询上级时间服务器

	ntpstat 查询状态


##### 6.优化虚拟内存需求率(所有节点)
1)检查虚拟内存需求率

	cat /proc/sys/vm/swappiness

显示如下：

	60

2)临时降低虚拟内存需求率

	sysctl vm.swappiness=0

3)永久降低虚拟内存需求率

	使用命令 vi /etc/sysctl.conf 增加

	vm.swappiness = 0

并运行如下命令使生效

	sysctl -p

##### 7.解决透明大页面问题(所有节点)

1)检查透明大页面问题

cat /sys/kernel/mm/transparent_hugepage/defrag

如果显示为：

	[always] madvise never

2)临时关闭透明大页面问题

	echo never > /sys/kernel/mm/transparent_hugepage/defrag

确认配置生效：

	cat /sys/kernel/mm/transparent_hugepage/defrag

应该显示为：

	always madvise [never]

3)配置开机自动生效

	使用命令 vi /etc/rc.local,加入如下内容

	echo never > /sys/kernel/mm/transparent_hugepage/defrag


##### 8.安装Oracle的Java（主节点安装，其他节点卸载）

CentOS，自带OpenJdk，不过运行CDH5需要使用Oracle的Jdk，需要Java 7的支持。

卸载自带的OpenJdk，使用 `rpm -qa | grep java` 查询java相关的包，使用 `rpm -e --nodeps 包名` 卸载。或者使用 `yum remove java` 卸载

将jdk-8u144-linux-x64.tar.gz  解压到目录 /usr/java/

	tar -xvf jdk-8u144-linux-x64.tar.gz

配置java环境变量：修改profile	`vi /etc/profile`
	
	export JAVA_HOME=/usr/java/jdk1.8.0_144
	export JRE_HOME=${JAVA_HOME}/jre
	export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
	export PATH=${JAVA_HOME}/bin:$PATH

立即生效 `source /etc/profile`


##### 9.安装配置MariaDB（管理节点）

a. 确保/var/lib/mysql目录有足够大的磁盘空间

b. 卸载自带的mysql。

	yum remove mysql

c. 安装MariaDB ，进入到MariaDB的rpm安装包目录下执行
	
	yum localinstall *.rpm

d. 配置my.conf

	vi /etc/my.cnf	
用以下内容替换

	[mysqld]
	character-set-server = utf8
	transaction-isolation = READ-COMMITTED
	# Disabling symbolic-links is recommended to prevent assorted security risks;
	# to do so, uncomment this line:
	# symbolic-links = 0
	
	key_buffer = 16M
	key_buffer_size = 32M
	max_allowed_packet = 32M
	thread_stack = 256K
	thread_cache_size = 64
	query_cache_limit = 8M
	query_cache_size = 64M
	query_cache_type = 1
	
	max_connections = 550
	#expire_logs_days = 10
	#max_binlog_size = 100M
	
	#log_bin should be on a disk with enough free space. Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your system
	#and chown the specified folder to the mysql user.
	log_bin=/var/lib/mysql/mysql_binary_log
	
	binlog_format = mixed
	
	read_buffer_size = 2M
	read_rnd_buffer_size = 16M
	sort_buffer_size = 8M
	join_buffer_size = 8M
	
	# InnoDB settings
	innodb_file_per_table = 1
	innodb_flush_log_at_trx_commit  = 2
	innodb_log_buffer_size = 64M
	innodb_buffer_pool_size = 4G
	innodb_thread_concurrency = 8
	innodb_flush_method = O_DIRECT
	innodb_log_file_size = 512M
	
	[mysqld_safe]
	log-error=/var/log/mysqld.log
	pid-file=/var/run/mysqld/mysqld.pid
	
	[client]
	default-character-set = utf8
	[mysql]
	default-character-set = utf8

e. 启动MariaDB

	service mysql start

f. 查看MariaDB版本

	mysql --version
输出
	
	mysql  Ver 15.1 Distrib 10.1.21-MariaDB, for Linux (x86_64) using readline 5.1

g. 设置开机启动

	chkconfig mysql on
h. 初始化数据库

	$ sudo /usr/bin/mysql_secure_installation
	[...]
	Enter current password for root (enter for none):
	OK, successfully used password, moving on...
	[...]
	Set root password? [Y/n] y
	New password:
	Re-enter new password:
	Remove anonymous users? [Y/n] Y
	[...]
	Disallow root login remotely? [Y/n] N
	[...]
	Remove test database and access to it [Y/n] Y
	[...]
	Reload privilege tables now? [Y/n] Y
	All done!


i. 使用`mysql -uroot -p`进入mysql命令行，创建数据库和用户：
	

	create database hive DEFAULT CHARACTER SET utf8;
	grant all on hive.* TO 'hive'@'%' IDENTIFIED BY 'hive';
	
	create database hue DEFAULT CHARACTER SET utf8;
	grant all on hue.* TO 'hue'@'%' IDENTIFIED BY 'hue';
	
	create database oozie DEFAULT CHARACTER SET utf8;
	grant all on oozie.* TO 'oozie'@'%' IDENTIFIED BY 'oozie';	

刷新 `flush privileges;`


##### 10.安装mysql JDBC 驱动（管理节点）

下载mysql JDBC 驱动放到目录 `/usr/share/java/` 并修改名为mysql-connector-java.jar 
下载地址
[http://www.mysql.com/downloads/connector/j/5.1.html.](http://www.mysql.com/downloads/connector/j/5.1.html.)


#### 五、安装配置 Cloudera Manager（管理节点）

----------

##### 1.建立Cloudera Manager安装文件自定义存储库

a.安装httpd服务器

查询一下是否已经安装了apache 
	
	rpm -qa httpd   
如果还没有则进行安装
 
	yum -y install httpd 

启动apache

	service httpd start
开机自启动

	chkconfig httpd on

b.将Cloudera Manager仓库镜像包cm5.12.1-centos6.tar.gz

解压到`/var/www/html/cm`目录，文件目录结构如下

![](image\install\filedir1.png)




##### 2.通过rpm安装包本地安装 Cloudera Manager

到目录 `/var/www/html/cm/5/RPMS/x86_64`
	
	 yum --nogpgcheck localinstall cloudera-manager-daemons-5.*.rpm cloudera-manager-server-5.*.rpm enterprise-debuginfo-5.*.rpm


##### 3.Parcel和csd格式文件上传

上传下列文件到Parcel包的存放路径： `/opt/cloudera/parcel-repo/`

		CDH-5.12.1-1.cdh5.12.1.p0.3-el6.parcel            
		CDH-5.12.1-1.cdh5.12.1.p0.3-el6.parcel.sha              
		manifest.json                                       


最后目录结构如下：

![](image\install\filedir2.png)

##### 4.配置 Cloudera Manager Server 数据库

使用命令scm_prepare_database.sh创建Cloudera Manager Server数据库配置文件

命令格式如下
	
	/usr/share/cmf/schema/scm_prepare_database.sh database-type [options] database-name username password
如：

	/usr/share/cmf/schema/scm_prepare_database.sh mysql -hlocalhost -uroot -p123456 --scm-host localhost scm scm scm
执行完成后生成数据库配置文件`/etc/cloudera-scm-server/db.properties`

	# Auto-generated by scm_prepare_database.sh on Tue Feb 28 18:23:16 CST 2017
	#
	# For information describing how to configure the Cloudera Manager Server
	# to connect to databases, see the "Cloudera Manager Installation Guide."
	#
	com.cloudera.cmf.db.type=mysql
	com.cloudera.cmf.db.host=localhost
	com.cloudera.cmf.db.name=scm
	com.cloudera.cmf.db.user=scm
	com.cloudera.cmf.db.password=scm
	com.cloudera.cmf.db.setupType=EXTERNAL

参考链接：
[http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_installing_configuring_dbs.html](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_installing_configuring_dbs.html)

##### 5.启动Cloudera Manager Server

	service cloudera-scm-server start

等待大概两分钟，访问 http://192.168.2.22:7180/  进入管理端
(登陆名:admin 密码:admin)

##### 6.配置

---------- 
![](image/install/cm01.png)

----------
![](image/install/cm02.png)

----------
![](image/install/cm03.png)

----------
![](image/install/cm04.png)

----------
![](image/install/cm05.png)

----------
![](image/install/cm06.png)

----------
![](image/install/cm07.png)

----------
![](image/install/cm08.png)

----------
![](image/install/cm09.png)

----------
![](image/install/cm10.png)

----------
![](image/install/cm11.png)

----------
![](image/install/cm12.png)

----------
![](image/install/cm13.png)

----------
![](image/install/cm14.png)

----------
![](image/install/cm15.png)

----------
![](image/install/cm16.png)

----------
![](image/install/cm17.png)

----------
![](image/install/cm18.png)

----------
![](image/install/cm19.png)

----------
![](image/install/cm20.png)

----------
![](image/install/cm21.png)

----------
![](image/install/cm22.png)


#### 六、修改impala参数

1、时区问题：

默认impala配置不是中国的时区，所以在用from_unixtime的时候，有误差。
解决方案：impala启动时加  -use_local_tz_for_unix_timestamp_conversions=true

在cm 里面  `impala->配置->impala Daemon  ->Impala Daemon 命令行参数高级配置代码段（安全阀）`   
加 
	
	 -use_local_tz_for_unix_timestamp_conversions=true


#### 七、配置sqoop（所有数据节点）

1. 下载oracle驱动包 ojdbc6.jar 到 /var/lib/sqoop/ 目录

	下载地址
	http://www.oracle.com/technetwork/database/enterprise-edition/jdbc-112010-090769.html. 

2. 解决Oracle jdbc的Connection Reset Errors 问题

	在cm 里面  `YARN (MR2 Included)->配置->Map 任务 Java 选项库` 增加 
	
	 	-Djava.security.egd=file:/dev/../dev/urandom
	如图：

	![](image/install/conf01.png)
	

3. Sqoop测试
		
		sqoop eval --connect jdbc:oracle:thin:@192.168.x.x:1521:orcl --username xxx --password xxx \
		--query "select sysdate from dual"


#### 八、常见问题
	
	问题1：在主节点初始化 CM5的数据库
	报错：ld-linux.so.2   bad ELF interpreter
	解决：安装 glibc 和 glibc.i686
	 
	问题2：安装主机时报错
	报错：ProtocolError: <ProtocolError for 127.0.0.1/RPC2: 401 Unauthorized>
	解决：$> ps -ef | grep supervisord
	$> kill -9 <processID>
	/opt/cm-5.6.0/etc/init.d/cloudera-scm-agent restart
	 
	问题3：server启动时，日志提示端口被占用。
	解决：关闭java进程。
	 
	问题4：web安装，当前管理的主机显示都是本地地址
	解决：注释/etc/hosts 的loaclhost ，在检查agent日志的报错。
	重启所有agent
	重启server
	 
	问题5：web数据库设置，登入被拒绝
	解决：grant all privileges on *.* to 'hive'@'cdh1' identified by '123456' with grant option;
	FLUSH privileges;
	 
	问题6：web安装时，群集设置 HDFS格式失败
	解决：删除原有的/dfs
	 
	问题7：web安装时，群集设置HDFS 创建/tmp失败
	解决：ntp一定启动服务器，不能光用命令同步。（这个好像不是问题的所在，但是ntp服务必须要启动的）
	还出现，再重试试试。
	 
	问题8:web管理页面提示时间偏差
	解决：检查ntpdc -c loopinfo
	Name or service not known
	vim /etc/hosts
	添加 本机IP对应localhost 地址
	
	问题9
	报错	JDBC driver cannot be found. Unable to find the JDBC database jar on host
	解决
	下载jar包： 
	MySQL-connector-Java-5.1.27.jar
	重命名：
	mv mysql-connector-java-5.1.27.jar mysql-connector-java.jar
	移动：
	mv mysql-connector-java.jar  /usr/share/java/

	问题10
	登录用户共同私钥
	私钥文件id_rsa所在位置
	$ cd ~/.ssh/
	将此文件下载后，在cm在线安装界面上传即可。

	问题11
	添加zookeeper实例错误
	当有一台机器正在跑zookeeper的时候，再添加其他的，就会报错如下
	Starting these new ZooKeeper Servers may cause the existing ZooKeeper Datastore to be lost. Try again after restarting any 
	existing ZooKeeper Servers with outdated configurations. If you do not want to preserve the existing Datastore, you can start 
	each ZooKeeper Server from its respective Status page.
	将正在运行的zookeeper实例停止，然后再三台一起启动即可。


#### 九、非root用户方案

1.所有节点添加普通用户（在本例中使用hadoop）

	useradd -u 1050 hadoop

2.让普通用户获得sudo执行操作权限

编辑sudoers文件 `vi /etc/sudoers`
    
	在
	# %wheel        ALL=(ALL)       NOPASSWD: ALL
	下增加
	%hadoop        ALL=(ALL)       NOPASSWD: ALL

允许用户组hadoop里面的用户执行sudo命令,并且在执行的时候不输入密码.

3.ssh免密登录

同准备步骤中的免密登录配置方式一致，用户hadoop用户生成秘钥，拷贝到其他机器的hadoop用户名目录，注意目录权限。



#### 参考 ####

>[http://blog.csdn.net/ymh198816/article/details/52423200](http://blog.csdn.net/ymh198816/article/details/52423200 "Cloudera简介和安装部署概述")
>
> [http://www.cnblogs.com/jasondan/p/4011153.html?utm_source=tuicool&utm_medium=referral](http://www.cnblogs.com/jasondan/p/4011153.html?utm_source=tuicool&utm_medium=referral "离线安装Cloudera Manager 5和CDH5(最新版5.1.3) 完全教程")
> 
> [http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_b.html](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_b.html)
> 
> [https://mariadb.com/kb/en/mariadb/yum/](https://mariadb.com/kb/en/mariadb/yum/ "Installing MariaDB with yum")
> 
> [http://www.aboutyun.com/thread-9006-1-1.html](http://www.aboutyun.com/thread-9006-1-1.html)



####卸载cloudera参考
>[http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_uninstall_cm.html](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_uninstall_cm.html)
