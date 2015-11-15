## keystone安装与配置


### 创建数据库，并授权

    $ mysql -uroot -p


### 创建keystone数据库

    MariaDB [(none)]> CREATE DATABASE keystone;


### 对keystone授权

    MariaDB [(none)]>
    MariaDB [(none)]> grant all privileges on keystone.* to 'keystone'@'localhost' identified by '5564155';
    Query OK, 0 rows affected (0.00 sec)

    MariaDB [(none)]>
    MariaDB [(none)]> grant all privileges on keystone.* to 'keystone'@'%' identified by '5564155';
    Query OK, 0 rows affected (0.00 sec)

    注：上面的含义实现了，对keystone用户实现了，本地和远程都可以访问


    退出mysql
    MariaDB [(none)]> quit


### 生成token

    $ openssl rand -hex 10


### 安装keystone包


    1) 默认keystone服务监听端口5000 和 35357，尽管如此向导配置 Apache HTTP server 监听这些端口，
       为了避免端口冲突，安装后禁止开机启动keystone 服务。

        # echo "manual" > /etc/init/keystone.override


    2) 下载并安装keystone

        # apt-get install keystone \
                          python-openstackclient \
                          apache2 \
                          libapache2-mod-wsgi \
                          memcached \
                          python-memcache


    3) 编辑/etc/keystone/keystone.conf

        [DEFAULT]
        admin_token = 570f150cb897e793e58f

        verbose = True

        [database]
        connection = mysql://keystone:5564155@localhost/keystone

        补充：
        记得一定注释掉：
        connection=sqlite:////var/lib/keystone/keystone.db

        [memcache]
        servers = localhost:11211

        [token]
        provider = keystone.token.providers.uuid.Provider
        driver = keystone.token.persistence.backends.memcache.Token

        修改 [revoke] 部分, 配置  SQL revocation driver:
        driver = keystone.contrib.revoke.backends.sql.Revoke


### 填充keystone

    注：最好切到root执行,否则会同步不成干
    # sh -c "keystone-manage db_sync" keystone


### 配置 Apache HTTP server


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

### 完成安装

    1. 重启 Apache HTTP server:

        # service apache2 restart


    2. 如果存在 SQLite 数据库，则删除

        # rm -f /var/lib/keystone/keystone.db
