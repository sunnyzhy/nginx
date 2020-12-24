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
### 3.1 修改 192.168.197.106 节点的 keepalived 配置
```bash
# vim /etc/keepalived/keepalived.conf
global_defs {
   router_id node_106
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

### 3.5 查看占用 VIP 的节点
#### 3.5.1 查看 192.168.197.106 节点
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

#### 3.5.2 查看 192.168.197.107 节点
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

### 3.6 VIP 漂移
#### 3.6.1 通过 VIP 连接 mysql 服务
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

#### 3.6.2 停止 192.168.197.106 节点的 keepalived 服务
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

#### 3.6.3 查看 192.168.197.107 节点
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

#### 3.6.4 通过 VIP 连接 mysql 服务
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
