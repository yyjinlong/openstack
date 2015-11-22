## 创建服务实例和 API endpoint


###一、准备


####1）配置token

        root@king:~# export OS_TOKEN=570f150cb897e793e58f
        格式：export OS_TOKEN=ADMIN_TOKEN
        注：ADMIN_TOKEN为keystone的配置文件里的admin_token的设置值


#####2）配置endpoint url

        root@king:~# export OS_URL=http://localhost:35357/v2.0


###二、创建服务实例和API endpoint


####1）创建Identity实例服务

        root@king:~# openstack service create \
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
        root@king:~#


####2）创建实例服务

        root@king:~# openstack endpoint create \
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
        root@king:~#
