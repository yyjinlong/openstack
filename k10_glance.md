## glance安装配置[控制节点]


## 配置准备

### 一、 创建database,完成下面步骤:

    1) 使用root用户登录

        root@king:~# mysql -uroot -p


    2) 创建glance 数据库

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

        MariaDB [(none)]>

    3) 授权

        MariaDB [(none)]>
        MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' identified by '5564155';
        Query OK, 0 rows affected (1.38 sec)

        MariaDB [(none)]> grant all privileges on glance.* to 'glance'@'%' identified by '5564155';
        Query OK, 0 rows affected (0.00 sec)
        MariaDB [(none)]>

    4) 退出数据库

        MariaDB [(none)]> Ctrl-C -- exit!
        Aborted


### 二、生效admin环境变量

    # source admin-openrc.sh


### 三、创建认证服务

    1) 创建glance用户

    root@king:~# openstack user create --password-prompt glance
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
    root@king:~#

    2) 添加admin角色到glance用户和service租户

    root@king:~# openstack role add --project service --user glance admin
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | d574ffdf0ef748138fb3ec2460d47d9e |
    | name  | admin                            |
    +-------+----------------------------------+
    root@king:~#

    3) 创建glance服务实例

    root@king:~# openstack service create --name glance \
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
    root@king:~#


### 四、创建镜像服务 API endpoint

    root@king:~# openstack endpoint create \
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
    root@king:~#


## 安装配置glance服务组件

### 一、安装glance

    # apt-get install glance python-glanceclient -y


### 二、编辑文件 /etc/glance/glance-api.conf，完成下面内容

    1) 在[database]部分，配置数据库访问

    [database]
    ...
    connection = mysql://glance:5564155@localhost/glance


    2) 在[keystone_authtoken]和[paste_deploy]部分, 配置Identity服务访问

    [keystone_authtoken]
    ...
    auth_uri = http://localhost:5000
    auth_url = http://localhost:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = glance
    password = 5564155

    [paste_deploy]
    ...
    flavor = keystone

    注: 注释掉其它[keystone_authtoken]部分


    3) 在[glance_store]部分, 配置本地文件系统存储和image文件路径

    [glance_store]
    ...
    default_store = file
    filesystem_store_datadir = /var/lib/glance/images/


    4) 在[DEFAULT]部分，配置noop禁用通知驱动，因为它只属于可选的遥测服务

    [DEFAULT]
    ...
    notification_driver = noop


    5) 在[DEFAULT]部分启用日志详细信息记录

    [DEFAULT]
    ...
    verbose = True


### 三、编辑 /etc/glance/glance-registry.conf 文件，完成下面内容


    1) 在[database]部分，配置数据库访问

    [database]
    ...
    connection = mysql://glance:5564155@localhost/glance


    2) 在[keystone_authtoken]和[paste_deploy]部分, 配置 Identity service 访问

    [keystone_authtoken]
    ...
    auth_uri = http://localhost:5000
    auth_url = http://localhost:35357
    auth_plugin = password
    project_domain_id = default
    user_domain_id = default
    project_name = service
    username = glance
    password = 5564155

    [paste_deploy]
    ...
    flavor = keystone

    注: 注释掉其它[keystone_authtoken]部分


    3) 在[DEFAULT]部分, 配置noopdriver禁用通知因为他们只属于遥测服务

    [DEFAULT]
    ...
    notification_driver = noop


    4) 方便排除，启用日志信息详细记录

    [DEFAULT]
    ...
    verbose = Tru

### 四、同步数据库

    # su -s /bin/sh -c "glance-manage db_sync" glance


## 四、完成安装

    1) 重启镜像服务

        # service glance-registry restart
        # service glance-api restart

    2） 如果存在SQLite 数据库则删除

        # rm -f /var/lib/glance/glance.sqlite

## 遇到问题

    ERROR: openstack No tenant with a name or ID of 'service' exists.
    原因没有创建service 租户

    Solution: 创建Service租户即可
    # openstack project create --description "Service Project" service
