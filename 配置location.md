# nginx指定文件路径有两种方式 root 和 alias
在 nginx.conf 中，添加 location 配置
```
 server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
```

# 以root方式设置资源路径
```
location /img/ {
    root       /var/www/image;
    autoindex  on;
}
```
当输入 localhost:80/img/ 时，会访问本机的 /var/www/image/img/ 目录

# 以alias 方式设置资源路径
```
location /img/ {
    alias      /var/www/image/;
    autoindex  on;
}
```
当输入 localhost:80/img/ 时，会访问本机的 /var/www/image/ 目录

# 注意
```
1.使用 alias 时，目录名后面一定要加 "/"

2.使用 alias 标签的目录块中不能使用 rewrite 的 break

3.alias 在使用正则匹配时，必须捕捉要匹配的内容并在指定的内容处使用

4.alias 只能位于 location 块中
```
