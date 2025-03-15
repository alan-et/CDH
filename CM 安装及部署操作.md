[TOC]
cm agent主机异常Error, CM server guid updated, expected
产生的原因是服务器以前作为集群节点安装过agent服务，再次使用时没有卸载干净。

解决办法：

rm -rf /var/lib/cloudera-scm-agent/cm_guid

service cloudera-scm-agent restart


# 一、CM 概述及架构

## 1. CM概述

cloudera managerment 简称，是由cloudera公司开源 ，对以Hadoop为基础的生态圈框架所组件的集成的分布式的自动化安装部署集群，并且会对安装部署后的集群的资源信息、框架的服务运作状态进行实时监控及报警的平台软件
	CM框架只能安装cdh版本的大数据框架
	Apache版本的Hadoop可以使用ambari框架进行安装及监控工作
	为什么要使用CM工具搭建并管理大数据集群（手动tar包安装） 
	
		

## 2.CDH版本的Hadoop框架的优点 

版本划分清晰

支持Kerberos安全认证

完善的集群，计算资源，网络状态实时监控及报警

方便快捷安全的数据倾斜迁移及滚动升级

官方网站的文档清晰
	支持多种安装方式（Cloudera Manager方式）
		tar包安装 （Apache版本和cdh版本都支持）
		yum安装 （Apache版本和cdh版本都支持）
		rpm安装  （Apache版本和cdh版本都支持） 
Cloudera Manager安装方式

​		

## 3.CM框架技术架构 

![CM架构](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1561444389792&di=3738a64e88973bbf9a9e8ec8bb4d96ec&imgtype=0&src=http%3A%2F%2Fwww.th7.cn%2Fd%2Ffile%2Fp%2F2016%2F12%2F02%2Fc5554df9f9a980c4186c3acd6d1421db.jpg)

- **server** 
  		CM框架的主节点（主服务进程） 
    		通常部署在一台单独的服务器上 
    		与各个agent从节点保持心跳获取从节点发送的所在节点的服务器信息
    		负责集群安装部署文件的分发、安装命令的发布 
    		server负责整个集群的启动和停止命令的发布
           一个server节点可以同时管理和部署多套集群

- **agent**  
  		CM框架的从节点
    		负责所在服务器节点上的大数据框架的安装及后续资源状态监控的具体工作  
    		agent节点又称为主机节点 
    		**CM只能在启动了agent服务进程的主机节点上去安装部署大数据框架及启动服务进程** 


- **database**   
  		CM需要一个数据库的支持存储集群的元数据信息    
    			mysql oracle  postgresql   
- ​			mysql  
  ​	​	​				5.5 -不支持impala的使用
    ​				5.6 -选择  
    ​		需要临时存储监控信息   
- **managerment service**
  		是由一些服务进程组成的一组监控组件 
    		通过这组服务进程可以监控各台agent主机上的资源信息及提供实时报警功能 	
- **web-ui**
  		CM内置了一个基于web的用户交互管理界面，是用户进行组件安装部署，日常操作和维护的主要操作对象
    			安装部署集群 
    			监控到整个集群上服务的资源信息及各个大数据框架的运行状态

# 二、安装概述

## （一）软件环境

### 1.软件配置清单

| 序号 |     软件名称     |        版本         |
| :--: | :--------------: | :-----------------: |
|  1   |     操作系统     |   CentOS 7.2 64位   |
|  2   |       JDK        | jdk-8u112-linux-x64 |
|  3   | Cloudera manager |       5.16.2        |
|  4   |    CHD precel    |       5.16.2        |
|  5   |      MySQL       |         5.7         |

**【扩展：查看服务的硬件信息】**

>基本参数
>
>Physical id 	#相同表示为同一个物理CPU
>Processor 	#逻辑CPU
>Cpu cores 	#CPU核数，内核个数
>Core id 	#内核id号
>Siblings 	#每个物理CPU里面的逻辑CPU个数
>
>-  1 查看cpu型号
>
>```bash
>cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
>```
>
>- 2 查看物理cpu个数
>
>```bash
>cat /proc/cpuinfo | grep "physical id" | sort -u | wc -l
>```
>
>- 3 查看逻辑cpu
>
>```bash
>cat /proc/cpuinfo | grep "processor" | wc -l 
>```
>
>- 查看CPU内核数
>
>  ```bash
>  cat /proc/cpuinfo | grep "cpu cores" | uniq  
>  ```

### 2.所需的软件资源

1. #### JDK环境

   JDK版本：jdk-8u112-linux-x64.tar.gz

   scala版本：scala-2.11.8

   

2. #### Cloudera Manager

   cm版本：5.16.2

   http://archive.cloudera.com/cm5/cm/5/cloudera-manager-centos7-cm5.16.2_x86_64.tar.gz

   cloudera-manager-centos7-cm5.16.2_x86_64.tar.gz

3. #### CDH

   cdh版本：5.16.2

   http://archive.cloudera.com/cdh5/parcels/latest/

**CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel**
**CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha1**
**manifest.json**

4. #### JDBC连接jar包

   版本：5.14.43

   mysql-connector-java-5.1.43.jar

5. #### MySQL包

   版本：5.7.25

   mysql-5.7.25-1.el7.x86_64.rpm-bundle.tar
   

## （二）、配置规划

本次安装共5台服务器，服务器配置及用途如下表所示：

| 序号 | 主机名 | ip地址         | 核/内存/硬盘 | 角色             |
| ---- | ------ | -------------- | ------------ | ---------------- |
| 1    | cdh01  | 172.23.201.191 | 10C/16G/100G | cm-server，mysql |
| 2    | cdh02  | 172.23.201.192 | 10C/16G/100G | cm-agent         |
| 3    | cdh03  | 172.23.201.193 | 10C/16G/100G | cm-agent         |
| 4    | cdh04  | 172.23.201.194 | 10C/16G/100G | cm-agent         |
| 5    | cdh05  | 172.23.201.195 | 10C/16G/100G | cm-agent         |

# 三、安装步骤

# ==提前声明：整个安装过程使用root用户==

## （一）、基本环境

### 1.修改主机名(所有节点)

```bash
#按照规划依次修改，修改完成记得重启服务器
vi /etc/hostname
```

### 2.配置主机映射(所有节点)

```bash
vi /etc/hosts
172.23.201.191 cdh01
172.23.201.192 cdh02
172.23.201.193 cdh03

```

### 3.关闭防火墙和安全子系统(所有节点)

```bash
#关闭防火墙
systemctl stop firewalld.service 
systemctl stop firewalld.service
# 禁止firewall开机启动
systemctl disable firewalld.service

#关闭安全子系统
vi /etc/selinux/config
SELINUX=disabled
```





### 4.配置ssh免密钥登陆(所有节点)

```bash
ssh-keygen

ssh-copy-id hadoop-node1
ssh-copy-id hadoop-node2
ssh-copy-id hadoop-node3
ssh-copy-id hadoop-node4
ssh-copy-id hadoop-node5

# 做完之后必须要做测试
```

**如果不成功，怎么办：**

- 删除用户主目录下的.ssh目录，重新来

  ```shell
  rm -r /root/.ssh
  ```


### 5.修改linxu内核参数(所有节点)

**==5.1和5.2暂时不用做，后边启动cm之后报错再操作==**

5.1 设置swappiness，控制换出运行时内存的相对权重，Cloudera 建议将 swappiness 设置为 10：

```bash
# 查看swappiness 
cat /proc/sys/vm/swappiness 

#永久性修改，执行下面两条命令 
sysctl -w vm.swappiness=10 
echo vm.swappiness = 10 >> /etc/sysctl.conf
```

5.2 关闭透明大页面：

自CentOS6版本开始引入了Transparent Huge Pages(THP)，从CentOS7版本开始，该特性默认就会启用。尽管THP的本意是为提升内存的性能，不过某些数据库厂商还是建议直接关闭THP，否则可能会导致性能出现下降。 

```shell
# 临时关闭（重启机器会变回默认开启状态）： 
echo never > /sys/kernel/mm/transparent_hugepage/defrag 
echo never > /sys/kernel/mm/transparent_hugepage/enabled 
# 永久关闭： 
# 编辑/etc/rc.d/rc.local 
vi /etc/rc.d/rc.local 
# 在文件后添加下面内容: 
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then 
echo never > /sys/kernel/mm/transparent_hugepage/enabled 
fi 
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then 
echo never > /sys/kernel/mm/transparent_hugepage/defrag 
fi 
# 保存退出，然后赋予rc.local文件执行权限： 
chmod +x /etc/rc.d/rc.local 
# 重启系统，以后再检查THP状态，显示状态被禁用了。
```

5.3 修改文件句柄数

```shell
# 查看文件句柄数，显示1024，显然太小 
ulimit -n 
1024 
# 修改系统文件句柄数限制 
vim /etc/security/limits.conf 
# 在文件后加入下面内容： 
* soft nofile 100000 
* hard nofile 100000 
# 修改后需要重启机器。
```

### 6.安装jdk（所有节点）

### ==注意：JDK必须安装在/usr/java目录==

```shell
#1.查询java相关的安装包
rpm -qa | grep java

#2.卸载查询出来的所有的包
rpm -e --nodeps  xxx

#3.解压
tar -zxf jdk-8u112-linux-x64.tar.gz /usr/java

#4.配置环境变量
vi /etc/profile
# 增加一下配置
export JAVA_HOME=/usr/java/jdk1.8.0_171
export PATH=$PATH:$JAVA_HOME/bin

#5.source生效并测试
source /etc/profile

java -version

```

### 7.配置NTP时间同步服务

**基本原理：**

将cdh01节点作为整套集群的时间同步服务器，其他的节点都向cdh01节点同步时间，这样即使cdh01节点不是标准的互联网时间，但是其他节点也只能向cdh01节点同步时间，并以其为统一的集群时间

timedatectl set-timezone Asia/Shanghai
确定三台电脑都是 上海时间

#### 7.1 所有节点安装ntp服务

```bash
yum -y install ntp
```

#### 7.2 在cdh01节点

```shell
vi /etc/ntp.conf 

# 增加，限制只允许172.23.201，也就是集群中其他节点向cdh01节点同步时间
restrict 172.23.201.0 mask 255.255.255.0

# 增加 设置cdh01节点为集群的时间同步服务器
server 127.127.1.0 

# 同时将以下四行进行注释掉，不主动同以下时间服务器同步时间
#server 0.centos.pool.nep.org iburst
#server 1.centos.pool.nep.org iburst
#server 2.centos.pool.nep.org iburst
#server 3.centos.pool.nep.org iburst
```

#### 7.3 在cdh02-cdh05节点

```shell
vi /etc/ntp.conf 
# 增加 设置cdh01节点为集群的时间同步服务器
server 172.23.201.191

# 同时将以下四行进行注释掉，不主动同以下时间服务器同步时间
#server 0.centos.pool.nep.org iburst
#server 1.centos.pool.nep.org iburst
#server 2.centos.pool.nep.org iburst
#server 3.centos.pool.nep.org iburst
```

#### 7.4  所有节点（cdh01-cdh05）

1. 先与网络服务器同步时间

   ```shell
   ntpdate ntp1.aliyun.com
   ```

2. 启动ntp服务，并设置其为开机启动

   ```shell
   systemctl start ntpd.service
   systemctl enable ntpd.service
   ```

#### 7.41 cdh01

移除chronyd(没有就忽略)

```
systemctl stop chronyd
yum remove chrony
```



#### 7.5 测试其它节点能否向cdh01同步时间

```
ntpdate -u cdh01
```

7.6 为了进一步保证，也可以编写定时任务来每10分钟同步一次

```bash
crontab -e

*/10 * * * * /usr/sbin/ntpdate -u hadoop-3
```

常见问题:

```
解决NTP synchronized: no的方法：
1. 首先：执行ntpstat，实时命令查看状态，结果发现所有服务器的状态都是未同步状态“unsynchronised”
2. 停掉ntpd, 执行ntpd -gq重新调整时间后，再启动ntpd:
  systemctl stop ntpd
  sudo ntpd -gq
  systemctl start ntpd
3. 等待五分钟左右，执行timedatectl，查看服务器时间状态，发现NTP synchronized为true
```



### 8. 安装依赖（所有节点）

```shell
yum -y install chkconfig python bind-utils psmisc libxslt zlib sqlite cyrus-sasl-plain cyrus-sasl-gssapi fuse portmap fuse-libs redhat-lsb   mod_ssl unzip
yum install -y psmisc
yum -y install libaio
yum -y install net-tools
yum -y install  perl
yum install -y httpd



######1 下载并安装MySQL官方的 Yum Repository       https://www.cnblogs.com/ianduin/p/7679239.html 按照这个配置
#####[root@localhost ~]# wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
####  使用上面的命令就直接下载了安装用的Yum Repository，大概25KB的样子，然后就可以直接yum安装了。
####[root@localhost ~]# yum -y install mysql57-community-release-el7-10.noarch.rpm
 #### 之后就开始安装MySQL服务器。
####[root@localhost ~]# yum -y install mysql-community-server
 ##### 这步可能会花些时间，安装完成后就会覆盖掉之前的mariadb。




## （二）MySQL安装

### 1.下载

```bash
# wget https://cdn.mysql.com//archives/ mysql-5.7.25-1.el7.x86_64.rpm-bundle.tar
```

### 2.解压

```bash
# tar -xvf mysql-5.7.25-1.el7.x86_64.rpm-bundle.tar
```

> mysql-community-embedded-compat-5.7.25-1.el7.x86_64.rpm
>mysql-5.7.25-1.el7.x86_64.rpm-bundle.tar            
>
>mysql-community-embedded-devel-5.7.25-1.el7.x86_64.rpm
>mysql-community-client-5.7.25-1.el7.x86_64.rpm   
>
> mysql-community-libs-5.7.25-1.el7.x86_64.rpm
>mysql-community-common-5.7.25-1.el7.x86_64.rpm    
>
>mysql-community-libs-compat-5.7.25-1.el7.x86_64.rpm
>mysql-community-devel-5.7.25-1.el7.x86_64.rpm     
>
>mysql-community-server-5.7.25-1.el7.x86_64.rpm
>mysql-community-embedded-5.7.25-1.el7.x86_64.rpm  mysql-community-test-5.7.25-1.el7.x86_64.rpm

### 3.卸载自带的mysql和mariadb

```bash
#查询
# rpm -qa | grep MariaDB
# rpm -qa | grep  mysql

#卸载
rpm -e mariadb-libs-1:5.5.56-2.el7.x86_64 --nodeps
```

### 4.依次安装各个rpm的mysql组件包

```bash
rpm -ivh mysql-community-common-5.7.27-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.27-1.el7.x86_64.rpm 
rpm -ivh mysql-community-client-5.7.27-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.271.el7.x86_64.rpm
rpm -ivh mysql-community-devel-5.7.27-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-compat-5.7.27-1.el7.x86_64.rpm
```

***注意：可能出现的报错***

>warning: mysql-community-libs-5.7.25-1.el7.x86_64.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
>error: Failed dependencies:
>mysql-community-common(x86-64) >= 5.7.9 is needed by mysql-community-libs-5.7.25-1.el7.x86_64



>Preparing...                          ################################# [100%]
>	file /usr/share/mysql/czech/errmsg.sys from install of mysql-community-common-5.7.25-1.el7.x86_64 conflicts with file from package mariadb-libs-1:5.5.60-1.el7_5.x86_64
>	file /usr/share/mysql/danish/errmsg.sys from install of mysql-community-common-5.7.25-1.el7.x86_64 conflicts with file from package mariadb-libs-1:5.5.60-1.el7_5.x86_64

**出现以上报错的主要原因还是自带的mysql和mariadb没有卸载干净，注意一定要检查和卸载**

另外可能还需要其他的安装其他的依赖包

```bash

```

在线安装




### 5.启动服务

```bash
# systemctl start mysqld
开机自启动mysql服务
# systemctl enable mysqld
```

### 6.查询初始密码

```shell
# cat /var/log/mysqld.log | grep passw
```

>2019-06-27T06:53:17.393288Z 1 [Note] A temporary password is generated for root@localhost: Gzja+qiU3BQ9

***如果无法找到密码，请按照以下步骤***

1. 设置无密码登录

```bash
# vi /etc/my.cnf

# 在如下位置添加一项配置
# Disabling symbolic-links is recommended to prevent assorted security risks
#添加这句话，这时候登入mysql就不需要密码
skip-grant-tables     
```

2. 重启mysql服务

```bash
# systemctl restart mysqld
```

3. 无密码登录

```
mysql
```



### 7.登陆，修改密码安全策略，授权远程登陆

```mysql
mysql -u root -p

# 修改密码安全策略
#修改密码长度限制
set global validate_password_policy=0;
set global validate_password_length=1;

#设置密码
ALTER USER 'root'@'localhost' IDENTIFIED BY 'anming123456!';

#授权外部主机访问
# GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'anming123456!' WITH GRANT OPTION;
update mysql.user set host='%' where user='root';
flush privileges;

#刷新授权
FLUSH PRIVILEGES;

# 测试能否进行外部访问,在window的cmd中，或者其他节点的命令行终端
mysql -h cdh01 -u root -p
```

**出错记录：**

在原定的cdh01节点上，无论是rpm方式还是yum方式，都安装不正常，启动之后，查看日志都是Error

>2019-06-30T06:41:25.478227Z 0 [ERROR] Event Scheduler: An error occurred when initializing system tables. Disabling the Event Scheduler.
>2019-06-30T06:41:25.478616Z 0 [Note] /usr/sbin/mysqld: ready for connections.
>Version: '5.7.26'  socket: '/var/lib/mysql/mysql.sock'  port: 3306  MySQL Community Server (GPL)
>2019-06-30T06:44:14.368522Z 2 [Warning] InnoDB: Table mysql/innodb_table_stats has length mismatch in the column name table_name.  Please run mysql_upgrade
>2019-06-30T06:44:14.368607Z 2 [ERROR] InnoDB: Column last_update in table `mysql`.`innodb_table_stats` is INT UNSIGNED NOT NULL but should be BINARY(4) NOT NULL (type mismatch).
>2019-06-30T06:44:14.392694Z 2 [ERROR] Incorrect definition of table mysql.proc: expected column 'sql_mode' at position 14 to have type set('REAL_AS_FLOAT','PIPES_AS_CONCAT','ANSI_QUOTES','IGNORE_SPACE','NOT_USED','ONLY_FULL_GROUP_BY','NO_UNSIGNED_SUBTRACTION','NO_DIR_IN_CREATE','POSTGRESQL','ORACLE','MSSQL','DB2','MAXDB','NO_KEY_OPTIONS','NO_TABLE_OPTIONS','NO_FIELD_OPTIONS','MYSQL323','MYSQL40','ANSI','NO_AUTO_VALUE_ON_ZERO','NO_BACKSLASH_ESCAPES','STRICT_TRANS_TABLES','STRICT_ALL_TABLES','NO_ZERO_IN_DATE','NO_ZERO_DATE','INVALID_DATES','ERRO
>2019-06-30T06:44:14.399303Z 2 [ERROR] Incorrect definition of table mysql.event: expected column 'sql_mode' at position 14 to have type set('REAL_AS_FLOAT','PIPES_AS_CONCAT','ANSI_QUOTES','IGNORE_SPACE','NOT_USED','ONLY_FULL_GROUP_BY','NO_UNSIGNED_SUBTRACTION','NO_DIR_IN_CREATE','POSTGRESQL','ORACLE','MSSQL','DB2','MAXDB','NO_KEY_OPTIONS','NO_TABLE_OPTIONS','NO_FIELD_OPTIONS','MYSQL323','MYSQL40','ANSI','NO_AUTO_VALUE_ON_ZERO','NO_BACKSLASH_ESCAPES','STRICT_TRANS_TABLES','STRICT_ALL_TABLES','NO_ZERO_IN_DATE','NO_ZERO_DATE','INVALID_DATES','ERROR_FOR_DIVISION_BY_ZERO','TRADITIONAL','NO_AUTO_CREATE_USER','HIGH_NOT_PRECEDENCE','NO_ENGINE_SUBSTITUTION','PAD_CHAR_TO_FULL_LENGTH'), found type set('REAL_AS_FLOAT','PIPES_AS_CONCAT','ANSI_QUOTES','IGNORE_SPACE','IGNORE_BAD_TABLE_OPTIONS','ONLY_FULL_GROUP_BY','NO_UNSIGNED_SUBTRACTION','NO_DIR_IN_CREATE','POSTGRESQL','ORACLE','MSSQL','DB2','MAXDB','NO_KEY_OPTIONS','NO_TABLE_OPTIONS','NO_FIELD_OPTIONS','MYSQL323','MYSQL40','ANSI','NO_AUTO_VALUE_ON_ZERO','NO_BACKSLASH_ESCAPES','STRICT_TRANS_TABLES','STRICT_A
>2019-06-30T06:44:14.998994Z 2 [Warning] System table 'servers' is expected to be transactional.
>2019-06-30T06:44:15.007780Z 2 [Warning] InnoDB: Table mysql/innodb_table_stats has length mismatch in the column name table_name.  Please run mysql_upgrade
>2019-06-30T06:44:15.007835Z 2 [ERROR] InnoDB: Column last_update in table `mysql`.`innodb_table_stats` is INT UNSIGNED NOT NULL but should be BINARY(4) NOT NULL (type mismatch).
>2019-06-30T06:44:15.007905Z 2 [ERROR] InnoDB: Fetch of persistent statistics requested for table `mysql`.`gtid_executed` but the required system tables mysql.innodb_table_stats and mysql.innodb_index_stats are not present or have unexpected structure. Using transient stats instead.
>2019-06-30T06:44:15.173583Z 2 [Warning] InnoDB: Table mysql/innodb_table_stats has length mismatch in the column name table_name.  Please run mysql_upgrade
>2019-06-30T06:44:15.173676Z 2 [ERROR] InnoDB: Column last_update in table `mysql`.`innodb_table_stats` is INT UNSIGNED NOT NULL but should be BINARY(4) NOT NULL (type mismatch).
>2019-06-30T06:44:15.173704Z 2 [Warning] InnoDB: Table mysql/innodb_table_stats has length mismatch in the column name table_name.  Please run mysql_upgrade
>2019-06-30T06:44:15.173724Z 2 [ERROR] InnoDB: Column last_update in table `mysql`.`innodb_table_stats` is INT UNSIGNED NOT NULL but should be BINARY(4) NOT NULL (type mismatch).
>2019-06-30T06:44:15.184587Z 2 [Warning] InnoDB: Table mysql/innodb_table_stats has length mismatch in the column name table_name.  Please run mysql_upgrade



**解决方式：**

MySQL的这个问题，尝试解决发现，应该是系统问题导致的，因此选择在cdh02节点安装MySQL，测试发现直接采用rpm安装没有任何问题



## (三)cloudera manager Server & Agent 安装

1. 在**所有节点**上创建 /opt/cloudera-manager

```bash
mkdir /opt/cloudera-manager
```

2. 将cm的tar安装包上传到**每个节点**的/opt/software(这个目录需要提前创建)目录下，并解压

```shell
 tar -zxvf cloudera-manager-centos7-cm5.16.2_x86_64.tar.gz -C /opt/cloudera-manager/
```

3. 在**所有节点**上创建cloudera-scm伪（系统）用户

```shell
useradd --system --home=/opt/cloudera-manager/cm-5.16.2/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
```

4.  配置cm-agent（可以为所有的agent节点，也可以是所有节点）

cd /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-agent

```shell
vi config.ini
server_host=172.23.201.191 
```

5. 配置CM数据库

   拷贝mysql-connector-java.jar到/usr/share/java/目录下，注意jdbc驱动包的名称必须是mysql-connector-java.jar

   1. 在**所有节点**上，如果没有/usr/share/java目录，请先自己 创建

   ```shell
   cp mysql-connector-java-5.1.45-bin.jar /usr/share/java/mysql-connector-java.jar
   ```

   2. 在主节点上

   ```shell
   cp mysql-connector-java-5.1.45-bin.jar /opt/cloudera-manager/cm-5.16.2/share/cmf/lib/
   ```

   3. 配置cm数据库

   进入到/opt/cloudera-manager/cm-5.16.2/share/cmf/schema，执行mysql的建库建表语句

   ```
   ./scm_prepare_database.sh mysql -h 10.0.6.117 -uroot -p123456 --scm-host 10.0.6.117 scm scm scm
   ```
   ./scm_prepare_database.sh mysql -uscm –p123456 --scm-host amxx01 scm scm scm
   **格式：**
   数据库类型、 数据库名称、 数据库服务器地址、 用户名 、密码 、server主机名
   、数据库、授权访问scm的用户名和密码

   **问题：由于未能在预定规划的cdh01节点上安装好mysql，而是在cdh02上，涉及到跨节点访问**

   ```shell
   # 1.先选择在直接执行
   ./scm_prepare_database.sh mysql -h 192.168.220.138 -uroot –p123456 --scm-host 192.168.220.138 scm scm scm
   ```
   
   报错
   
   >JAVA_HOME=/usr/java/jdk1.8.0_112
   >Verifying that we can write to /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-server
   >[                          main] DbProvisioner                  ERROR Error creating database of type: mysql
   >[                          main] DbProvisioner                  ERROR Stack Trace:
   >java.lang.IllegalArgumentException: Invalid user name or database name. DB Name: –p123456 User Name: scm. Database name and user name can only have alphanumeric characters and _
   >	at com.cloudera.enterprise.dbutil.DbProvisioner.doMain(DbProvisioner.java:106)[db-common-5.16.2.jar:]
   >	at com.cloudera.enterprise.dbutil.DbProvisioner.main(DbProvisioner.java:123)[db-common-5.16.2.jar:]
   >--> Error 1, giving up (use --force if you wish to ignore the error)
   
   ```mysql
   # 2.在cdh02的MySQL主机上授权外部主机
   GRANT ALL PRIVILEGES ON scm.* TO 'scm'@'%' IDENTIFIED BY 'anming123456!' WITH GRANT OPTION;
   
   FLUSH PRIVILEGES;
   ```

   重新尝试在cdh01上，重新部署cm的数据库

   ```shell
    ./scm_prepare_database.sh mysql -uroot -p123456 --scm-host 192.168.220.138 scm scm 123456
   ```
   
   >JAVA_HOME=/usr/java/jdk1.8.0_112
>Verifying that we can write to /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-server
   >Sun Jun 30 15:49:50 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
>Creating SCM configuration file in /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-server
   >Executing:  /usr/java/jdk1.8.0_112/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/java/postgresql-connector-java.jar:/opt/cloudera-manager/cm-5.16.2/share/cmf/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
   >Sun Jun 30 15:49:52 CST 2019 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
   >[                          main] DbCommandExecutor              INFO  Successfully connected to database.
>==All done, your SCM database is configured correctly!==



   4. 创建后续部署hive，hue等组件所需的数据库

```sql
   create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci; 
create database amon DEFAULT CHARSET utf8 COLLATE utf8_general_ci; 
   create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci; 
   create database monitor DEFAULT CHARSET utf8 COLLATE utf8_general_ci; 
   create database oozie DEFAULT CHARSET utf8 COLLATE utf8_general_ci; 

   --可以使用以下方式进行授权外部主机访问
GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'*' IDENTIFIED BY 'hive' WITH GRANT OPTION;
   GRANT ALL PRIVILEGES ON amon.* TO 'amon'@'*' IDENTIFIED BY 'amon' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON hue.* TO 'hue'@'*' IDENTIFIED BY 'hue' WITH GRANT OPTION;
   GRANT ALL PRIVILEGES ON monitor.* TO 'monitor'@'*' IDENTIFIED BY 'monitor' WITH GRANT OPTION;
   GRANT ALL PRIVILEGES ON oozie.* TO 'oozie'@'*' IDENTIFIED BY 'oozie' WITH GRANT OPTION;
   
--也可以使用以下方式进行授权外部主机访问，推荐用这种
grant all on hive.* TO 'hive'@'%' IDENTIFIED BY 'anming123456!';
grant all on hue.* TO 'hue'@'%' IDENTIFIED BY 'anming123456!';
grant all on oozie.* TO 'oozie'@'%' IDENTIFIED BY 'anming123456!';
grant all on amon.* TO 'amon'@'%' IDENTIFIED BY 'anming123456!';
grant all on monitor.* TO 'monitor'@'%' IDENTIFIED BY 'anming123456!';

   
flush privileges;
```

   5. 制作parcels

      - 在chd01节点上创建/opt/cloudera/parcel-repo目录
      
        ```shell
        mkdir -p /opt/cloudera/parcel-repo
        ```

   - 在agent（cdh02-cdh05）节点上创建/opt/cloudera/parcels目录
   
     ```shell
        mkdir -p /opt/cloudera/parcels
     ```
   
      - 上传以下文件到cdh01节点的/opt/cloudera/parcel-repo的目录下
 - 在agent（cdh02-cdh05）/opt/cloudera/parcels目录

        >**CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel**
     >**CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha1**
        >**manifest.json**

 

      - 注意：必须将CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha1最后的1去掉，否则无法生效
     
        ```shell
        mv CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha1 CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha
        ```
     
      - 更改 /opt/cloudera/parcel-repo（server）和/opt/cloudera/parcels（agent）目录的所属者与所属组为cloudera-scm
     
        agent（cdh02-cdh05）
     
        ```shell
        chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcels/  
        ```
     
        server(cdh01)
     
        ```shell
        chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo/ 
        ```

   6. 启动 CM Manager&Agent 服务
   
      注意：首次启动，耗时较长
   
      - 在server(cdh01)节点启动cloudera-scm-server
      
        ```shell
        cd /opt/cloudera-manager/cm-5.16.2/etc/init.d
        ./cloudera-scm-server start
        ```
      
        
      
      - 在agent(cdh02-cdh05)节点启动cloudera-scm-agent
      
        ```shell
        cd /opt/cloudera-manager/cm-5.16.2/etc/init.d
        ./cloudera-scm-agent start
        ```
      
      7. 启动成功后，打开web浏览器
      
         访问http://10.0.5.26:7180
      
         设置用户名和密码均为 admin

## (四)大透明页面等异常处理

1. 已启用“透明大页面”，它可能会导致重大的性能问题。

   ```shell
 echo never >/sys/kernel/mm/transparent_hugepage/defrag >> /etc/rc.local
 echo never > /sys/kernel/mm/transparent_hugepage/enabled>> /etc/rc.local
   ```

2. Cloudera 建议将 /proc/sys/vm/swappiness 设置为 10。当前设置为 60。

   ```shell
   echo 10 > /proc/sys/vm/swappiness
   ```


# 四、参数设置和基本操作

## （一）测试HDFS和yarn

### ==1.调整yarn的资源参数==

```shell
yarn.nodemanager.resource.memory-mb     #默认值是8G
yarn.nodemanager.resource.cpu-vcores    #默认值是8核

#假设服务器是16core 64G，分配给yarn的资源是这样，必须为服务器上其他的必要的应用预留一份部分资源
yarn.nodemanager.resource.memory-mb  #设置为54G
yarn.nodemanager.resource.cpu-vcores  #设置为12

```



### 2.在任一agent节点的命令行创建目录

```shell
hdfs dfs -mkdir /input
```

**报错：**

> mkdir: Permission denied: user=root, access=WRITE, inode="/":hdfs:supergroup:drwxr-xr-x

**解决办法：**

声明HDFS的操作用户（窗口变量，只在当前窗口有效）
hdfs-site.xml
	<property>
        <name>dfs.permissions.enabled</name>
        <value>false</value>
    </property>


```shell
 export HADOOP_USER_NAME=hdfs
```

### 3.上传一个文件

```shell
hdfs dfs -put data.txt /input
```

### 4.运行mapreuce的测试程序

```shell
cd /opt/cloudera/parcels/CDH-5.16.2-1.cdh5.16.2.p0.8/jars

ls | grep hadoop-mapreduce-examples
hadoop-mapreduce-examples-2.6.0-cdh5.16.2.jar

yarn jar hadoop-mapreduce-examples-2.6.0-cdh5.16.2.jar pi 300 500 
```



## (二) 测试Hive

### 1.在任一agent节点运行hive

```sql
hive

hive > show databases;
```

### 2.配置显示数据库名称

***hive*** --> 

​           ***配置***  -->

​                       ***搜索框*** --> "*hive-site.xml 的 Hive 客户端高级配置代码段（安全阀）*"

```xml
<!--显示命令行提示头-->
<property>
  <name>hive.cli.print.header</name>
  <value>true</value>
</property>
```

```xml
<!--# 显示数据库名-->
<property>
  <name>hive.cli.print.current.db</name>
  <value>true</value>
</property>
```

**==部署客户端配置 --> 重启==**

### 3.在beeline中测试创建表，导入数据，基本查询

```shell

beeline> !connect jdbc:hive2://cdh03:10000 
	   > show databases;
	   > use mydb;
	   > show tables;
	   > select * from emp;
	   > !quit
```

[虎扑体育-NBA球员得分数据排行](https://nba.hupu.com/stats/players)

 ```sql
create table plaryers(
id int,
name string,
team string,
score float,
goal  string,
goalPercent  string,
threeGoal string,
threePercent  string,
freeThrow string,
freeThrowPercent string,
times int,
long float   
)
row format delimited fields terminated by '\t';

load data inpath /input/score.txt into table players;
 ```

## (三) 测试Sqoop

## 如果导入的数据库是 mysql8  那么就要重新导入  驱动jar包

 mysql-connector-java-8.0.19.jar

 将这个jar 包 分别放到 集群各个节点的
/var/lib/sqoop
/opt/cloudera-manager/cm-5.16.2/share/cmf/lib/

这两个目录下





### 1.在任一agent节点

```shell
sqoop list-databases --connect jdbc:mysql://cdh02:3306 --username root --password 123456
```

**报错：**

> java.lang.RuntimeException: Could not load db driver class: com.mysql.jdbc.Driver

**解决：**

```shell
cd /usr/share/java

scp -r mysql-connector-java.jar cdh02:/usr/share/java/
```

### 2.测试导入操作

在数据库中创建测试用表

```sql
create  database mydb;
use mydb;
--创建表
create table if not exists customer(
cid char(6) primary key,
name varchar(30)not null,
location varchar(30),
salary decimal(8,2)
);

--插入数据
insert into customer 
values('101001','sunyang','guangzhou',1234),
('101002','guohai','nanjing',3526),
('101003','lujing','suzhou',6892),
('101004','guihui','jinan',3492);

```

sqoop导入

```shell
sqoop import \
--connect jdbc:mysql://cdh02:3306/mydb \
--username root \
--password 123456 \
--table customer \
--delete-target-dir \
--target-dir /sqoop/data \
-m 1 \
--fields-terminated-by ','
```

## (四) Flume

### 1.编辑flume agent

在flume“配置”中的“Agent Default Group”，设置代理名称为 “a1”，配置文件用如下配置替换

```
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /opt/cloudera-manager/cm-5.16.2/log/cloudera-scm-agent/supervisord.log
a1.sources.r1.channels = c1

# Describe the sink
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.path = hdfs://nameservice1/cmagent/logs/%Y-%m-%d
# default:FlumeData
a1.sinks.k1.hdfs.filePrefix = cmagent-
# useLocalTimeStamp set true
a1.sinks.k1.hdfs.useLocalTimeStamp = true
a1.sinks.k1.hdfs.rollInterval = 0
a1.sinks.k1.hdfs.rollCount = 0
# block 128 120 125
a1.sinks.k1.hdfs.rollSize = 10240
a1.sinks.k1.hdfs.fileType = DataStream
#a1.sinks.k1.hdfs.round = true
#a1.sinks.k1.hdfs.roundValue = 10
#a1.sinks.k1.hdfs.roundUnit = minute

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1	
```

### 2.启动并查看日志信息

### 3.查看结果

**报错：**

> org.apache.hadoop.security.AccessControlException: Permission denied: user=flume, access=WRITE, inode="/":hdfs:supergroup:drwxr-xr-x	

**原因：flume 会以flume用户访问hdfs**

**解决办法：**

在hdfs上手动将cmagent/logs创建出来，更改该目录的所属者为flume

```shell
 export HADOOP_USER_NAME=hdfs
 hdfs dfs -mkdir /cmagent
 
 hdfs dfs -chown -R flume /cmagent
```

## (五) Hue和Oozie的部署及测试

### 1.上传Oozie的依赖包

上传Oozie的Web console的依赖的 Web console的依赖的ext-2.2.zip到某个agent节点（cdh02）

### 2.分发到其他的agent节点

```shell
scp ext-2.2.zip cdh03:/opt/software/
scp ext-2.2.zip cdh04:/opt/software/
scp ext-2.2.zip cdh05:/opt/software/

#每个节点上都进行
cp ext-2.2.zip /opt/cloudera/parcels/CDH/lib/oozie/libext/
cd /opt/cloudera/parcels/CDH/lib/oozie/libext/
unzip ext-2.2.zip
chown oozie:oozie -R ext-2.2
```

### 3. 检查或者创建所需的数据库表并授权访问

```sql
--创建oozie 所需要的库
create database oozie default character set utf8; 
grant all privileges on oozie.* to 'oozie'@'%' identified by 'oozie' with grant option;
flush privileges; 

--创建hue所需要的库：
create database hue default character set utf8; 
grant all privileges on hue.* to 'hue'@'%' identified by 'huehue' with grant option;
flush privileges; 
```



# 五 升级Spark2.4 

## （一）升级spark2.4 的系统及组件版本要求

[CDS Powered by Apache Spark Requirements | 2.4.x | Cloudera Documentation]( https://www.cloudera.com/documentation/spark2/latest/topics/spark2_requirements.html)

**要求如下：**

- **java ：** jdk1.8 
- **scala：** 只支持2.11.x（不需要提前安装）
- **python：**python2.7 或者 python3.4 python3.5 更高
- **cm：** Cloudera Manager 5.11 and any higher Cloudera Manager 5.x versions

## （二）下所需的csd和parcels文件

[csd和parcels文件下载地址](http://archive.cloudera.com/spark2)

 **csd文件：**

> SPARK2_ON_YARN-2.4.0.cloudera2.jar

**parcels文件及校验文件：**

> manifest.json
>
> SPARK2-2.4.0.cloudera2-1.cdh5.13.3.p0.1041012-el7.parcel
>
> SPARK2-2.4.0.cloudera2-1.cdh5.13.3.p0.1041012-el7.parcel.sha1

## （三）安装步骤

### 1.停止整个集群和cm

```shell
#停止集群
在7180的web界面，集群 -》 停止

# cm-server节点
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-server stop

# cm-agent节点
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent stop
```

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/06SMjT6KtbBQYWhbjbmClgniBZ4NHkaHhCH44fOXcrQ!/b/dMUAAAAAAAAA&bo=7gHRAgAAAAADBx4!&rf=viewer_4)

### 2. 上传CSD包到CM主节点(cdh01)的/opt/cloudera/csd目录

```shell
# 如果没有/opt/cloudera/csd，手动创建
mkdir /opt/cloudera/csd

#拷贝csd包到/opt/cloudera/csd/
mv /opt/software/SPARK2_ON_YARN-2.4.0.cloudera2.jar /opt/cloudera/csd/

#修改所属用户为cloudera-scm
chown -R cloudera-scm:cloudera-scm /opt/cloudera/csd/

#效果如下
ll /opt/cloudera/csd/

-rw-r--r-- 1 cloudera-scm cloudera-scm 19066 7月   2 11:56 SPARK2_ON_YARN-2.4.0.cloudera2.jar

```



### 3. 上传parcel的3个包到CM主节点(cdh02）的/opt/cloudera/parcel-repo目录下

```shell
# 由于之前该目录下已经有cdh的parcel的manifest.json
mv /opt/cloudera/parcel-repo/manifest.json /opt/cloudera/parcel-repo/manifest.json.bak

#上传之后，修改/opt/cloudera/parcel-repo目录所属用户
chown -R cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo

ll /opt/cloudera/parcel-repo/
-rw-r--r-- 1 cloudera-scm cloudera-scm 2132782197 6月  30 15:54 CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel
-rw-r--r-- 1 cloudera-scm cloudera-scm         40 6月  30 15:54 CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.sha
-rw-r----- 1 cloudera-scm cloudera-scm      81526 6月  30 16:02 CDH-5.16.2-1.cdh5.16.2.p0.8-el7.parcel.torrent
-rw-r--r-- 1 cloudera-scm cloudera-scm       5327 7月   2 11:56 manifest.json
-rw-r--r-- 1 cloudera-scm cloudera-scm      68484 6月  30 15:54 manifest.json.bak
-rw-r--r-- 1 cloudera-scm cloudera-scm  198924405 7月   2 11:56 SPARK2-2.4.0.cloudera2-1.cdh5.13.3.p0.1041012-el7.parcel
-rw-r--r-- 1 cloudera-scm cloudera-scm         41 7月   2 11:56 SPARK2-2.4.0.cloudera2-1.cdh5.13.3.p0.1041012-el7.parcel.sha

```

##   （四）  启动CM和集群

```shell
# cm-server节点
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-server start

# cm-agent节点
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent start
```

## （五）部署Spark2.4

### 1.检查或者配置jdk（从一开始就是符合要求的jdk1.8，因此这一步可以不做）



![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/McfZsXjHfgQp0TSFSvOAlXn5.v0SGsMCigUeuALYvYc!/b/dE0BAAAAAAAA&bo=sgYoAgAAAAADB7w!&rf=viewer_4)



![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/HBosZMwjOiA567j58uxCmeC0XxgIzuBgv9d39V3xBYA!/b/dD4BAAAAAAAA&bo=FwS5AgAAAAADF5o!&rf=viewer_4)

### 2.分配激活Spark2的parcel

**查看Parcle包**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/ezmNB5uD*.2f9zFO8GM2w7UlSGiGN.vw8I7HcTNh14I!/b/dFQBAAAAAAAA&bo=3AKRAQAAAAADF3w!&rf=viewer_4)



**检查是否识别成功**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/*V*qPodRyC*pr0DzKeES9LVA6k*rjqQHVrLIf3dRctw!/b/dFMBAAAAAAAA&bo=4QOpAgAAAAADF3s!&rf=viewer_4)

**分配Parcle**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/XrPbVICmICVWtbPSQXWtMhFzoBrQQrH2pACf18mqwz4!/b/dLYAAAAAAAAA&bo=Owb7AAAAAAADF*Q!&rf=viewer_4)

**激活Parcel**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/.XIYfBxfKBxkU5L*neNY1e9rngBBzbNnDNwZkYTQz5I!/b/dDABAAAAAAAA&bo=PQbTAAAAAAADF9o!&rf=viewer_4)



![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/hvQD50TGYEQRx2j9R.rCnEEaomDQRYGykRKxjytPIOs!/b/dDEBAAAAAAAA&bo=MQbnAAAAAAADF.I!&rf=viewer_4)

### 3.添加部署Spark2.4

**准备添加spark2**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/Zcv6fWtjgD*7Gal9BpDeWsu4bSRjMlLokbS5.n7GbEw!/b/dE8BAAAAAAAA&bo=fQV7AQAAAAADFzA!&rf=viewer_4)

**选择与其他框架的集成依赖**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/pBCeWS*eTmCFKdz5wS2T5FSOwh8QjMN2xZLlW9vBoUQ!/b/dL4AAAAAAAAA&bo=bwWTAQAAAAADF8o!&rf=viewer_4)

**分配主机角色**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/TpTpLs1NjzPuSfVfS00qEQKd2gatx9dZj9zGk9YeRoo!/b/dL4AAAAAAAAA&bo=ZgRKAQAAAAADFxs!&rf=viewer_4)

**重启依赖的服务的重启**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/2cnffNEOT94YFo.87Tfxq9iR7jI7mTKovcs6gvEGJno!/b/dLYAAAAAAAAA&bo=mgWmAQAAAAADFwo!&rf=viewer_4)

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/BZd*c3MT0Zqys7keQV0MDG0UxyNfTv95Zcw9yHL1XFk!/b/dDMBAAAAAAAA&bo=0QETAgAAAAADF*M!&rf=viewer_4)

**部署配置信息**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/e*RIBTotNYiZNg4Msk6R4WlV*rbCX2.zgm4gvOMYpXw!/b/dMUAAAAAAAAA&bo=0QJfAgAAAAADF7w!&rf=viewer_4)

**部署过程**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/bgo1jpvQQyyAEmgZ60E*NeIsb2x6YkwBjPS.P7bdocI!/b/dDUBAAAAAAAA&bo=ZAS7AQAAAAADF.g!&rf=viewer_4)

**部署过程**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/g9DfLVIXA53x3p9WzbsFXS1Q*E7nEQVRvp4fUwY*0FI!/b/dL4AAAAAAAAA&bo=rgawAQAAAAADFys!&rf=viewer_4)

**检查是否启动成功**

![](http://m.qpic.cn/psb?/V13iGoh31cNcAG/ZBisgTIcRAWKOTEv02dvD0dyr5Zaf0b*6cQglW73vaw!/b/dL8AAAAAAAAA&bo=bQYJAgAAAAADF1I!&rf=viewer_4)









# 六 CDH安装失败或中断怎么处理

### 1.server和agent分别停止服务

```shell
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-server stop
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent stop
```

### 2.删除Agent节点的UUID 

```shell
rm -rf  /opt/cloudera-manager/cm-5.16.2/lib/cloudera-scm-agent/*
```

### 3.清空主节点CM数据库

进入主节点的Mysql数据库，然后drop database cm;

### 4.重新初始化数据库

```shell
./scm_prepare_database.sh mysql cm root password
```

如果报找不到cm库，则create database cm;

server和agent分别启动服务

```shell
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-server start

/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent start
```

### 5.CDH安装前核查服务启动情况

agent：

```shell
ps -ef|grep cloudera
```

>root 17255 1 0 11:12 ? 00:00:05 /opt/cloudera-manager/cm-5.16.2/lib64/cmf/agent/build/env/bin/python /opt/cloudera-manager/cm-5.16.1/lib64/cmf/agent/build/env/bin/supervisord  
>
>root 17257 17255 0 11:12 ? 00:00:00 python2.7 /opt/cloudera-manager/cm-5.16.2/lib64/cmf/agent/build/env/bin/cmf-listener -l /opt/cloudera-manager/cm-5.16.1/log/cloudera-scm-agent/cmf_listener.log /opt/cloudera-manager/cm-5.16.1/run/cloudera-scm-agent/events 
>
>root 17460 17255 0 11:16 ? 00:00:34 python2.7 /opt/cloudera-manager/cm-5.16.2/lib64/cmf/agent/build/env/bin/flood
>
>root 31658 1 2 13:06 ? 00:00:05 python2.7 /opt/cloudera-manager/cm-5.16.2/lib64/cmf/agent/build/env/bin/cmf-agent --package_dir /opt/cloudera-manager/cm-5.16.2/lib64/cmf/service --agent_dir /opt/cloudera-manager/cm-5.16.1/run/cloudera-scm-agent --lib_dir /opt/cloudera-manager/cm-5.16.1/lib/cloudera-scm-agent --logfile /opt/cloudera-manager/cm-5.16.2/log/cloudera-scm-agent/cloudera-scm-agent.log --daemon --comm_name cmf-agent --pidfile /opt/cloudera-manager/cm-5.16.2/run/cloudera-scm-agent/cloudera-scm-agent.pid

正常是上面这四个，如果cmf-agent未启动，将supervisord kill掉再agent start

```shell
cat /opt/cloudera-manager/cm-5.16.2/log/cloudera-scm-agent/supervisord.out
```



CDH安装中途莫名失败，可以重新再来一次

hive安装报错

创建 Hive Metastore 数据库表 步骤报错
org.apache.hadoop.hive.metastore.HiveMetaException: Failed to load driver

所有节点拷贝驱动到hive的lib目录
cp /opt/cloudera-manager/cm-5.16.2/share/cmf/lib/mysql-connector-java.jar 

/opt/cloudera/parcels/CDH-5.16.2-p0.3/lib/hive/lib/

点击去逐个操作重启
hue LoadBalancer启动不了，提示Failed to find the Apache HTTPD executable.

LoadBalancer所在机器安装httpd即可
yum -y install mod_ssl httpd



cp hive-site.xml /opt/cdh/spark-2.2.1-bin-2.6.0-cdh5.14.2/conf/







```
Could not start SASL: Error in sasl_client_start (-4) SASL(-4): no mechanism available: No worthy mechs found (code THRIFTTRANSPORT): TTransportException('Could not start SASL: Error in sasl_client_start (-4) SASL(-4): no mechanism available: No worthy mechs found',)

缺少 cyrus-sasl  安装就OK了
yum install cyrus-sasl-plain cyrus-sasl-devel cyrus-sasl-gssapi  cyrus-sasl-md5 
```















