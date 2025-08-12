# nginx
[nginx 官网](http://nginx.org/en/download.html "nginx")

## 1. 安装依赖库
``` bash
# yum -y install gcc-c++

# yum -y install pcre pcre-devel

# yum -y install zlib zlib-devel
```

## 2. 安装 nginx
``` bash
# cd /usr/local

# tar -zxvf nginx-1.18.0.tar.gz

# mv nginx-1.18.0 nginx-1.18.0-install

# cd nginx-1.18.0-install

# ./configure --prefix=/usr/local/nginx-1.18.0

# make && make install
```

## 3. 配置 nginx 开机启动
```bash
# vim /etc/systemd/system/nginx.service
[Unit]
Description=nginx 
After=network.target 
   
[Service] 
Type=forking 
ExecStart=/usr/local/nginx-1.18.0/sbin/nginx
ExecReload=/usr/local/nginx-1.18.0/sbin/nginx -s reload
ExecStop=/usr/local/nginx-1.18.0/sbin/nginx -s quit
PrivateTmp=true 
   
[Install] 
WantedBy=multi-user.target

# systemctl daemon-reload
```

## 4. 设置开机自启动

```bash
# systemctl enable nginx
```

## 5. 启动nginx服务
``` bash
# systemctl start nginx

# ps -ef | grep nginx
root       8701      1  0 14:47 ?        00:00:00 nginx: master process /usr/local/nginx-1.18.0/sbin/nginx
nobody     8702   8701  0 14:47 ?        00:00:00 nginx: worker process
root       8704   3321  0 14:48 pts/1    00:00:00 grep --color=auto nginx
```
在浏览器的地址栏中输入http://localhost/ ，就会有nginx的欢迎界面
Welcome to nginx!

## 6. 重启nginx服务
``` bash
# systemctl restart nginx
```

## 7. 停止nginx服务
``` bash
# systemctl stop nginx
```
