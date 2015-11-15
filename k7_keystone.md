## 创建租户、用户、角色


### 一、创建管理员租户、用户、角色

    1) 创建admin租户

    root@king:~# openstack project create --description "admin project" admin
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | admin project                    |
    | enabled     | True                             |
    | id          | aa3622f4a6ee4c379f4294b4492562b0 |
    | name        | admin                            |
    +-------------+----------------------------------+
    root@king:~#


    2) 创建admin用户

    root@king:~# openstack user create --password-prompt admin
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
    root@king:~#


    3) 创建admin角色

    root@king:~# `openstack role create admin`
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | d574ffdf0ef748138fb3ec2460d47d9e |
    | name  | admin                            |
    +-------+----------------------------------+
    root@king:~#


    4) 添加 admin 角色到 admin 租户和用户

    root@king:~# openstack role add --project admin --user admin admin
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | d574ffdf0ef748138fb3ec2460d47d9e |
    | name  | admin                            |
    +-------+----------------------------------+
    root@king:~#


### 二、创建一个service租户

    root@king:~# openstack project create --description "service project" service
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | service project                  |
    | enabled     | True                             |
    | id          | 255de1bf31434000a4824ef4aec84f36 |
    | name        | service                          |
    +-------------+----------------------------------+
    root@king:~#


### 三、创建非管理员demo租户

    1) 创建demo租户

    root@king:~# openstack project create --description "demo project" demo
    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | demo project                     |
    | enabled     | True                             |
    | id          | b8f1c26e2b3c447e8ba664fc999c6e70 |
    | name        | demo                             |
    +-------------+----------------------------------+
    root@king:~#


    2) 创建demo用户

    root@king:~# openstack user create --password-prompt demo
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
    root@king:~#


    3) 创建demo角色

    root@king:~# openstack role create user
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | 8b355da0962d4da692d05a60cec737a1 |
    | name  | user                             |
    +-------+----------------------------------+
    root@king:~#


    4) 添加 demo 角色到 demo 租户和用户

    root@king:~# openstack role add --project demo --user demo user
    +-------+----------------------------------+
    | Field | Value                            |
    +-------+----------------------------------+
    | id    | 8b355da0962d4da692d05a60cec737a1 |
    | name  | user                             |
    +-------+----------------------------------+
    root@king:~#
