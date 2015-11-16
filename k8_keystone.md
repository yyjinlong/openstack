root@king:~# openstack --os-auth-url http://localhost:35357   --os-project-name admin --os-username admin --os-auth-type password   token issue
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



root@king:~# openstack --os-auth-url http://localhost:35357   --os-project-name admin --os-username admin --os-auth-type password   token issue
Password: 
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-11-16T16:29:49Z             |
| id         | 717662f4bbbf4094bc69f556c0213939 |
| project_id | aa3622f4a6ee4c379f4294b4492562b0 |
| user_id    | 22488b4d6ec3414aac16de978337909d |
+------------+----------------------------------+
root@king:~# 


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



root@king:~# openstack --os-auth-url http://localhost:35357 \
> --os-project-name admin --os-username admin --os-auth-type password \
> > project list
> Password: 
> +----------------------------------+---------+
> | ID                               | Name    |
> +----------------------------------+---------+
> | 255de1bf31434000a4824ef4aec84f36 | service |
> | aa3622f4a6ee4c379f4294b4492562b0 | admin   |
> | b8f1c26e2b3c447e8ba664fc999c6e70 | demo    |
> +----------------------------------+---------+
> root@king:~# 
> root@king:~# 
> root@king:~# openstack --os-auth-url http://localhost:35357 \
> > --os-project-name admin --os-username admin --os-auth-type password \
> > user list
> Password: 
> +----------------------------------+-------+
> | ID                               | Name  |
> +----------------------------------+-------+
> | 22488b4d6ec3414aac16de978337909d | admin |
> | de83ed698d0744e29c75fed800498670 | demo  |
> +----------------------------------+-------+
> root@king:~# 
> root@king:~# 
> root@king:~# 
> root@king:~# opens
> openssl    openstack/ 
> root@king:~# opens
> openssl    openstack/ 
> root@king:~# openstack --os-auth-url http://localhost:35357 \
> > --os-project-name admin --os-username admin --os-auth-type password \
> > role list
> Password: 
> +----------------------------------+-------+
> | ID                               | Name  |
> +----------------------------------+-------+
> | 8b355da0962d4da692d05a60cec737a1 | user  |
> | d574ffdf0ef748138fb3ec2460d47d9e | admin |
> +----------------------------------+-------+
> root@king:~# 
>


root@king:~# openstack --os-auth-url http://localhost:5000 \
> --os-project-domain-id default --os-user-domain-id default \
> > --os-project-name demo --os-username demo --os-auth-type password \
> > token issue
> Password: 
> +------------+----------------------------------+
> | Field      | Value                            |
> +------------+----------------------------------+
> | expires    | 2015-11-16T16:42:11.621579Z      |
> | id         | e424226f71b14ef1895207769a9eed31 |
> | project_id | b8f1c26e2b3c447e8ba664fc999c6e70 |
> | user_id    | de83ed698d0744e29c75fed800498670 |
> +------------+----------------------------------+
> root@king:~# 
>




















