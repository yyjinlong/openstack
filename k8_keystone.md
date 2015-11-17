## 验证keystone安装部署


### 一、为了安全，禁用临时token

    编辑 /etc/keystone/keystone-paste.ini 文件，移除admin_token_auth从
    [pipeline:public_api], [pipeline:admin_api], 和[pipeline:api_v3]部分.


    下面是分别从pipline中移出admin_token_auth:
    [pipeline:public_api]
    # The last item in this pipeline must be public_service or an equivalent
    # application. It cannot be a filter.
    #pipeline = sizelimit url_normalize request_id build_auth_context token_auth admin_token_auth json_body
    pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body

    [pipeline:admin_api]
    # The last item in this pipeline must be admin_service or an equivalent
    # application. It cannot be a filter.
    #pipeline = sizelimit url_normalize request_id build_auth_context token_auth admin_token_auth json_body
    pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body

    [pipeline:api_v3]
    # The last item in this pipeline must be service_v3 or an equivalent
    # application. It cannot be a filter.
    #pipeline = sizelimit url_normalize request_id build_auth_context token_auth admin_token_auth json_body
    pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body


###二、去掉环境变量OS_TOKEN、OS_URL

    # unset OS_TOKEN OS_URL


###三、作为管理员，请求身份验证令牌API版本2

    root@king:~# openstack --os-auth-url http://localhost:35357 \
    > --os-project-name admin --os-username admin --os-auth-type password \
    > token issue
    Password:
    +------------+----------------------------------+
    | Field      | Value                            |
    +------------+----------------------------------+
    | expires    | 2015-11-16T16:26:20Z             |
    | id         | a0d7182b5bf24549886a9f45b903c905 |
    | project_id | aa3622f4a6ee4c379f4294b4492562b0 |
    | user_id    | 22488b4d6ec3414aac16de978337909d |
    +------------+----------------------------------+
    root@king:~#


###四、Identity 版本 3 API添加支持域

    root@king:~# openstack --os-auth-url http://localhost:35357 \
    --os-project-domain-id default --os-user-domain-id default \
    >  --os-project-name admin --os-username admin --os-auth-type password \
    >  token issue
    Password:
    +------------+----------------------------------+
    | Field      | Value                            |
    +------------+----------------------------------+
    | expires    | 2015-11-16T16:33:53.150489Z      |
    | id         | 5956011a5bb24bf38910b31e1f2d89f1 |
    | project_id | aa3622f4a6ee4c379f4294b4492562b0 |
    | user_id    | 22488b4d6ec3414aac16de978337909d |
    +------------+----------------------------------+
    root@king:~#
    注意密码：是admin用户的密码


###五、作为admin用户，列出用户作为admin核实admin可以执行admin-only CLI命令


    root@king:~# openstack --os-auth-url http://localhost:35357 \
    > --os-project-name admin --os-username admin --os-auth-type password \
    > > project list
    Password:
    +----------------------------------+---------+
    | ID                               | Name    |
    +----------------------------------+---------+
    | 255de1bf31434000a4824ef4aec84f36 | service |
    | aa3622f4a6ee4c379f4294b4492562b0 | admin   |
    | b8f1c26e2b3c447e8ba664fc999c6e70 | demo    |
    +----------------------------------+---------+
    root@king:~#


###六、作为admin用户，列出用户核实认证服务

    root@king:~#
    root@king:~# openstack --os-auth-url http://localhost:35357 \
    > > --os-project-name admin --os-username admin --os-auth-type password \
    > > user list
    Password:
    +----------------------------------+-------+
    | ID                               | Name  |
    +----------------------------------+-------+
    | 22488b4d6ec3414aac16de978337909d | admin |
    | de83ed698d0744e29c75fed800498670 | demo  |
    +----------------------------------+-------+
    root@king:~#


###七、作为 admin 用户, 列出角色验证keystone服务

    root@king:~# openstack --os-auth-url http://localhost:35357 \
    > --os-project-name admin --os-username admin --os-auth-type password \
    > role list
    Password:
    +----------------------------------+-------+
    | ID                               | Name  |
    +----------------------------------+-------+
    | 8b355da0962d4da692d05a60cec737a1 | user  |
    | d574ffdf0ef748138fb3ec2460d47d9e | admin |
    +----------------------------------+-------+
    root@king:~#


###八、作为demo用户，请求token 认证从3版本的API

    root@king:~# openstack --os-auth-url http://localhost:5000 \
    --os-project-domain-id default --os-user-domain-id default \
    > --os-project-name demo --os-username demo --os-auth-type password \
    > token issue
    Password:
    +------------+----------------------------------+
    | Field      | Value                            |
    +------------+----------------------------------+
    | expires    | 2015-11-16T16:42:11.621579Z      |
    | id         | e424226f71b14ef1895207769a9eed31 |
    | project_id | b8f1c26e2b3c447e8ba664fc999c6e70 |
    | user_id    | de83ed698d0744e29c75fed800498670 |
    +------------+----------------------------------+
    root@king:~#
    注释：这里输入demo密码


###九、作为 demo 用户, 尝试列出用户不能执行 admin-only CLI 命令

    root@king:~# openstack --os-auth-url http://localhost:5000 \
    > --os-project-domain-id default --os-user-domain-id default \
    > --os-project-name demo --os-username demo --os-auth-type password \
    > user list
    Password:
    ERROR: openstack You are not authorized to perform the requested action: admin_required (HTTP 403) (Request-ID: req-f7d2441e-d029-4ea7-88e9-ae7797cca764)
    root@king:~#
