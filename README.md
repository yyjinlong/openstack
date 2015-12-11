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


安装ubuntu server14.04
----------------------

```
1) 网卡配置: 高级选项中的控制芯片、混杂模式 均不改.
   网卡1的界面名称选择: (仅主机(Host-Only)适配器), 界面名称选择vboxnet1 对应eth0
   网卡2的界面名称选择: (仅主机(Host-Only)适配器), 界面名称选择vboxnet2 对应eth1
   网卡3的连接方式选择: (仅主机(Host-Only)适配器), 界面名称选择vboxnet3 对应eth2
   网卡4的连接方式选择: 网络地址转换(NAT)   ----   对应eth3

2) 系统配置: 2 cpu 20G disk 2G memory

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

```sh
0) 安装过程中推荐使用root

1) 安装ubuntu-cloud-keyring
    apt-get install ubuntu-cloud-keyring

2) 添加kilo源
    echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu" \
         "trusty-updates/kilo main" > /etc/apt/sources.list.d/cloudarchive-kilo.list

3) 更新及源升级
    apt-get update && apt-get dist-upgrade
```


一、ssh安装
-----------

ssh是一种安全协议，主要用于给远程登录会话数据进行加密，保证数据传输的安全.

```sh
1. 安装ssh
apt-get install openssh-server

2. 查看ssh服务是否启动
ps -e | grep ssh
如果有sshd,说明ssh服务已经启动，如果没有启动，则输入如下命令服务就会启动
service ssh start

3. 修改配置文件
vim /etc/ssh/sshd_config
注: ssh默认端口号是22，可以修改（但不建议）。
PermitRootLogin without-password 把这一句注释掉，不然有设置密码的root用户就无法登陆。
PermitRootLogin yes 增加这一句，允许root用户登陆。
```


二、NTP安装
----------

```
ntp同步方式有两种：
1. 一种是同步远程网络服务器
2. 一种是同步局域网内的服务器。

如果是多节点安装：
1. 控制节点（controller）同步的是远程网络服务器
2. 计算节点和网络节点则是同步的控制节点

一) 控制节点
    ntp配置的思路：
    控制节点同步其它服务器，可以有多个。其它节点同步控制节点。

    1) 安装NTP
    apt-get install ntp

    2) 修改配置文件
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

    3) 重启NTP
    service ntp restart

    4) 验证安装
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


三、mysql(MariaDB)安装[控制节点]
-----------------------------------

```
问题导读
1. MariaDB与mysql的关系是什么？
2. 遇到Checking for corrupt, not cleanly closed and upgrade needing tables.该如何解决？

为什么产生MariaDB
    首先这里介绍一下，大家对MariaDB可能不太熟悉，MariaDB数据库管理系统是MySQL的一个分支,
    主要由开源社区在维护，采用GPL授权许可。开发这个分支的原因之一是：甲骨文公司收购了MySQL后,
    有将MySQL闭源的潜在风险，因此社区采用分支的方式来避开这个风险。

1. 安装
    apt-get install mariadb-server python-mysqldb -y

2. 创建文件/etc/mysql/conf.d/mysqld_openstack.cnf
    在[mysqld]部分,设置bind-address为控制节点管理网络ip地址,使能通过管理网络访问其它节点,并设置其他内容:

    # vim /etc/mysql/conf.d/mysqld_openstack.cnf
    [mysqld]
    bind-address = 0.0.0.0
    default-storage-engine = innodb
    innodb_file_per_table
    collation-server = utf8_general_ci
    init-connect = 'set names utf8'
    character-set-server = utf8

3. 重启mysql
    # service mysql restart
    输出如下信息
    * Stopping MariaDB database server mysqld                               [ OK ]
    * Starting MariaDB database server mysqld                               [ OK ]
    * Checking for corrupt, not cleanly closed and upgrade needing tables.
    这个只是个提示，告诉你在做什么。不管它。

4. 执行如下命令
    # mysql_secure_installation
    ........................
    Thanks for using MariaDB!
```


四、RabbitMQ安装[控制节点] 
---------------------------

```sh
1. 安装消息队列服务
    # apt-get install rabbitmq-server

2. 配置消息队列服务

    a) 添加 openstak 用户
    # rabbitmqctl add_user openstack 123456

    b) 禁止访问读写openstack user
    # rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```


五、keystone安装与配置
---------------------

```sh
1. 创建keystone数据库

    $ mysql -uroot -p
    MariaDB [(none)]> CREATE DATABASE keystone;
    MariaDB [(none)]> grant all privileges on keystone.* to 'keystone'@'localhost' identified by '123456';
    Query OK, 0 rows affected (0.00 sec)
    MariaDB [(none)]> grant all privileges on keystone.* to 'keystone'@'%' identified by '123456';
    Query OK, 0 rows affected (0.00 sec)
    退出mysql
    MariaDB [(none)]> quit
```

```sh
2. 安装keystone包

    1) 默认keystone服务监听端口5000 和 35357，尽管如此向导配置 Apache HTTP server 监听这些端口，
       为了避免端口冲突，安装后禁止开机启动keystone 服务。
       # echo "manual" > /etc/init/keystone.override

    2) 下载并安装keystone
       # apt-get install keystone python-openstackclient apache2 libapache2-mod-wsgi \
                         memcached python-memcache -y
```

```sh
3. 编辑/etc/keystone/keystone.conf

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

```sh
4. 填充keystone 注：最好切到root执行,否则会同步不成功

    # /bin/sh -c "keystone-manage db_sync" keystone
```

```
5. 配置 Apache HTTP server

    1) 修改配置文件 /etc/apache2/apache2.conf，配置ServerName选项为控制节点hostname
        ServerName localhost

    2) 创建/etc/apache2/sites-available/wsgi-keystone.conf 文件，添加如下内容
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

    3) 启用身份服务虚拟主机：
        # ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled

    4) 创建WSGI组件的目录结构：
        # mkdir -p /var/www/cgi-bin/keystone

    5) 下载复制WSGI组件到目录 /var/www/cgi-bin/keystone
        # curl http://git.openstack.org/cgit/openstack/keystone/plain/httpd/keystone.py?h=stable/kilo \
          | tee /var/www/cgi-bin/keystone/main /var/www/cgi-bin/keystone/admin

    6) 修改权限
        # chown -R keystone:keystone /var/www/cgi-bin/keystone
        # chmod 755 /var/www/cgi-bin/keystone/*
```

```sh
6. 完成安装
    1) 重启 Apache HTTP server:
    # service apache2 restart

    2) 如果存在 SQLite 数据库，则删除
    # rm -f /var/lib/keystone/keystone.db
```


六、创建服务实例和 API endpoint
------------------------------

```
1. 准备
    1）配置token
        # export OS_TOKEN=570f150cb897e793e58f
        格式：export OS_TOKEN=ADMIN_TOKEN
        注: ADMIN_TOKEN为keystone的配置文件里的admin_token的设置值

    2）配置endpoint url
        # export OS_URL=http://localhost:35357/v2.0

2. 创建服务实例和API endpoint

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

1. 创建管理员租户、用户、角色

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
    [pipeline:public_api], [pipeline:admin_api]和[pipeline:api_v3]部分.

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

9、作为demo用户,尝试列出用户不能执行admin-only CLI命令
    # openstack --os-auth-url http://localhost:5000 \
      --os-project-domain-id default --os-user-domain-id default \
      --os-project-name demo --os-username demo --os-auth-type password \
      user list
    Password:
    ERROR: openstack You are not authorized to perform the requested action:
           admin_required (HTTP 403) (Request-ID: req-f7d2441e-d029-4ea7-88e9-ae7797cca764)
```


九、创建openstack客户端环境变量脚本
----------------------------------

```sh
1、创建脚本

    创建admin和demo租户脚本:

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

2、加载客户端环境脚本
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


十、glance安装配置[控制节点]
-----------------------------

```sh
1. 创建glance 数据库
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

2. 生效admin环境变量
    # source admin-openrc.sh

3. 创建认证服务

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

4. 创建镜像服务API endpoint

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

5. 安装配置glance服务组件

    1) 安装glance
        # apt-get install glance python-glanceclient -y

    2) 编辑文件 /etc/glance/glance-api.conf，完成下面内容
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

    3) 编辑 /etc/glance/glance-registry.conf 文件，完成下面内容
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

    4) 同步数据库
        # /bin/sh -c "glance-manage db_sync" glance

    5) 完成安装
        a) 重启镜像服务
            # service glance-registry restart
            # service glance-api restart

        b） 如果存在SQLite 数据库则删除
            # rm -f /var/lib/glance/glance.sqlite

        遇到问题
            ERROR: openstack No tenant with a name or ID of 'service' exists.
            原因没有创建service 租户

            Solution: 创建Service租户即可
            # openstack project create --description "Service Project" service
```


十一、glance安装验证
-------------------

```sh
1. 在每一个客户端脚本,配置镜像服务客户端使用 API version 2.0:
    echo "export OS_IMAGE_API_VERSION=2" | tee -a admin-openrc.sh demo-openrc.sh

2. 生效admin环境变量
    source admin-openrc.sh

3. 创建一个临时目录
    # mkdir /tmp/images

4. 下载镜像到当前目录
    # wget -P /tmp/images http://download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img

5. 上传镜像到glance，镜像使用qcow2 格式，镜像使用格式
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

6. 确认上传成功，核实属性
    # glance image-list
    +--------------------------------------+---------------------+
    | ID                                   | Name                |
    +--------------------------------------+---------------------+
    | 087252c1-838d-4645-977f-6e410ad051c6 | cirros-0.3.3-x86_64 |
    +--------------------------------------+---------------------+
```


十二、nova
---------

```sh
1. 创建nova数据库及授权

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

2. 生效admin用户
    # source admin-openrc.sh 

3. 创建keystone认证，完成下面内容

    a) 创建nova用户
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

    b) 添加admin角色到nova用户
        # openstack role add --project service --user nova admin
        +-------+----------------------------------+
        | Field | Value                            |
        +-------+----------------------------------+
        | id    | e5f57dabbeef4447be2aa94fc99fd45b |
        | name  | admin                            |
        +-------+----------------------------------+

    c) 创建nova 服务实例
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

4. 创建nova服务APIendpoint
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

5. 安装配置计算控制节点组件【控制节点】

    a) 安装nova
        # apt-get install nova-api nova-cert nova-conductor nova-consoleauth \
          nova-novncproxy nova-scheduler python-novaclient -y

    b) 修改配置/etc/nova/nova.conf文件，完成下面内容
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
        host = openstack  #注: openstack是我的主机名

        在 [oslo_concurrency]部分，配置 lock 路径:
        [oslo_concurrency]
        lock_path = /var/lib/nova/tmp

        在[DEFAULT]部分启用日志信息详细记录

        [DEFAULT]
        verbose = True

6. 同步数据库
    # /bin/sh -c "nova-manage db sync" nova

7. 重启计算服务
    # service nova-api restart
    # service nova-cert restart
    # service nova-consoleauth restart
    # service nova-scheduler restart
    # service nova-conductor restart
    # service nova-novncproxy restart

8. 如果存在SQLite 数据库，则删除
    # rm -f /var/lib/nova/nova.sqlite
```


安装配置[计算节点]
------------------

```sh
1.安装nova
# apt-get install nova-compute sysfsutils -y

2.编辑文件 /etc/nova/nova.conf完成下面内容
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


验证安装[控制节点]
------------------

```sh
1.生效环境变量
    # source admin-openrc.sh

2.目录服务组件来验证每个进程的成功创建和注册：
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

3.列出API endpoints  在 Identity service核实身份验证连接服务
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

4.列出 镜像 在 Image service 目录验证连接 Image service:
    # nova image-list
    +--------------------------------------+---------------------+--------+--------+
    | ID                                   | Name                | Status | Server |
    +--------------------------------------+---------------------+--------+--------+
    | 087252c1-838d-4645-977f-6e410ad051c6 | cirros-0.3.3-x86_64 | ACTIVE |        |
    +--------------------------------------+---------------------+--------+--------+
```


十三、neutron安装
----------------

```sh
1. 创建数据及授权
    MariaDB [(none)]> CREATE DATABASE neutron;
    MariaDB [(none)]> show databases;
    +--------------------+
    | Database           |
    +--------------------+
    | information_schema |
    | glance             |
    | keystone           |
    | mysql              |
    | neutron            |
    | nova               |
    | performance_schema |
    +--------------------+
    7 rows in set (0.02 sec)

    MariaDB [(none)]>
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' identified by '123456';
    MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' identified by '123456';
    MariaDB [(none)]> Ctrl-C -- exit!
    Aborted
```

```sh
2. 生效环境变量
    source admin-openrc.sh
```

```sh
3. 创建网络用户并授权
    a) 创建neutron用户
        openstack user create --password-prompt neutron
        User Password:
        Repeat User Password:
        +----------+----------------------------------+
        | Field    | Value                            |
        +----------+----------------------------------+
        | email    | None                             |
        | enabled  | True                             |
        | id       | b183ab0f376a4b52807d0491d1261fc9 |
        | name     | neutron                          |
        | username | neutron                          |
        +----------+----------------------------------+

    b) 创建admin角色到neutron用户
        openstack role add --project service --user neutron admin
        +-------+----------------------------------+
        | Field | Value                            |
        +-------+----------------------------------+
        | id    | af30e3c420714279aa340a4a3863011f |
        | name  | admin                            |
        +-------+----------------------------------+

    c) 创建neutron 服务实例
        # openstack service create --name neutron \
        >   --description "OpenStack Networking" network
        +-------------+----------------------------------+
        | Field       | Value                            |
        +-------------+----------------------------------+
        | description | OpenStack Networking             |
        | enabled     | True                             |
        | id          | 5c0180374c01467e99643f3b8a87c7e4 |
        | name        | neutron                          |
        | type        | network                          |
        +-------------+----------------------------------+
```

```sh
4. 创建网络服务API endpoint
    # openstack endpoint create \
      --publicurl http://localhost:9696 \
      --adminurl http://localhost:9696 \
      --internalurl http://localhost:9696 \
      --region RegionOne \
      network
    +--------------+----------------------------------+
    | Field        | Value                            |
    +--------------+----------------------------------+
    | adminurl     | http://localhost:9696            |
    | id           | 8ed5dd97de054494955dd4615f624891 |
    | internalurl  | http://localhost:9696            |
    | publicurl    | http://localhost:9696            |
    | region       | RegionOne                        |
    | service_id   | 5c0180374c01467e99643f3b8a87c7e4 |
    | service_name | neutron                          |
    | service_type | network                          |
    +--------------+----------------------------------+
```

```sh
5. 安装新的网络组件
    apt-get install neutron-server neutron-plugin-ml2 python-neutronclient -y
```

```sh
6. 配置网络服务组件

编辑文件 /etc/neutron/neutron.conf完成下面内容
    
    a) 在[database]部分,配置数据库访问
        connection = mysql://neutron:123456@localhost/neutron

    b) 在[DEFAULT]和[oslo_messaging_rabbit]部分,配置RabbitMQ 消息队列服务
        [DEFAULT]
        rpc_backend = rabbit
     
        [oslo_messaging_rabbit]
        rabbit_host = openstack
        rabbit_userid = openstack
        rabbit_password = 123456

    c) 在[DEFAULT]和[keystone_authtoken]部分,配置认证访问
        [DEFAULT]
        auth_strategy = keystone
         
        [keystone_authtoken]
        auth_uri = http://localhost:5000
        auth_url = http://localhost:35357
        auth_plugin = password
        project_domain_id = default
        user_domain_id = default
        project_name = service
        username = neutron
        password = 123456

    d) 在[DEFAULT]部分,启用Modular Layer 2 (ML2) plug-in,路由服务,和 overlapping IP addresses:
        [DEFAULT]
        core_plugin = ml2
        service_plugins = router
        allow_overlapping_ips = True

    e) 在[DEFAULT]和[nova]部分,配置计算节点网络拓扑变化通知
        [DEFAULT]
        notify_nova_on_port_status_changes = True
        notify_nova_on_port_data_changes = True
        nova_url = http://localhost:8774/v2
        
        [nova]
        auth_url = http://localhost:35357
        auth_plugin = password
        project_domain_id = default
        user_domain_id = default
        region_name = RegionOne
        project_name = service
        username = nova
        password = 123456

    f) 启用日志信息详细记录
        [DEFAULT]
        verbose = True
```

```sh
7. 配置Modular Layer 2 (ML2) plug-in

ML2插件使用e Open vSwitch (OVS) 机制作为实例的虚拟网络架构，尽管如此，计算节点不需要ovs组件，因为它不处理实例的网络

编辑文件 /etc/neutron/plugins/ml2/ml2_conf.ini完成下面内容
    a) 在[ml2]部分,启用flat,VLAN,generic routing encapsulation (GRE),
        和virtual extensible LAN (VXLAN)网络类型驱动,GRE租户网络,和OVS机制驱动:
        [ml2]
        type_drivers = flat,vlan,gre,vxlan
        tenant_network_types = gre
        mechanism_drivers = openvswitch
        注意: 一旦配置ML2插件,如果改变type_drivers值的话,会导致数据库不一致

    b) 在[ml2_type_gre]部分,配置隧道标识符id的范围
        [ml2_type_gre]
        tunnel_id_ranges = 1:1000

    c) 在[securitygroup]部分,启用security groups和ipset并配置OVS iptables firewall驱动:
        [securitygroup]
        enable_security_group = True
        enable_ipset = True
        firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

```sh
8. 重新配置网络【控制节点】
编辑文件 /etc/nova/nova.conf完成下面内容
    
    a) 在[DEFAULT]部分,配置API和驱动
        [DEFAULT]
        network_api_class = nova.network.neutronv2.api.API
        security_group_api = neutron
        linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
        firewall_driver = nova.virt.firewall.NoopFirewallDriver

    b) 在[neutron]部分,配置访问参数
        [neutron]
        url = http://localhost:9696
        auth_strategy = keystone
        admin_auth_url = http://localhost:35357/v2.0
        admin_tenant_name = service
        admin_username = neutron
        admin_password = 123456
```

```sh
9. 完成安装
    a) 同步数据库
        /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
          --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
        数据同步脚本是根据配置文件特别是插件的配置来进行同步的

    b) 重启计算服务
        service nova-api restart

    c) 重启网络服务
        service neutron-server restart
```

```sh
10. 验证安装

    a) 生效环境变量
        source admin-openrc.sh

    b) 列出创建成功的neutron-server 进程
        # neutron ext-list
        +-----------------------+-----------------------------------------------+
        | alias                 | name                                          |
        +-----------------------+-----------------------------------------------+
        | security-group        | security-group                                |
        | l3_agent_scheduler    | L3 Agent Scheduler                            |
        | net-mtu               | Network MTU                                   |
        | ext-gw-mode           | Neutron L3 Configurable external gateway mode |
        | binding               | Port Binding                                  |
        | provider              | Provider Network                              |
        | agent                 | agent                                         |
        | quotas                | Quota management support                      |
        | subnet_allocation     | Subnet Allocation                             |
        | dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
        | l3-ha                 | HA Router extension                           |
        | multi-provider        | Multi Provider Network                        |
        | external-net          | Neutron external network                      |
        | router                | Neutron L3 Router                             |
        | allowed-address-pairs | Allowed Address Pairs                         |
        | extraroute            | Neutron Extra Route                           |
        | extra_dhcp_opt        | Neutron Extra DHCP opts                       |
        | dvr                   | Distributed Virtual Router                    |
        +-----------------------+-----------------------------------------------+
```


十四、安装配置网络[网络节点]
-----------------------------

```sh
1. 配置准备
    a) 编辑文件 /etc/sysctl.conf 添加下面内容
        net.ipv4.ip_forward=1
        net.ipv4.conf.all.rp_filter=0
        net.ipv4.conf.default.rp_filter=0

    b) 使其生效
        sysctl -p
```

```sh
2. 安全网络组件
    apt-get install neutron-plugin-ml2 neutron-plugin-openvswitch-agent \
      neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent -y
```

```sh
3. 配置网络通用组件
    网络通用组件包括认证机制、消息队列和插件
```

```sh
4. 配置Modular Layer 2 (ML2) 插件
    ML2插件使用Open vSwitch (OVS) 机制构建实例的虚拟网络框架
    编辑文件 /etc/neutron/plugins/ml2/ml2_conf.ini，完成下面内容

    1) 在[ml2_type_flat]部分, 配置external flat提供的网络
    [ml2_type_flat]
    flat_networks = external

    2) 在[ovs]部分,启用tunnels,配置本地tunnel endpoint和映射外部flat私有网络到br-ex外部网桥
    [ovs]
    local_ip = 192.168.4.10
    bridge_mappings = external:br-ex

    说明: local_ip为网络节点隧道网络ip地址.

    3) 在[agent]部分,启用GRE隧道
    [agent]
    tunnel_types = gre
```

```sh
5. 配置Layer-3(L3)代理
    Layer-3 (L3) 提供路由服务为虚拟网络
    编辑文件/etc/neutron/l3_agent.ini完成下面内容
    1) 在[DEFAULT]部分,配置网卡驱动和外部网桥并启用删除路由命名空间失效设置
        [DEFAULT]
        interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
        external_network_bridge =
        router_delete_namespaces = True

        注意: external_network_bridge 是没有值的

    2) 在[DEFAULT]部分，启用日志详细信息记录
        [DEFAULT]
        verbose = True
```

```sh
6. 配置DHCP代理
    DHCP代理为虚拟网络提供DHCP服务
    编辑文件/etc/neutron/dhcp_agent.ini完成下面内容
    1) 在[DEFAULT]部分,配置接口和dhcp驱动并启用失效删除DHCP命令空间
        [DEFAULT]
        interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
        dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
        dhcp_delete_namespaces = True

    2) 启用日志详细信息记录
        [DEFAULT]
        verbose = True
```

```sh
7. 配置metadata代理
    编辑文件/etc/neutron/metadata_agent.ini完成下面内容
    [DEFAULT]
    auth_uri = http://localhost:5000
    auth_url = http://localhost:35357
    auth_region = RegionOne
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = neutron
    password = 123456

    在[DEFAULT]部分,配置metadata客户端
    # IP address used by Nova metadata server
    nova_metadata_ip = openstack

    在[DEFAULT]部分,配置元数据代理共享密码
    [DEFAULT]
    metadata_proxy_shared_secret = METADATA_SECRET

    启用日志详细信息记录
    [DEFAULT]
    verbose = True
```

```sh
8. 在控制节点,编辑文件/etc/nova/nova.conf,添加下面内容
    [neutron]
    service_metadata_proxy = True
    metadata_proxy_shared_secret = METADATA_SECRET
```

```sh
9. 在控制节点,重启Compute API服务
    service nova-api restart
```

```sh
10. 配置Open vSwitch (OVS)服务[网络节点]
    1) 重启OVS 服务:
        service openvswitch-switch restart

    2) 添加外部网桥
        ovs-vsctl add-br br-ex

    3) 添加混杂模式网卡到br-ex
        ovs-vsctl add-port br-ex eth1

        ip link set br-ex up
        ip addr add 172.16.0.10/24 dev br-ex
```

```sh
11. 完成安装,重启网络
    service neutron-plugin-openvswitch-agent restart
    service neutron-l3-agent restart
    service neutron-dhcp-agent restart
    service neutron-metadata-agent restart
```

```sh
12. 验证安装【控制节点】
    1) 生成环境变量
        root@openstack:~# source admin-openrc.sh

    2) 列出创建成功的neutron代理
        root@openstack:~# neutron agent-list
        +--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
        | id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
        +--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
        | 266eb6b5-b5fa-429d-b952-52c388173f4f | Metadata agent     | openstack | :-)   | True           | neutron-metadata-agent    |
        | 2ce4deee-30f0-45b3-9984-10e66174adba | L3 agent           | openstack | :-)   | True           | neutron-l3-agent          |
        | 3c8a405b-0adb-4160-9a40-54025f43d37a | Open vSwitch agent | openstack | :-)   | True           | neutron-openvswitch-agent |
        | 5e95879e-66b4-4dca-a300-d86503503128 | DHCP agent         | openstack | :-)   | True           | neutron-dhcp-agent        |
        +--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
```

十五、安装配置[计算节点]
-----------------------

```sh
1.在安装配置openstack网络，你必须配置一定的内核网络参数.
    1) 编辑文件 /etc/sysctl.conf, 追加
        net.bridge.bridge-nf-call-iptables=1
        net.bridge.bridge-nf-call-ip6tables=1

    2) 生效
        sysctl -p
```

```sh
2. 安装网络组建
    apt-get install neutron-plugin-ml2 neutron-plugin-openvswitch-agent
```

```sh
3. 配置Open vSwitch (OVS)服务
    重启ovs服务
    service openvswitch-switch restart
```

```sh
4. 完成安装
    1) 重启计算服务
        service nova-compute restart

    2) 重启Open vSwitch (OVS) 代理
        service neutron-plugin-openvswitch-agent restart
```

```sh
5. 验证安装[控制节点]
    1) 生效环境变量
        source admin-openrc.sh

    2) 列出创建成功的neutron 代理
    # neutron agent-list
    +--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
    | id                                   | agent_type         | host      | alive | admin_state_up | binary                    |
    +--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
    | 266eb6b5-b5fa-429d-b952-52c388173f4f | Metadata agent     | openstack | :-)   | True           | neutron-metadata-agent    |
    | 2ce4deee-30f0-45b3-9984-10e66174adba | L3 agent           | openstack | :-)   | True           | neutron-l3-agent          |
    | 3c8a405b-0adb-4160-9a40-54025f43d37a | Open vSwitch agent | openstack | :-)   | True           | neutron-openvswitch-agent |
    | 5e95879e-66b4-4dca-a300-d86503503128 | DHCP agent         | openstack | :-)   | True           | neutron-dhcp-agent        |
    +--------------------------------------+--------------------+-----------+-------+----------------+---------------------------+
```

十六、实例化网络
---------------

```sh
1. 创建外部网络[控制节点]
    1) 生效环境变量
        source admin-openrc.sh

    2) 共享网络
    neutron net-create ext-net --shared --router:external  --provider:physical_network external --provider:network_type flat
```

```sh
2. 创建外网上的子网
    neutron subnet-create ext-net 172.16.0.0/24 --name ext-subnet \
    --allocation-pool start=172.16.0.100,end=172.16.0.200 \
    --disable-dhcp --gateway 172.16.0.1
```

```sh
3. 创建租户网络
    1) 生效环境变量
        source admin-openrc.sh

    2) 创建租户网络
        neutron net-create demo-net
        Created a new network:
        +-----------------+--------------------------------------+
        | Field           | Value                                |
        +-----------------+--------------------------------------+
        | admin_state_up  | True                                 |
        | id              | 760ca325-c244-4f10-9cf1-14d57492be92 |
        | mtu             | 0                                    |
        | name            | demo-net                             |
        | router:external | False                                |
        | shared          | False                                |
        | status          | ACTIVE                               |
        | subnets         |                                      |
        | tenant_id       | cba4341c6c7f486caed41e984a388599     |
        +-----------------+--------------------------------------+
```

```sh
4. 创建租户网络子网

    # neutron subnet-create demo-net 192.168.4.10/24 \
      --name demo-subnet --gateway 192.168.4.1
    Created a new subnet:
    +-------------------+--------------------------------------------------+
    | Field             | Value                                            |
    +-------------------+--------------------------------------------------+
    | allocation_pools  | {"start": "192.168.4.2", "end": "192.168.4.254"} |
    | cidr              | 192.168.4.0/24                                   |
    | dns_nameservers   |                                                  |
    | enable_dhcp       | True                                             |
    | gateway_ip        | 192.168.4.1                                      |
    | host_routes       |                                                  |
    | id                | ea76cd29-2861-4f5c-8b3b-615662779ac9             |
    | ip_version        | 4                                                |
    | ipv6_address_mode |                                                  |
    | ipv6_ra_mode      |                                                  |
    | name              | demo-subnet                                      |
    | network_id        | 760ca325-c244-4f10-9cf1-14d57492be92             |
    | subnetpool_id     |                                                  |
    | tenant_id         | cba4341c6c7f486caed41e984a388599                 |
    +-------------------+--------------------------------------------------+
```

```sh
5. 创建租户路由，并附加外网和租户网络到路由
    # neutron router-create demo-router
    Created a new router:
    +-----------------------+--------------------------------------+
    | Field                 | Value                                |
    +-----------------------+--------------------------------------+
    | admin_state_up        | True                                 |
    | external_gateway_info |                                      |
    | id                    | 26bc67d7-cbcd-4383-8d9a-f7add2e4b588 |
    | name                  | demo-router                          |
    | routes                |                                      |
    | status                | ACTIVE                               |
    | tenant_id             | cba4341c6c7f486caed41e984a388599     |
    +-----------------------+--------------------------------------+
```

```sh
6. 连接路由器到租户网络
    # neutron router-interface-add demo-router demo-subnet
    Added interface 12bfda13-6e9d-41f7-8af6-c384572c985e to router demo-router.
```

```sh
7. 连接路由器到外部网络通过设置为网关
    # neutron router-gateway-set demo-router ext-net
    Set gateway for router demo-router
```

```sh
8. 安装验证
    # ping 172.16.0.100
    PING 172.16.0.100 (172.16.0.100) 56(84) bytes of data.
    64 bytes from 172.16.0.100: icmp_seq=1 ttl=64 time=0.308 ms
    64 bytes from 172.16.0.100: icmp_seq=2 ttl=64 time=0.116 ms
    ^C
```

十七、创建实例
--------------

```sh
1. 生效认证

    # source demo-openrc.sh

2. 生成并添加一个密钥对

    # nova keypair-add demo-key
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEAuCZ6Qc7GAbnQ+03qCxmo0cxWXHrDb1kYd+svdcWjs1u5Whm9
    62kOSbu+8VFJ/5ESQrWHsE0BFPWfgUG38dSodATpPp7wtL758qnE17hwN9Fp8Mw5
    Q6FPfd2juvrkBev2nDp2Jb4mJpMFuZubjwlhUpm2oDGPkMsYXVxD90aww3NARpY+
    dV2/SwoQBQ2m3HtbyMgVa12Pb9Ixc6SJYZarwEd0JB+C8Pir2n+bW9h4TpqTf4Xi
    OWL5pfssJURxTmsS7kgRgi9DpH5zhvhxxitqwHF8vAJKsp4JksM/LWH1ehXnjeGt
    3/ZJIEn4MNzwwFFe+cEOjkTv37Ij/istBHJ2iQIDAQABAoIBAAHntAAWSYofCABx
    j+hJfaud947BXmA6hbxH3JfVUZo7arF57rMOxS0SGimY87EHKS8zfZHfWhGDcQD/
    Uw3Xa1635knVjxvvldpi0zyAFfkd24C4PCds9cuRjW4TxmQhSs3W9P3y96YSg06m
    Q3e5Wx5lpLQHjzqqPzhIChP20UFUXWnED5cDmKsvqoSfMJwyMxNgZkEbIpgJ7b9F
    EA8KMXgxaXsbdHp0KxcxaldLSGMO7UtZqKn+W6xdaiC1QzkjhE4fGsoQsvjBmqU5
    KylA5pMS1y+hvkPJVuJb8Ou6XUhux/g2xG7ITaY7gQTdNk7W3CGyd28FzcLFZF3W
    cDLERJkCgYEA2pAnka6KnnneoB/RWKBWc4i6xR+6gD1Q3jTb78z7Te9vpIWTHbOd
    RTkqFS11lljyFKpIkFPHGiGb83nlOotuHfQy96zo0UAd3uADBqWc9wIYt0wQ6+3W
    vZZjoB7dRBnJYpAgW2l7QLmvSfldwAhbhm4wWggB6r6jnB9BEEOPHAMCgYEA17FY
    Q2c58zY8KY6Aj2kl7/Y+id/hGJI2pRr8NFmb7efrkX/9EjTOW4s+SkFoSgsP0cvR
    U4nZKsKjwU63mAasXVNqFyA6ulaRIxDAaXUcfNCxBvVBdNiP6q8JDK4goTd+9qUf
    eSVmevuw3OXaH1SX1tMUbny7jNde50/ijBABC4MCgYEAhrLtEAWn/L9TCxBQ7vPy
    E8YihTZmtH4VhrzBB2snPgLgpV6FKnr15CG049RecchjeYTwr7JSNLKd8FIhihFA
    Tkmf17DC06NWRXN9qe0Lbdfm76B7lUvBWpqCz7310/CogowcxPmfMma9tzNuKdl8
    vr7OIc5pkAjpwGAqsyFP440CgYAQrtHh3MEZs68xk6kT7pEVn1k09tEFQoHhgVXS
    gr/RxedtiJW9a8IuSHXX7nkviO1/T6FwMbBPY2ChGgKPSqzYRxRkl4STVxDAwpHv
    VjSO3uFiZWPbsshm4YT0qx8w+Qbj8t+dUiw8BO2oGEsnszZPUmI5LYKgISRhBcfD
    B5XdGwKBgE6D40ZwKQAYSM692Wz/y6VIJQ2eE6ta9Vf69vwDzo/6NoFJiHx0oNWN
    chZmY16g7b+HLHvV3fnQM5dZaOwBHdBKECHcOmKkhBmg+7R6HFqkXkF/cBKcqbs4
    Zkgyiv5BbJcVn7V35b3uG2LClxGjIaPkvCvWfGrEC5YVwtP10xZQ
    -----END RSA PRIVATE KEY-----

3. 验证密钥对

    # nova keypair-list
    +----------+-------------------------------------------------+
    | Name     | Fingerprint                                     |
    +----------+-------------------------------------------------+
    | demo-key | 3b:1b:77:ff:6d:86:e7:d0:20:2e:1a:59:e5:27:b8:91 |
    +----------+-------------------------------------------------+
    root@openstack:~#
```

```sh
4. 列出flavors

    # nova flavor-list
    +----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
    | ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
    +----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
    | 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
    | 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
    | 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
    | 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
    | 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
    +----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
```

```sh
5. 列出镜像

    # nova image-list
    +--------------------------------------+---------------------+--------+--------+
    | ID                                   | Name                | Status | Server |
    +--------------------------------------+---------------------+--------+--------+
    | ad40689f-ae13-4152-9b9d-f2ff3f427e53 | cirros-0.3.3-x86_64 | ACTIVE |        |
    +--------------------------------------+---------------------+--------+--------+
```

```sh
6. 列出网络

    # neutron net-list
    +--------------------------------------+----------+-----------------------------------------------------+
    | id                                   | name     | subnets                                             |
    +--------------------------------------+----------+-----------------------------------------------------+
    | d1c2a9b4-9def-4d76-8bdb-c441dbc64659 | ext-net  | d4942fe3-0a6e-448b-ba35-2f5a856cc885 172.16.0.0/24  |
    | 760ca325-c244-4f10-9cf1-14d57492be92 | demo-net | ea76cd29-2861-4f5c-8b3b-615662779ac9 192.168.4.0/24 |
    +--------------------------------------+----------+-----------------------------------------------------+
```


```sh
7. 创建实例

    # nova boot --flavor m1.tiny --image cirros-0.3.3-x86_64 --nic net-id=760ca325-c244-4f10-9cf1-14d57492be92 \
      --security-group default --key-name demo-key demo-instance1
    +--------------------------------------+------------------------------------------------------------+
    | Property                             | Value                                                      |
    +--------------------------------------+------------------------------------------------------------+
    | OS-DCF:diskConfig                    | MANUAL                                                     |
    | OS-EXT-AZ:availability_zone          | nova                                                       |
    | OS-EXT-STS:power_state               | 0                                                          |
    | OS-EXT-STS:task_state                | scheduling                                                 |
    | OS-EXT-STS:vm_state                  | building                                                   |
    | OS-SRV-USG:launched_at               | -                                                          |
    | OS-SRV-USG:terminated_at             | -                                                          |
    | accessIPv4                           |                                                            |
    | accessIPv6                           |                                                            |
    | adminPass                            | 7RQ3rU9XLEq2                                               |
    | config_drive                         |                                                            |
    | created                              | 2015-12-08T23:25:59Z                                       |
    | flavor                               | m1.tiny (1)                                                |
    | hostId                               |                                                            |
    | id                                   | 8c33b003-29b9-4d24-b3aa-a5c1c4101da0                       |
    | image                                | cirros-0.3.3-x86_64 (ad40689f-ae13-4152-9b9d-f2ff3f427e53) |
    | key_name                             | demo-key                                                   |
    | metadata                             | {}                                                         |
    | name                                 | demo-instance1                                             |
    | os-extended-volumes:volumes_attached | []                                                         |
    | progress                             | 0                                                          |
    | security_groups                      | default                                                    |
    | status                               | BUILD                                                      |
    | tenant_id                            | cba4341c6c7f486caed41e984a388599                           |
    | updated                              | 2015-12-08T23:25:59Z                                       |
    | user_id                              | 019183c1dd2644138131ef0e6d502748                           |
    +--------------------------------------+------------------------------------------------------------+
```

```sh
8. 列出实例

    # nova list
    +--------------------------------------+----------------+---------+------------+-------------+----------------------+
    | ID                                   | Name           | Status  | Task State | Power State | Networks             |
    +--------------------------------------+----------------+---------+------------+-------------+----------------------+
    | 8c33b003-29b9-4d24-b3aa-a5c1c4101da0 | demo-instance1 | SHUTOFF | -          | Shutdown    | demo-net=192.168.4.3 |
    +--------------------------------------+----------------+---------+------------+-------------+----------------------+
```

```sh
9. 通过浏览器访问实例

    # nova get-vnc-console demo-instance1 novnc
    +-------+--------------------------------------------------------------------------------+
    | Type  | Url                                                                            |
    +-------+--------------------------------------------------------------------------------+
    | novnc | http://localhost:6080/vnc_auto.html?token=15069eef-8d54-4035-be8f-8500c3452e34 |
    +-------+--------------------------------------------------------------------------------+
```


十八、安装horizon(dashboard)
---------------------------

```sh
1. 安装horizon
apt-get install -y openstack-dashboard
```

```sh
2. 登陆Dashboard
    获取当前生效的环境变量的用户名和密码, 当前生效的环境变量脚本: demo-openrc.sh
    用户名为: demo
    密码为: 123456
    访问地址: http://10.20.0.10/horizon
```

```sh
3. 去掉ubuntu主题
    打开Dashboard后，我们发现是ubuntu的主题，感觉有些不爽，那么我们将这个主题去掉:
    apt-get remove --purge openstack-dashboard-ubuntu-theme
    之后，再刷新Dashboard的地址就变成openstack的了。
```

十九、nova启动虚拟机并通过浏览器访问
-----------------------------------

```sh
1) 启动虚拟机：
    root@openstack:~# nova list
    +--------------------------------------+----------------+---------+------------+-------------+----------------------+
    | ID                                   | Name           | Status  | Task State | Power State | Networks             |
    +--------------------------------------+----------------+---------+------------+-------------+----------------------+
    | 8c33b003-29b9-4d24-b3aa-a5c1c4101da0 | demo-instance1 | SHUTOFF | -          | Shutdown    | demo-net=192.168.4.3 |
    +--------------------------------------+----------------+---------+------------+-------------+----------------------+
    root@openstack:~# nova start demo-instance1
    Request to start server demo-instance1 has been accepted.
    root@openstack:~#
    root@openstack:~# nova list
    +--------------------------------------+----------------+--------+------------+-------------+----------------------+
    | ID                                   | Name           | Status | Task State | Power State | Networks             |
    +--------------------------------------+----------------+--------+------------+-------------+----------------------+
    | 8c33b003-29b9-4d24-b3aa-a5c1c4101da0 | demo-instance1 | ACTIVE | -          | Running     | demo-net=192.168.4.3 |
    +--------------------------------------+----------------+--------+------------+-------------+----------------------+
    root@openstack:~#

2) 通过浏览器访问:
    root@openstack:~# nova get-vnc-console demo-instance1 novnc
    +-------+--------------------------------------------------------------------------------+
    | Type  | Url                                                                            |
    +-------+--------------------------------------------------------------------------------+
    | novnc | http://localhost:6080/vnc_auto.html?token=f5e105c6-0d5f-417d-b4b3-4d19ad832804 |
    +-------+--------------------------------------------------------------------------------+

    将localhost换成10.20.0.10,之后贴到浏览器的地址栏里访问:
    http://10.20.0.10:6080/vnc_auto.html?token=f5e105c6-0d5f-417d-b4b3-4d19ad832804
```
