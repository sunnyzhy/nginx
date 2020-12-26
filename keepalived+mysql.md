# keepalived + mysql 实现 mysql-HA
环境
|节点名称|IP 地址|
|--|--|
|mysql master|192.168.197.106|
|mysql slave|192.168.197.107|
|VIP|192.168.197.100|

## 1. 安装 mysql
### 1.1 安装 mysql
[安装 mysql](https://github.com/sunnyzhy/mysql/blob/master/%E5%AE%89%E8%A3%85mysql8.md 'mysql')

### 1.2 配置 mysql 主从同步
[配置 mysql 主从同步](https://github.com/sunnyzhy/mysql/blob/master/mysql8%E6%95%B0%E6%8D%AE%E5%BA%93%E4%B8%BB%E4%BB%8E%E5%90%8C%E6%AD%A5%E9%85%8D%E7%BD%AE.md 'mysql 主从同步')

### 1.3 向数据库添加数据
#### 1.3.1 向主库 192.168.197.106 添加数据
```bash
# mysql -uroot -ppassword -h192.168.197.106

mysql> insert t_user(`name`) value('aa');
Query OK, 1 row affected (0.00 sec)

mysql> select * from t_user;
+----+------+
| id | name |
+----+------+
|  1 | aa   |
+----+------+
1 row in set (0.00 sec)
```

#### 1.3.2 向从库 192.168.197.107 添加数据
```bash
# mysql -uroot -ppassword -h192.168.197.107

mysql> select * from t_user;
+----+------+
| id | name |
+----+------+
|  1 | aa   |
+----+------+
1 row in set (0.00 sec)

mysql> insert t_user(`name`) value('bb');
Query OK, 1 row affected (0.00 sec)

mysql> select * from t_user;
+----+------+
| id | name |
+----+------+
|  1 | aa   |
|  2 | bb   |
+----+------+
2 rows in set (0.00 sec)
```

#### 1.3.3 查询主库 192.168.197.106 的数据
```bash
mysql> select * from t_user;
+----+------+
| id | name |
+----+------+
|  1 | aa   |
+----+------+
1 row in set (0.00 sec)
```

因为配置的是主从单向同步，所以从库的数据不会同步到主库。

## 2. 安装 keepalived
[安装 keepalived](https://github.com/sunnyzhy/nginx/blob/master/keepalived.md 'keepalived')

## 3. keepalived + mysql 实现 mysql-HA
keepalived 只是提供一个 VIP，但高可用的是服务，因此 keepalived 一般会与 lvs、nginx、haproxy、mysql 等服务配合使用，以实现服务的高可用。但是如果与 keepalived 配合使用的服务出现异常时，keepalived 提供的 VIP 也就没有任何意义了，因此我们可以通过 shell 脚本来自动检测服务是否正常；如果服务不正常，VIP 就自动飘移至 backup 节点，这个功能就可以使用 **vrrp_script** 实现。

### 3.1 修改 192.168.197.106 节点的 keepalived 配置
```bash
# vim /etc/keepalived/keepalived.conf
global_defs {
   router_id node_106
}

vrrp_script check_mysql {                     # 定义监测脚本
    script "/usr/local/script/check_mysql.sh" # 监测进程的 shell 脚本
    interval 5                                # 脚本执行间隔，每 5s 检测一次
    weight -5                                 # 检测失败（脚本返回非0），则优先级变更为 -5
    fall 3                                    # 检测失败 3 次，才确定是真正失败。会减少 weight 优先级（1-255之间）
    rise 3                                    # 检测成功 3 次，才确定是真正成功。但不修改优先级
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.197.100
    }
    track_script {                            # 调用监测脚本
        check_mysql
    }
}

virtual_server 192.168.197.100 3306 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.197.106 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 3306
       }
    }
}
```

### 3.2 重启 192.168.197.106 节点的 keepalived 服务
```bash
# systemctl restart keepalived
```

### 3.3 修改 192.168.197.107 节点的 keepalived 配置
```bash
# vim /etc/keepalived/keepalived.conf
global_defs {
   router_id node_107
}

vrrp_script check_mysql {
    script "/usr/local/script/check_mysql.sh"
    interval 5
    weight -5
    fall 3
    rise 3
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.197.100
    }
    track_script {
        check_mysql
    }
}

virtual_server 192.168.197.100 3306 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.197.107 3306 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 3306
       }
    }
}
```

### 3.4 重启 192.168.197.107 节点的 keepalived 服务
```bash
# systemctl restart keepalived
```

### 3.5 mysql 监测脚本
#### 3.5.1 mysql 监测脚本
```bash
# vim /usr/local/script/check_mysql.sh
#!/bin/bash
mysql -uroot -pRoot@123 -e "select now();" > /dev/null 2>&1

if [ $? = 0 ] ;then
    exit 0
else
    exit 1
fi

# chmod 777 check_mysql.sh
```

#### 3.5.2 mysql 监测脚本授权
```bash
# chmod 777 check_mysql.sh
```

### 3.6 查看占用 VIP 的节点
#### 3.6.1 查看 192.168.197.106 节点
```bash
# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:7c:62:5c brd ff:ff:ff:ff:ff:ff
    inet 192.168.197.106/24 brd 192.168.197.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.197.100/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ae82:a00d:668a:4cf2/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

#### 3.6.2 查看 192.168.197.107 节点
```bash
# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a1:ab:0a brd ff:ff:ff:ff:ff:ff
    inet 192.168.197.107/24 brd 192.168.197.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ed8d:b0e:23a3:cee4/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

因为 192.168.197.106 节点的 keepalived 的优化级 100 大于 192.168.197.107 节点的 keepalived 的优化级 99，所以此时的 VIP 是被 192.168.197.106 的节点（mysql master）占用着的。

### 3.7 VIP 漂移
#### 3.7.1 停止 keepalived 服务
##### 3.7.1.1 通过 VIP 连接 mysql 服务
```bash
# mysql -uroot -ppassword -h 192.168.197.100

mysql> select * from t_user;
+----+------+
| id | name |
+----+------+
|  1 | aa   |
+----+------+
1 row in set (0.00 sec)
```

此时连接的是主库 192.168.197.106，所以查询出了一条数据。

##### 3.7.1.2 停止 192.168.197.106 节点的 keepalived 服务
```bash
# systemctl stop keepalived

# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:7c:62:5c brd ff:ff:ff:ff:ff:ff
    inet 192.168.197.106/24 brd 192.168.197.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ae82:a00d:668a:4cf2/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

##### 3.7.1.3 查看 192.168.197.107 节点
```bash
# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a1:ab:0a brd ff:ff:ff:ff:ff:ff
    inet 192.168.197.107/24 brd 192.168.197.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.197.100/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ed8d:b0e:23a3:cee4/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

此时 VIP 被 192.168.197.107 的节点（mysql slave）占用。

##### 3.7.1.4 通过 VIP 连接 mysql 服务
```bash
# mysql -uroot -ppassword -h 192.168.197.100

mysql> select * from t_user;
+----+------+
| id | name |
+----+------+
|  1 | aa   |
|  2 | bb   |
+----+------+
2 rows in set (0.01 sec)
```

此时连接的是从库 192.168.197.107，所以查询出了两条数据。

#### 3.7.2 停止 mysqld 服务
##### 3.7.2.1 查看 192.168.197.106 节点
```bash
# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:7c:62:5c brd ff:ff:ff:ff:ff:ff
    inet 192.168.197.106/24 brd 192.168.197.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.197.100/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ae82:a00d:668a:4cf2/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

##### 3.7.2.2 查看 192.168.197.107 节点
```bash
# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a1:ab:0a brd ff:ff:ff:ff:ff:ff
    inet 192.168.197.107/24 brd 192.168.197.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ed8d:b0e:23a3:cee4/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

VIP 被 192.168.197.106 的节点（mysql master）占用。

##### 3.7.2.3 停止 192.168.197.106 节点的 mysqld 服务
```bash
# systemctl stop mysqld
```

##### 3.7.2.4 查看 192.168.197.106 节点
```bash
# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:7c:62:5c brd ff:ff:ff:ff:ff:ff
    inet 192.168.197.106/24 brd 192.168.197.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ae82:a00d:668a:4cf2/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

##### 3.7.2.5 查看 192.168.197.107 节点
```bash
# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a1:ab:0a brd ff:ff:ff:ff:ff:ff
    inet 192.168.197.107/24 brd 192.168.197.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet 192.168.197.100/32 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ed8d:b0e:23a3:cee4/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

192.168.197.106 节点的 mysql 服务停止之后， VIP 被 192.168.197.107 的节点（mysql slave）占用。

