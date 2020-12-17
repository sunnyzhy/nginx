# keepalived
## 安装 keepalived
[keepalived 官网](https://www.keepalived.org/ "keepalived")

### 1. 安装依赖
```bash
# yum -y install gcc

# yum -y install openssl openssl-devel

# yum -y install libnl libnl-devel
```

### 2. 安装 keepalived
```bash
# cd /usr/local

# tar -zxvf keepalived-2.0.20.tar.gz

# mv keepalived-2.0.20 keepalived-2.0.20-install

# cd keepalived-2.0.20-install

# ./configure --prefix=/usr/local/keepalived-2.0.20

# make && make install
```

### 3. 配置 keepalived 启动项
```bash
# mkdir /etc/keepalived/

# cp /usr/local/keepalived-2.0.20/etc/keepalived/keepalived.conf /etc/keepalived/

# cp /usr/local/keepalived-2.0.20/etc/sysconfig/keepalived /etc/sysconfig/

# cp /usr/local/keepalived-2.0.20-install/keepalived/etc/init.d/keepalived /etc/init.d/
```

### 4. 编辑 keepalived.conf
```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp4s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 74:27:ea:65:2c:28 brd ff:ff:ff:ff:ff:ff
    inet 20.0.0.3/8 brd 20.255.255.255 scope global noprefixroute enp4s0
       valid_lft forever preferred_lft forever
    inet6 fe80::ccba:3c36:f0e6:7319/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

# vim /etc/keepalived/keepalived.conf
global_defs {
   # 路由id，主备节点不能相同
   router_id node_03
}

vrrp_instance VI_1 {
    # 指定 keepalived 的角色，MASTER 为主，BACKUP 为备
    state MASTER
    # 指定监测的网卡，可以使用 ifconfig 或 ip a 进行查看
    interface enp4s0
    # 虚拟路由的 id，主备节点需要设置为相同
    virtual_router_id 51
    # 优先级，主节点的优先级需要设置比备份节点高
    priority 100
    # 设置主备之间的心跳检测时间，单位为秒
    advert_int 1
    # 定义验证类型和密码
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 虚拟IP地址(VIP)，可以设置多个
    virtual_ipaddress {
        20.0.0.116
    }
}
```

### 5. 启动 keepalived
```bash
# systemctl start keepalived

# systemctl status keepalived
?.keepalived.service - LVS and VRRP High Availability Monitor
   Loaded: loaded (/usr/lib/systemd/system/keepalived.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-12-16 12:01:54 CST; 7min ago
  Process: 22453 ExecStart=/usr/local/keepalived-2.0.20/sbin/keepalived $KEEPALIVED_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 22454 (keepalived)
    Tasks: 3
   CGroup: /system.slice/keepalived.service
           ?..22454 /usr/local/keepalived-2.0.20/sbin/keepalived -D
           ?..22455 /usr/local/keepalived-2.0.20/sbin/keepalived -D
           ?..22456 /usr/local/keepalived-2.0.20/sbin/keepalived -D

Dec 16 12:02:03 node1 Keepalived_vrrp[22456]: (VI_1) Sending/queueing gratuitous ARPs o...16
Dec 16 12:02:03 node1 Keepalived_vrrp[22456]: Sending gratuitous ARP on enp4s0 for 20.0...16
Dec 16 12:02:03 node1 Keepalived_vrrp[22456]: Sending gratuitous ARP on enp4s0 for 20.0...16
Dec 16 12:02:03 node1 Keepalived_vrrp[22456]: Sending gratuitous ARP on enp4s0 for 20.0...16
Dec 16 12:02:03 node1 Keepalived_vrrp[22456]: Sending gratuitous ARP on enp4s0 for 20.0...16
Dec 16 12:02:30 node1 Keepalived_healthcheckers[22455]: Timeout reading data to remote S....
Dec 16 12:02:31 node1 Keepalived_healthcheckers[22455]: Timeout reading data to remote S....
Dec 16 12:02:31 node1 Keepalived_healthcheckers[22455]: Timeout reading data to remote S....
Dec 16 12:02:32 node1 Keepalived_healthcheckers[22455]: Timeout reading data to remote S....
Dec 16 12:02:33 node1 Keepalived_healthcheckers[22455]: Timeout reading data to remote S....
Hint: Some lines were ellipsized, use -l to show in full.
```

**active (running) 说明 keepalived 正在运行。**

### 6. 验证 keepalived
```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp4s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 74:27:ea:65:2c:28 brd ff:ff:ff:ff:ff:ff
    inet 20.0.0.3/8 brd 20.255.255.255 scope global noprefixroute enp4s0
       valid_lft forever preferred_lft forever
    inet 20.0.0.116/32 scope global enp4s0
       valid_lft forever preferred_lft forever
    inet6 fe80::ccba:3c36:f0e6:7319/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

在网卡 enp4s0 下已经添加了 20.0.0.116 的 VIP。
