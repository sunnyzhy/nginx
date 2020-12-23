# keepalived + mysql 实现 MySQL-HA
环境
|节点名称|IP 地址|
|--|--|
|MySQL Master|192.168.197.106|
|MySQL Slave|192.168.197.107|
|VIP|192.168.197.100|

## 1. 安装 mysql 并实现主从同步
[安装 mysql 并实现主从同步](https://github.com/sunnyzhy/mysql/blob/master/mysql8%E6%95%B0%E6%8D%AE%E5%BA%93%E4%B8%BB%E4%BB%8E%E5%90%8C%E6%AD%A5%E9%85%8D%E7%BD%AE.md 'mysql')

## 2. 安装 keepalived
[安装 keepalived](https://github.com/sunnyzhy/nginx/blob/master/keepalived.md 'keepalived')

## 3. keepalived + mysql 实现 MySQL-HA
### 3.1 修改 MySQL Master 的 keepalived 配置
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

### 3.2 重启 MySQL Master 的 keepalived 服务
```bash
# systemctl restart keepalived
```

### 3.3 修改 MySQL Slave 的 keepalived 配置
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

### 3.4 重启 MySQL Slave 的 keepalived 服务
```bash
# systemctl restart keepalived
```

### 3.5 查看占用 VIP 的节点
#### 3.5.1 查看 MySQL Master 节点
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

#### 3.5.2 查看 MySQL Slave 节点
```bash
# ip a
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:a1:ab:0a brd ff:ff:ff:ff:ff:ff
    inet 192.168.197.107/24 brd 192.168.197.255 scope global noprefixroute ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::ed8d:b0e:23a3:cee4/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

因为 MySQL Master 节点的 keepalived 的优化级 100 大于 MySQL Slave 节点的 keepalived 的优化级 99，所以此时的 VIP 是被 192.168.197.106 的节点（MySQL Master）占用着的。

### 3.6 VIP 漂移


