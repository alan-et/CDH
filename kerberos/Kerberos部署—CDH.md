

## Kerberos部署和使用——CDH

### 部署

#### 1. 主节点安装KDC服务

```javascript
yum -y install krb5-server krb5-libs krb5-auth-dialog krb5-workstation
```

#### 2. 修改/etc/krb5.conf配置

 `vim /etc/krb5.conf` 按照以下提示修改即可

```javascript
includedir /etc/krb5.conf.d/

[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 default_realm = ALAN.COM     # 自己替换，下面ALAN.COM需同步更改
 #default_ccache_name = KEYRING:persistent:%{uid}

[realms]
 ALAN.COM = {
  kdc = hadoop-node1         # 注意替换成主节点的hostname
  admin_server = hadoop-node1  # 同KDC
 }

[domain_realm]
 .alan.com = ALAN.COM
 alan.com = ALAN.COM
```

#### 3. 修改/var/kerberos/krb5kdc/kadm5.acl配置

`vim /var/kerberos/krb5kdc/kadm5.acl ` 替换ALAN.COM

```
*/admin@ALAN.COM      *
```

#### 4. 修改/var/kerberos/krb5kdc/kdc.conf配置

` vim /var/kerberos/krb5kdc/kdc.conf `

```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 ALAN.COM = {   # 替换ALAN.COM
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hma
c:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:norm
al des-cbc-crc:normal
 }
```

#### 5. 创建Kerberos数据库

```
kdb5_util create –r FAYSON.COM -s
```

***下面是系统提示信息***

```
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'ALAN.COM',
master key name 'K/M@FAYSON.COM'
You will be prompted for the database Master Password.
It is important that you NOT FORGET this password.
Enter KDC database master key:                     # 输入KDC数据库的密码
Re-enter KDC database master key to verify:        # 确认密码
```

#### 6. 创建Kerberos的管理账号

```shell
kadmin.local
```

***下面是系统提示信息***

```javascript
Authenticating as principal root/admin@ALAN.COM with password.
kadmin.local:  addprinc admin/admin@ALAN.COM        # 自己输入addprinc admin/admin@ALAN.COM,其中‘/admin@ALAN.COM’是上面配置文件定义的
WARNING: no policy specified for admin/admin@ALAN.COM; defaulting to no policy
Enter password for principal "admin/admin@ALAN.COM":  # 输入密码
Re-enter password for principal "admin/admin@ALAN.COM": # 输入密码
Principal "admin/admin@ALAN.COM" created.
kadmin.local:  exit
```

#### 7. 将Kerberos服务添加到自启动服务，并启动krb5kdc和kadmin服务

```javascript
systemctl enable krb5kdc
systemctl enable kadmin
systemctl start krb5kdc
systemctl start kadmin
```

#### 8. 测试

`[root@hadoop-node1 ~]#` `kinit admin/admin@ALAN.COM`

`Password for admin/admin@ALAN.COM: `

`[root@hadoop-node1 ~]#` `klist`

如果返回以下信息则成功

```javascript
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: admin/admin@ALAN.COM

Valid starting       Expires              Service principal
11/09/2020 02:52:21  11/10/2020 02:52:21  krbtgt/ALAN.COM@ALAN.COM
        renew until 11/16/2020 02:52:21 
```

#### 9. 其他节点安装Kerberos客户端

方式1：每个节点单独执行

```
yum -y install krb5-libs krb5-workstation
```

方式2：批处理脚本执行(脚本参考12.1)

```
./batch_ssh.sh 'yum -y install krb5-libs krb5-workstation' node_list.txt
```

#### 10. 在Cloudera Manager Server服务器上安装额外的包

```
yum -y install openldap-clients
```

#### 11. 将KDC Server上的krb5.conf文件拷贝到所有Kerberos客户端

批处理scp脚本参考12.2

```
./batch_scp.sh local_path remote_path node_list.txt
```

#### 12.1 SSH批处理脚本

```
#!/bin/bash

# 检查是否提供了要执行的命令
if [ -z "$1" ]; then
    echo "Usage: $0 <command_to_execute>"
    exit 1
fi

# 获取要执行的命令
command_to_execute="$1"

# 检查是否提供了服务器列表文件作为参数
if [ -z "$2" ]; then
    echo "Usage: $0 <command_to_execute> <server_list_file>"
    exit 1
fi

# 从外部文件读取服务器列表
servers_file="$2"
if [ ! -f "$servers_file" ]; then
    echo "Server list file not found: $servers_file"
    exit 1
fi

# 读取服务器列表文件中的服务器名
mapfile -t servers < "$servers_file"

# 迭代服务器列表并执行远程命令
for server in "${servers[@]}"; do
    echo "Executing command on $server"
    # 在远程服务器上执行命令
    ssh root@$server "$command_to_execute"

    echo "Command execution on $server complete"
done
```

#### 12.2 SCP批处理脚本

```
#!/bin/bash

# 检查是否提供了本地文件路径和远程目标路径
if [ -z "$1" ] || [ -z "$2" ]; then
    echo "Usage: $0 <local_file_path> <remote_target_path> <server_list_file>"
    exit 1
fi

local_file_path="$1"
remote_target_path="$2"

# 检查是否提供了服务器列表文件作为参数
if [ -z "$3" ]; then
    echo "Usage: $0 <local_file_path> <remote_target_path> <server_list_file>"
    exit 1
fi

# 从外部文件读取服务器列表
servers_file="$3"
if [ ! -f "$servers_file" ]; then
    echo "Server list file not found: $servers_file"
    exit 1
fi

# 读取服务器列表文件中的服务器名
mapfile -t servers < "$servers_file"

# 迭代服务器列表并传输文件
for server in "${servers[@]}"; do
    echo "Transferring file to $server"
    # 使用 scp 传输文件到远程服务器
    scp -r "$local_file_path" root@$server:"$remote_target_path"

    echo "File transfer to $server complete"
done

```



### CDH启用kerberos

#### 1. 在KDC中给Cloudera Manager添加管理员账号

```javascript
[root@hadoop-node1 ~]$ kadmin.local
Authenticating as principal root/admin@ALAN.COM with password.
kadmin.local:  addprinc cloudera-scm/admin@ALAN.COM
WARNING: no policy specified for cloudera-scm/admin@ALAN.COM; defaulting to no policy
Enter password for principal "cloudera-scm/admin@ALAN.COM": 
Re-enter password for principal "cloudera-scm/admin@ALAN.COM": 
Principal "cloudera-scm/admin@ALAN.COM" created.
kadmin.local:  exit
```

#### 2. 进入Cloudera Manager的“管理”=>“安全”界面，点击启用kerberos

![image](https://github.com/alan-et/CDH/assets/46310121/c1bc8a2d-04b6-429d-81df-6fa9f5fff2cf)


#### 3. 后续步骤比较简单，把配置信息填上去即可

注意 **通过Cloudera Manager管理krb5.conf** 选项不勾选；否则前面配置的会被覆盖



### 常见问题

1. 如使用A用户运行MapReduce任务及操作Hive，需要在集群所有节点创建A用户。

   使用kadmin创建一个A的principal

   ```
   [root@hadoop-node1 ~]$ kadmin.local
   Authenticating as principal root/admin@FAYSON.COM with password.
   kadmin.local:  addprinc A@ALAN.COM
   WARNING: no policy specified for fayson@FAYSON.COM; defaulting to no policy
   Enter password for principal "A@ALAN.COM": 
   Re-enter password for principal "A@ALAN.COM": 
   Principal "A@ALAN.COM" created.
   kadmin.local:  exit
   ```

2. 执行MR作业报“User XXX not found”

   问题原因：在集群的节点上没有XXX 这个用户

   解决方法：需要在集群所有节点添加XXX 用户

   检查用户是否存在

   ```
   id xxx
   ```

   ```
   useradd xxx
   passwd xxx
   gpasswd -a xxx xxx
   ```

   

   
