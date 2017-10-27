# 下载
http://nginx.org/en/download.html
```
Nginx官网提供了三个类型的版本
Mainline version：Mainline 是 Nginx 目前主力在做的版本，可以说是开发版
Stable version：最新稳定版，生产环境上建议使用的版本
Legacy versions：遗留的老版本的稳定版
```

# 安装依赖库
``` javascript
# yum -y install gcc-c++

# yum -y install pcre*

# yum -y install zlib*
```

# 解压
``` javascript
# mkdir -p /usr/local/nginx/tar.gz

# cd /usr/local/nginx/tar.gz

# tar -zxvf nginx-1.12.2.tar.gz

# cd nginx-1.12.2
```

# 编译 & 安装
``` javascript
# ./configure --prefix=/usr/local/nginx

# make

# make install
```

# 启动nginx服务
``` javascript
# cd /usr/local/nginx/sbin

# ./nginx
```

# 查看nginx进程
``` javascript
# ps -ef | grep nginx
root     31586     1  0 10:26 ?        00:00:00 nginx: master process ./nginx
nobody   31587 31586  0 10:26 ?        00:00:00 nginx: worker process
root     31592 28966  0 10:27 pts/2    00:00:00 grep nginx
```
在浏览器的地址栏中输入http://localhost/ ，就会有nginx的欢迎界面
Welcome to nginx!

# 重启nginx服务
``` javascript
# ./nginx -s reload
```

# 停止nginx服务
``` javascript
# ./nginx -s stop
```