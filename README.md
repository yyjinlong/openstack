openstack kilo for ubuntu server 14.04 (all in one)
===================================================


虚拟网络
-------
```
我们需要新建3个虚拟网络vboxnet1, vboxnet2, vboxnet3, 其virtualbox全局配置如下:
vboxnet1:
    Network name: VirtualBox  host-only Ethernet Adapter#1
    Purpose: administrator / management network
    IP block: 10.20.0.0/24
    DHCP: disable
    Linux device: eth0

vboxnet2:
    Network name: VirtualBox  host-only Ethernet Adapter#2
    Purpose: public network
    DHCP: disable
    IP block: 172.16.0.0/24
    Linux device: eth1

vboxnet3：
    Network name: VirtualBox  host-only Ethernet Adapter#3
    Purpose: Storage/private network
    DHCP: disable
    IP block: 192.168.4.0/24
    Linux device: eth2
```


安装ubuntu server 14.04
----------------------

```
1) 网卡配置: 高级选项中的控制芯片、混杂模式 均不改.
   网卡1的界面名称选择: (仅主机(Host-Only)适配器), 界面名称选择vboxnet1
   网卡2的界面名称选择: (仅主机(Host-Only)适配器), 界面名称选择vboxnet2
   网卡3的连接方式选择: (仅主机(Host-Only)适配器), 界面名称选择vboxnet3
   网卡4的连接方式选择: `网络地址转换(NAT)`

2) 系统配置: 2cpu 20Gdisk 2Gmemory

3) ubuntu server安装过程中
   a) location为hongkong.
   b) 选择dhcp上网的网卡为eth3.

4) 安装完成后,ifconfig查看只有eth3和lo,所以需要下面的网卡配置.
```


网卡配置
-------
```sh
# cat /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth3
iface eth3 inet dhcp

# The management network interface
auto eth0
iface eth0 inet static
address 10.20.0.10
network 255.255.255.0

# The external network interface
auto eth1
iface eth1 inet static
address 172.16.0.10
network 255.255.255.0

# The private network interface
auto eth2
iface eth2 inet static
address 192.168.4.10
network 255.255.255.0
```


安装openstack包
--------------
0) 安装过程中推荐使用`root`

1) 安装`ubuntu-cloud-keyring`
```sh
apt-get install ubuntu-cloud-keyring
```

2) 添加`kilo`源
```sh
echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu" \
  "trusty-updates/kilo main" > /etc/apt/sources.list.d/cloudarchive-kilo.list
```

3) 更新及源升级
```sh
apt-get update && apt-get dist-upgrade
```


### 一、Ubuntu14.04远程连接（ssh安装）

ssh是一种安全协议，主要用于给远程登录会话数据进行加密，保证数据传输的安全.

1) 更新源列表
```sh
apt-get update
```

2) 安装ssh
```sh
apt-get install openssh-server
```

3) 查看ssh服务是否启动
```sh
ps -e |grep ssh
```

如果有sshd,说明ssh服务已经启动，如果没有启动，则输入如下命令服务就会启动
```sh
service ssh start
```

3) 修改配置文件
```sh
vim /etc/ssh/sshd_config
注: ssh默认端口号是22，可以修改（但不建议）。
PermitRootLogin without-password 把这一句注释掉，不然有设置密码的root用户就无法登陆。
PermitRootLogin yes 增加这一句，允许root用户登陆。
```


二、NTP安装
----------
```
ntp同步方式有两种：
1) 一种是同步远程网络服务器
2) 一种是同步局域网内的服务器。

如果多节点安装：
1) 控制节点（controller）同步的是远程网络服务器
2) 计算节点和网络节点则是同步的控制节点

一）控制节点
ntp配置的思路：
控制节点同步其它服务器，可以有多个。其它节点同步控制节点。
```

1) 安装NTP
```sh
apt-get install ntp
```

2) 修改配置文件
```
修改 /etc/ntp.conf，添加如下配置:
server NTP_SERVER iburst
restrict -4 default kod notrap nomodify
restrict -6 default kod notrap nomodify

其中NTP_SERVER需要修改,

如果控制节点为controller，直接使用下面即可
server controller iburst
restrict -4 default kod notrap nomodify
restrict -6 default kod notrap nomodify

我的配置为:
# Use Ubuntu's ntp server as a fallback.
server ntp.ubuntu.com

server openstack iburst
restrict -4 default kod notrap nomodify
restrict -6 default kod notrap nomodify

注：
对于 restrict ,可以去掉 nopeer and noquery 这两个选项.这里不需要注释掉其他server选项
如果 /var/lib/ntp/ntp.conf文件存在，则移动
```
3) 重启NTP
```sh
service ntp restart
```

4) 验证安装
```sh
root@openstack:~# ntpq -c peers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 dns.sjtu.edu.cn .INIT.          16 u    - 1024    0    0.000    0.000   0.000
 news.neu.edu.cn .INIT.          16 u    - 1024    0    0.000    0.000   0.000
 juniperberry.ca .INIT.          16 u    - 1024    0    0.000    0.000   0.000

root@openstack:~# ntpq -c assoc
ind assid status  conf reach auth condition  last_event cnt
===========================================================
  1 56777  8011   yes    no  none    reject    mobilize  1
  2 56778  8011   yes    no  none    reject    mobilize  1
  3 56779  8011   yes    no  none    reject    mobilize  1
root@openstack:~#
```


三、mysql（MariaDB）安装【控制节点】
-----------------------------------
```
问题导读
1.MariaDB与mysql的关系是什么？
2.遇到Checking for corrupt, not cleanly closed and upgrade needing tables.该如何解决？

为什么产生MariaDB
首先这里介绍一下，大家对MariaDB可能不太熟悉，MariaDB数据库管理系统是MySQL的一个分支,
主要由开源社区在维护，采用GPL授权许可。开发这个分支的原因之一是：甲骨文公司收购了MySQL后,
有将MySQL闭源的潜在风险，因此社区采用分支的方式来避开这个风险。
```

1) 安装
```sh
apt-get install mariadb-server python-mysqldb -y
```

2) 创建文件/etc/mysql/conf.d/mysqld_openstack.cnf
```
在[mysqld]部分,设置bind-address为控制节点管理网络ip地址,使能通过管理网络访问其它节点,并设置其他内容:

# vim /etc/mysql/conf.d/mysqld_openstack.cnf
[mysqld]
bind-address = 127.0.0.1
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'set names utf8'
character-set-server = utf8
```

3) 重启mysql
```sh
service mysql restart

输出如下信息
* Stopping MariaDB database server mysqld                                                                                                    [ OK ]
* Starting MariaDB database server mysqld                                                                                                    [ OK ]
* Checking for corrupt, not cleanly closed and upgrade needing tables.
这个只是个提示，告诉你在做什么。不管它。
```

4) 执行如下命令
```sh
# mysql_secure_installation
........................
Thanks for using MariaDB!
```


四、RabbitMQ 安装【控制节点】 
---------------------------

1.安装消息队列服务
```sh
# apt-get install rabbitmq-server
```

2.配置消息队列服务
```sh
a) 添加 openstak 用户
# rabbitmqctl add_user openstack 123456

b) 禁止访问读写openstack user
# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```


五、keystone安装与配置
---------------------
1) 创建keystone数据库
```sh
$ mysql -uroot -p

MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> grant all privileges on keystone.* to 'keystone'@'localhost' identified by '123456';
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> grant all privileges on keystone.* to 'keystone'@'%' identified by '123456';
Query OK, 0 rows affected (0.00 sec)

注: 上面的含义实现了，对keystone用户实现了，本地和远程都可以访问

退出mysql
MariaDB [(none)]> quit
```

2) 安装keystone包
```
1. 默认keystone服务监听端口5000 和 35357，尽管如此向导配置 Apache HTTP server 监听这些端口，
   为了避免端口冲突，安装后禁止开机启动keystone 服务。
   # echo "manual" > /etc/init/keystone.override

2. 下载并安装keystone
   # apt-get install keystone python-openstackclient apache2 libapache2-mod-wsgi \
                     memcached python-memcache -y
```

3) 编辑/etc/keystone/keystone.conf
```
# vim /etc/keystone/keystone.conf
[DEFAULT]
verbose = True
admin_token = 570f150cb897e793e58f

[database]
connection = mysql://keystone:123456@localhost/keystone

注,记得一定注释掉：
connection=sqlite:////var/lib/keystone/keystone.db

[memcache]
servers = localhost:11211

[token]
provider = keystone.token.providers.uuid.Provider
driver = keystone.token.persistence.backends.memcache.Token

修改 [revoke] 部分, 配置  SQL revocation driver:
driver = keystone.contrib.revoke.backends.sql.Revoke
```

填充keystone 注：最好切到root执行,否则会同步不成功
```sh
# /bin/sh -c "keystone-manage db_sync" keystone
```

4) 配置 Apache HTTP server
```
1. 修改配置文件 /etc/apache2/apache2.conf，配置ServerName选项为控制节点hostname
ServerName localhost

2. 创建/etc/apache2/sites-available/wsgi-keystone.conf 文件，添加如下内容
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /var/www/cgi-bin/keystone/main
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
    ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    LogLevel info
    ErrorLog /var/log/apache2/keystone-error.log
    CustomLog /var/log/apache2/keystone-access.log combined
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /var/www/cgi-bin/keystone/admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
    ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    LogLevel info
    ErrorLog /var/log/apache2/keystone-error.log
    CustomLog /var/log/apache2/keystone-access.log combined
</VirtualHost>

3. 启用身份服务虚拟主机：
    # ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled

4. 创建WSGI组件的目录结构：
    # mkdir -p /var/www/cgi-bin/keystone

5. 下载复制WSGI组件到目录 /var/www/cgi-bin/keystone
    # curl http://git.openstack.org/cgit/openstack/keystone/plain/httpd/keystone.py?h=stable/kilo \
      | tee /var/www/cgi-bin/keystone/main /var/www/cgi-bin/keystone/admin

6. 修改权限
    # chown -R keystone:keystone /var/www/cgi-bin/keystone
    # chmod 755 /var/www/cgi-bin/keystone/*
```

5) 完成安装
1. 重启 Apache HTTP server:
```sh
# service apache2 restart
```

2. 如果存在 SQLite 数据库，则删除
```sh
# rm -f /var/lib/keystone/keystone.db
```


六：创建服务实例和 API endpoint
------------------------------
```
1、准备
    1）配置token
        # export OS_TOKEN=570f150cb897e793e58f
        格式：export OS_TOKEN=ADMIN_TOKEN
        注: ADMIN_TOKEN为keystone的配置文件里的admin_token的设置值

    2）配置endpoint url
        # export OS_URL=http://localhost:35357/v2.0

2、创建服务实例和API endpoint

    1）创建Identity实例服务
        # openstack service create \
          --name keystone --description "OpenStack Identity" identity
        +-------------+----------------------------------+
        | Field       | Value                            |
        +-------------+----------------------------------+
        | description | OpenStack Identity               |
        | enabled     | True                             |
        | id          | db9474ce160d44398923ad76ce5b6732 |
        | name        | keystone                         |
        | type        | identity                         |
        +-------------+----------------------------------+

    2）创建实例服务
        # openstack endpoint create \
          --publicurl http://localhost:5000/v2.0 \
          --internalurl http://localhost:5000/v2.0 \
          --adminurl http://localhost:35357/v2.0 \
          --region RegionOne \
          identity
        +--------------+----------------------------------+
        | Field        | Value                            |
        +--------------+----------------------------------+
        | adminurl     | http://localhost:35357/v2.0      |
        | id           | 27cebe349b2249ed86b76cf953545e5c |
        | internalurl  | http://localhost:5000/v2.0       |
        | publicurl    | http://localhost:5000/v2.0       |
        | region       | RegionOne                        |
        | service_id   | db9474ce160d44398923ad76ce5b6732 |
        | service_name | keystone                         |
        | service_type | identity                         |
        +--------------+----------------------------------+
```


七、创建租户、用户、角色
-----------------------

1、创建管理员租户、用户、角色
```sh
1) 创建admin租户
# openstack project create --description "admin project" admin
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | admin project                    |
| enabled     | True                             |
| id          | aa3622f4a6ee4c379f4294b4492562b0 |
| name        | admin                            |
+-------------+----------------------------------+

2) 创建admin用户
# openstack user create --password-prompt admin
User Password:
Repeat User Password:
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | 22488b4d6ec3414aac16de978337909d |
| name     | admin                            |
| username | admin                            |
+----------+----------------------------------+

3) 创建admin角色
# openstack role create admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | d574ffdf0ef748138fb3ec2460d47d9e |
| name  | admin                            |
+-------+----------------------------------+

4) 添加 admin 角色到 admin 租户和用户
# openstack role add --project admin --user admin admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | d574ffdf0ef748138fb3ec2460d47d9e |
| name  | admin                            |
+-------+----------------------------------+
```

2、创建一个service租户
```sh
# openstack project create --description "service project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | service project                  |
| enabled     | True                             |
| id          | 255de1bf31434000a4824ef4aec84f36 |
| name        | service                          |
+-------------+----------------------------------+
```

3、创建非管理员demo租户
```sh
1) 创建demo租户
# openstack project create --description "demo project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | demo project                     |
| enabled     | True                             |
| id          | b8f1c26e2b3c447e8ba664fc999c6e70 |
| name        | demo                             |
+-------------+----------------------------------+

2) 创建demo用户
# openstack user create --password-prompt demo
User Password:
Repeat User Password:
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | de83ed698d0744e29c75fed800498670 |
| name     | demo                             |
| username | demo                             |
+----------+----------------------------------+

3) 创建user角色
# openstack role create user
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 8b355da0962d4da692d05a60cec737a1 |
| name  | user                             |
+-------+----------------------------------+

4) 添加 user 角色到 demo 租户和用户
# openstack role add --project demo --user demo user
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 8b355da0962d4da692d05a60cec737a1 |
| name  | user                             |
+-------+----------------------------------+
```


八、验证keystone安装部署
-----------------------
```
1、为了安全，禁用临时token
编辑 /etc/keystone/keystone-paste.ini 文件，移除admin_token_auth从
[pipeline:public_api], [pipeline:admin_api], 和[pipeline:api_v3]部分.

下面举个例子[pipeline:public_api],已经从标签中移除admin_token_auth
[pipeline:public_api]
# The last item in this pipeline must be public_service or an equivalent
# application. It cannot be a filter.
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ......

2、去掉环境变量OS_TOKEN、OS_URL
# unset OS_TOKEN OS_URL

3、作为管理员，请求身份验证令牌API版本2
# openstack --os-auth-url http://localhost:35357 \
  --os-project-name admin --os-username admin --os-auth-type password \
  token issue
  Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-11-16T16:26:20Z             |
| id         | a0d7182b5bf24549886a9f45b903c905 |
| project_id | aa3622f4a6ee4c379f4294b4492562b0 |
| user_id    | 22488b4d6ec3414aac16de978337909d |
+------------+----------------------------------+

4、Identity 版本 3 API添加支持域
# openstack --os-auth-url http://localhost:35357 \
  --os-project-domain-id default --os-user-domain-id default \
  --os-project-name admin --os-username admin --os-auth-type password \
  token issue
  Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-11-16T16:33:53.150489Z      |
| id         | 5956011a5bb24bf38910b31e1f2d89f1 |
| project_id | aa3622f4a6ee4c379f4294b4492562b0 |
| user_id    | 22488b4d6ec3414aac16de978337909d |
+------------+----------------------------------+
注意密码: 是admin用户的密码

5、作为admin用户，列出用户作为admin核实admin可以执行admin-only CLI命令
# openstack --os-auth-url http://localhost:35357 \
  --os-project-name admin --os-username admin --os-auth-type password \
  project list
  Password:
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 255de1bf31434000a4824ef4aec84f36 | service |
| aa3622f4a6ee4c379f4294b4492562b0 | admin   |
| b8f1c26e2b3c447e8ba664fc999c6e70 | demo    |
+----------------------------------+---------+

6、作为admin用户，列出用户核实认证服务
# openstack --os-auth-url http://localhost:35357 \
  --os-project-name admin --os-username admin --os-auth-type password \
  user list
  Password:
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 22488b4d6ec3414aac16de978337909d | admin |
| de83ed698d0744e29c75fed800498670 | demo  |
+----------------------------------+-------+

7、作为 admin 用户, 列出角色验证keystone服务
# openstack --os-auth-url http://localhost:35357 \
  --os-project-name admin --os-username admin --os-auth-type password \
  role list
  Password:
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 8b355da0962d4da692d05a60cec737a1 | user  |
| d574ffdf0ef748138fb3ec2460d47d9e | admin |
+----------------------------------+-------+

8、作为demo用户，请求token 认证从3版本的API
# openstack --os-auth-url http://localhost:5000 \
  --os-project-domain-id default --os-user-domain-id default \
  --os-project-name demo --os-username demo --os-auth-type password \
  token issue
  Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-11-16T16:42:11.621579Z      |
| id         | e424226f71b14ef1895207769a9eed31 |
| project_id | b8f1c26e2b3c447e8ba664fc999c6e70 |
| user_id    | de83ed698d0744e29c75fed800498670 |
+------------+----------------------------------+
注释：这里输入demo密码

9、作为 demo 用户, 尝试列出用户不能执行 admin-only CLI 命令
# openstack --os-auth-url http://localhost:5000 \
  --os-project-domain-id default --os-user-domain-id default \
  --os-project-name demo --os-username demo --os-auth-type password \
  user list
Password:
ERROR: openstack You are not authorized to perform the requested action: admin_required (HTTP 403) (Request-ID: req-f7d2441e-d029-4ea7-88e9-ae7797cca764)
```


九、创建openstack客户端环境变量脚本
----------------------------------
1、创建脚本
```sh
创建admin 和 demo 租户脚本

a) 编辑admin-openrc.sh 文件，添加如下内容
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=123456
export OS_AUTH_URL=http://localhost:35357/v3
export OS_REGION_NAME=RegionOne

b) 编辑文件demo-openrc.sh ，添加如下内容
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=demo
export OS_TENANT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=123456
export OS_AUTH_URL=http://localhost:5000/v3
export OS_REGION_NAME=RegionOne
```

2、加载客户端环境脚本
```sh
1) 加载admin-openrc.sh环境变量
    # source admin-openrc.sh

2) 请求认证令牌
    # openstack token issue
    +------------+----------------------------------+
    | Field      | Value                            |
    +------------+----------------------------------+
    | expires    | 2015-11-17T16:34:46.403966Z      |
    | id         | 077e1df72cd740628861552b5085d777 |
    | project_id | aa3622f4a6ee4c379f4294b4492562b0 |
    | user_id    | 22488b4d6ec3414aac16de978337909d |
    +------------+----------------------------------+
```


十、glance安装配置【控制节点】
----------------------------

1) 创建glance 数据库
```sh
# mysql -uroot -p
MariaDB [(none)]> CREATE DATABASE glance;
Query OK, 1 row affected (0.10 sec)
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| glance             |
| keystone           |
| mysql              |
| performance_schema |
+--------------------+
5 rows in set (0.06 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' identified by '123456';
Query OK, 0 rows affected (1.38 sec)
MariaDB [(none)]> grant all privileges on glance.* to 'glance'@'%' identified by '123456';
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> Ctrl-C -- exit!
Aborted
```

2) 生效admin环境变量
```sh
 source admin-openrc.sh
```

3) 创建认证服务
```sh
a) 创建glance用户
    # openstack user create --password-prompt glance
    User Password:
    Repeat User Password:
    +----------+----------------------------------+
    | Field    | Value                            |
    +----------+----------------------------------+
    | email    | None                             |
    | enabled  | True                             |
    | id       | d7e038743884432da55138974bceb1da |
    | name     | glance                           |
    | username | glance                           |
    +----------+----------------------------------+

b) 添加admin角色到glance用户和service租户
    # openstack role add --project service --user glance admin
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | d574ffdf0ef748138fb3ec2460d47d9e |
    | name  | admin                            |
    +-------+----------------------------------+

c) 创建glance服务实例
    # openstack service create --name glance \
    --description "OpenStack Image Server" image
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | OpenStack Image Server           |
    | enabled     | True                             |
    | id          | 3ac5b88d9e00413cb94b8b567d4ce38a |
    | name        | glance                           |
    | type        | image                            |
    +-------------+----------------------------------+
```

4) 创建镜像服务API endpoint
```sh
# openstack endpoint create \
  --publicurl http://localhost:9292 \
  --internalurl http://localhost:9292 \
  --adminurl http://localhost:9292 \
  --region RegionOne \
  image
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://localhost:9292            |
| id           | b11e3e637f8f4726969d25ffa79813f8 |
| internalurl  | http://localhost:9292            |
| publicurl    | http://localhost:9292            |
| region       | RegionOne                        |
| service_id   | 3ac5b88d9e00413cb94b8b567d4ce38a |
| service_name | glance                           |
| service_type | image                            |
+--------------+----------------------------------+
```

5) 安装配置glance服务组件

1、安装glance
```sh
# apt-get install glance python-glanceclient -y
```

2、编辑文件 /etc/glance/glance-api.conf，完成下面内容
```
a) 在[database]部分，配置数据库访问
    [database]
    connection = mysql://glance:123456@localhost/glance

b) 在[keystone_authtoken]和[paste_deploy]部分, 配置Identity服务访问

    [keystone_authtoken]
    auth_uri = http://localhost:5000
    auth_url = http://localhost:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = glance
    password = 123456
    注: 注释掉其它[keystone_authtoken]部分

    [paste_deploy]
    flavor = keystone

c) 在[glance_store]部分, 配置本地文件系统存储和image文件路径

    [glance_store]
    default_store = file
    filesystem_store_datadir = /var/lib/glance/images/

d) 在[DEFAULT]部分，配置noop禁用通知驱动，因为它只属于可选的遥测服务

    [DEFAULT]
    notification_driver = noop

e) 在[DEFAULT]部分启用日志详细信息记录

    [DEFAULT]
    verbose = True
```

3、编辑 /etc/glance/glance-registry.conf 文件，完成下面内容
```
a) 在[database]部分，配置数据库访问

    [database]
    connection = mysql://glance:123456@localhost/glance

b) 在[keystone_authtoken]和[paste_deploy]部分, 配置 Identity service 访问

    [keystone_authtoken]
    auth_uri = http://localhost:5000
    auth_url = http://localhost:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = glance
    password = 123456
    注: 注释掉其它[keystone_authtoken]部分

    [paste_deploy]
    flavor = keystone

c) 在[DEFAULT]部分, 配置noopdriver禁用通知因为他们只属于遥测服务

    [DEFAULT]
    notification_driver = noop

d) 方便排除，启用日志信息详细记录

    [DEFAULT]
    verbose = True
```

4、同步数据库
```sh
# /bin/sh -c "glance-manage db_sync" glance
```

5、完成安装
```
1) 重启镜像服务
    # service glance-registry restart
    # service glance-api restart

2） 如果存在SQLite 数据库则删除
    # rm -f /var/lib/glance/glance.sqlite

遇到问题
    ERROR: openstack No tenant with a name or ID of 'service' exists.
    原因没有创建service 租户

    Solution: 创建Service租户即可
    # openstack project create --description "Service Project" service
```


十一、glance安装验证
-------------------
1. 在每一个客户端脚本,配置镜像服务客户端使用 API version 2.0:
```sh
 echo "export OS_IMAGE_API_VERSION=2" | tee -a admin-openrc.sh demo-openrc.sh
```

2. 生效admin环境变量
```sh
 source admin-openrc.sh
```

3. 创建一个临时目录
```sh
# mkdir /tmp/images
```

4. 下载镜像到当前目录
```sh
# wget -P /tmp/images http://download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img
```

5. 上传镜像到glance，镜像使用qcow2 格式，镜像使用格式
```sh
# glance image-create --name "cirros-0.3.3-x86_64" --file /tmp/images/cirros-0.3.3-x86_64-disk.img \
  --disk-format qcow2 --container-format bare --visibility public --progress
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 133eae9fb1c98f45894a4e60d8736619     |
| container_format | bare                                 |
| created_at       | 2015-11-22T11:43:35Z                 |
| disk_format      | qcow2                                |
| id               | 087252c1-838d-4645-977f-6e410ad051c6 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.3-x86_64                  |
| owner            | 87c4044b921a4ecda5461c2be14110a3     |
| protected        | False                                |
| size             | 13200896                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2015-11-22T11:43:36Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
```

6. 确认上次成功，核实属性
```sh
# glance image-list
+--------------------------------------+---------------------+
| ID                                   | Name                |
+--------------------------------------+---------------------+
| 087252c1-838d-4645-977f-6e410ad051c6 | cirros-0.3.3-x86_64 |
+--------------------------------------+---------------------+
```


### nova

1. 创建nova数据库及授权
```sh
# mysql -uroot -p
MariaDB [(none)]> CREATE DATABASE nova;
Query OK, 1 row affected (0.10 sec)
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| glance             |
| keystone           |
| mysql              |
| nova               |
| performance_schema |
+--------------------+
5 rows in set (0.06 sec)
MariaDB [(none)]>
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' identified by '123456';
Query OK, 0 rows affected (1.38 sec)
MariaDB [(none)]> grant all privileges on nova.* to 'nova'@'%' identified by '123456';
Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> Ctrl-C -- exit!
Aborted
```

2. 生效admin用户
```sh
# source admin-openrc.sh 
```

3. 创建keystone认证，完成下面内容

a) 创建nova用户
```sh
# openstack user create --password-prompt nova
User Password:
Repeat User Password:
+----------+----------------------------------+
| Field    | Value                            |
+----------+----------------------------------+
| email    | None                             |
| enabled  | True                             |
| id       | 86fe2ddc38c441b2b4ed25f8b6b29132 |
| name     | nova                             |
| username | nova                             |
+----------+----------------------------------+
```

b) 添加admin角色到nova用户
```sh
# openstack role add --project service --user nova admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | e5f57dabbeef4447be2aa94fc99fd45b |
| name  | admin                            |
+-------+----------------------------------+
```

4) 创建nova 服务实例
```sh
# openstack service create --name nova \
 --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 9567215f8daa4e70b5fafcec7d1365e0 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

5) 创建nova 服务 API endpoint
```sh
# openstack endpoint create \
--publicurl http://localhost:8774/v2/%\(tenant_id\)s \
--internalurl http://localhost:8774/v2/%\(tenant_id\)s \
--adminurl http://localhost:8774/v2/%\(tenant_id\)s \
--region RegionOne \
compute
+--------------+----------------------------------------+
| Field        | Value                                  |
+--------------+----------------------------------------+
| adminurl     | http://localhost:8774/v2/%(tenant_id)s |
| id           | e34e49a07d7940fba3b98684074a8881       |
| internalurl  | http://localhost:8774/v2/%(tenant_id)s |
| publicurl    | http://localhost:8774/v2/%(tenant_id)s |
| region       | RegionOne                              |
| service_id   | 9567215f8daa4e70b5fafcec7d1365e0       |
| service_name | nova                                   |
| service_type | compute                                |
+--------------+----------------------------------------+
```

6) 安装配置计算控制节点组件【控制节点】

a) 安装nova
```sh
# apt-get install nova-api nova-cert nova-conductor nova-consoleauth \
  nova-novncproxy nova-scheduler python-novaclient -y
```

b) 修改配置/etc/nova/nova.conf文件，完成下面内容
```sh
[database]
connection = mysql://nova:123456@localhost/nova

在[DEFAULT] 和 [oslo_messaging_rabbit] 部分，配置RabbitMQ 消息队列访问

[DEFAULT]
rpc_backend = rabbit
  
[oslo_messaging_rabbit]
rabbit_host = localhost
rabbit_userid = openstack
rabbit_password = 123456

在[DEFAULT] 和 [keystone_authtoken] 部分，Identity service 访问

[DEFAULT]
auth_strategy = keystone

[keystone_authtoken]
auth_uri = http://localhost:5000
auth_url = http://localhost:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = nova
password = 123456

在[DEFAULT] 部分，使用控制节点管理网络ip地址配置my_ip

[DEFAULT]
my_ip = 10.20.0.10

在 [DEFAULT]部分，使用控制节点管理网络ip地址配置 VNC proxy

[DEFAULT]
vncserver_listen = 10.20.0.10
vncserver_proxyclient_address = 10.20.0.10

在 [glance] 部分， 配置镜像服务位置

[glance]
host = openstack

在 [oslo_concurrency]部分，配置 lock 路径:
[oslo_concurrency]
lock_path = /var/lib/nova/tmp

在[DEFAULT]部分启用日志信息详细记录

[DEFAULT]
verbose = True
```

4. 同步数据库
```sh
# /bin/sh -c "nova-manage db sync" nova
```

5. 重启计算服务
```sh
# service nova-api restart
# service nova-cert restart
# service nova-consoleauth restart
# service nova-scheduler restart
# service nova-conductor restart
# service nova-novncproxy restart
```

6. 如果存在SQLite 数据库，则删除
```sh
   # rm -f /var/lib/nova/nova.sqlite
```


安装配置【计算节点】
------------------

1.安装nova
```sh
# apt-get install nova-compute sysfsutils -y
```

2.编辑文件 /etc/nova/nova.conf完成下面内容
```
在[DEFAULT]部分，启用和配置remote console访问:
[DEFAULT]
vnc_enabled = True
novncproxy_base_url = http://localhost:6080/vnc_auto.html

注意：如果通过浏览器访问，不能解析hostname controller ,则使用管理网络ip，代替controller
```

完成安装
-------

```
1.决定计算节点是否支持虚拟机的硬件加速：
    # egrep -c '(vmx|svm)' /proc/cpuinfo
    如果输出值是1或则比这更大，则不需要额外配置
    如果是0，计算节点不支持硬件加速，你必须配置libvirt 为QEMU ，代替KVM

    编辑文件/etc/nova/nova-compute.conf在 [libvirt]
        [libvirt]
        virt_type = qemu

2.重启计算服务
    # service nova-compute restart

3.如果存在SQLite 数据，则删除
    # rm -f /var/lib/nova/nova.sqlite
```


###验证安装【控制节点】

1.生效环境变量
```sh
# source admin-openrc.sh
```

2.目录服务组件来验证每个进程的成功创建和注册：
```sh
# nova service-list
+----+------------------+---------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host    | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+---------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-cert        | oneplus | internal | enabled | up    | 2015-11-23T05:29:24.000000 | -               |
| 2  | nova-consoleauth | oneplus | internal | enabled | up    | 2015-11-23T05:29:25.000000 | -               |
| 3  | nova-scheduler   | oneplus | internal | enabled | up    | 2015-11-23T05:29:25.000000 | -               |
| 4  | nova-conductor   | oneplus | internal | enabled | up    | 2015-11-23T05:29:22.000000 | -               |
| 5  | nova-compute     | oneplus | nova     | enabled | up    | 2015-11-23T05:29:21.000000 | -               |
+----+------------------+---------+----------+---------+-------+----------------------------+-----------------+
这个输出显示四个服务在控制节点启用，一个服务在计算节点
```

3.列出API endpoints  在 Identity service核实身份验证连接服务
```sh
# nova endpoints
+-----------+----------------------------------+
| glance    | Value                            |
+-----------+----------------------------------+
| id        | 8c803ad358aa4b5aab1a43961137ebdd |
| interface | internal                         |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://localhost:9292            |
+-----------+----------------------------------+
+-----------+-----------------------------------------------------------+
| nova      | Value                                                     |
+-----------+-----------------------------------------------------------+
| id        | 0d3e8d4679f84a79a14b66b46e316213                          |
| interface | public                                                    |
| region    | RegionOne                                                 |
| region_id | RegionOne                                                 |
| url       | http://localhost:8774/v2/87c4044b921a4ecda5461c2be14110a3 |
+-----------+-----------------------------------------------------------+
+-----------+----------------------------------+
| keystone  | Value                            |
+-----------+----------------------------------+
| id        | 07dcd275dc664c018049071c7d2ab559 |
| interface | public                           |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://localhost:5000/v2.0       |
+-----------+----------------------------------+
```

4.列出 镜像 在 Image service 目录验证连接 Image service:
```sh
# nova image-list
+--------------------------------------+---------------------+--------+--------+
| ID                                   | Name                | Status | Server |
+--------------------------------------+---------------------+--------+--------+
| 087252c1-838d-4645-977f-6e410ad051c6 | cirros-0.3.3-x86_64 | ACTIVE |        |
+--------------------------------------+---------------------+--------+--------+
```
