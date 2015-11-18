## 创建openstack客户端环境变量脚本


###一、创建脚本

    创建admin 和 demo 租户脚本

    1) 编辑admin-openrc.sh 文件，添加如下内容

    export OS_PROJECT_DOMAIN_ID=default
    export OS_USER_DOMAIN_ID=default
    export OS_PROJECT_NAME=admin
    export OS_TENANT_NAME=admin
    export OS_USERNAME=admin
    export OS_PASSWORD=5564155
    export OS_AUTH_URL=http://localhost:35357/v3
    export OS_REGION_NAME=RegionOne


    2) 编辑文件demo-openrc.sh ，添加如下内容

    export OS_PROJECT_DOMAIN_ID=default
    export OS_USER_DOMAIN_ID=default
    export OS_PROJECT_NAME=demo
    export OS_TENANT_NAME=demo
    export OS_USERNAME=demo
    export OS_PASSWORD=5564155
    export OS_AUTH_URL=http://localhost:5000/v3
    export OS_REGION_NAME=RegionOne


###二、加载客户端环境脚本

    1) 加载admin-openrc.sh环境变量

        # source admin-openrc.sh

    2) 请求认证令牌

        # openstack token issue

        root@king:~# openstack token issue
        +------------+----------------------------------+
        | Field      | Value                            |
        +------------+----------------------------------+
        | expires    | 2015-11-17T16:34:46.403966Z      |
        | id         | 077e1df72cd740628861552b5085d777 |
        | project_id | aa3622f4a6ee4c379f4294b4492562b0 |
        | user_id    | 22488b4d6ec3414aac16de978337909d |
        +------------+----------------------------------+
        root@king:~#
