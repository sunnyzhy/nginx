# proxy_set_header

## 前言

- 客户端 ```20.0.3.107```
- nginx 服务端 ```20.0.3.101```
- nginx 服务端 ```20.0.3.102```
- nginx 服务端 ```20.0.3.103```

## 示例

### ```20.0.3.101 nginx.conf```

```conf
http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '"$http_host" "$host" "$http_x_real_ip" '
                      '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  logs/access.log  main;
    
    server {
        listen       80;
        server_name  localhost;
        
        location /test/ {
            proxy_set_header   X-Real-IP   $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://20.0.3.102;
        }
}
```

### ```20.0.3.102 nginx.conf```

```conf
http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '"$http_host" "$host" "$http_x_real_ip" '
                      '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  logs/access.log  main;
    
    server {
        listen       80;
        server_name  localhost;
        
        location /test/ {
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://20.0.3.103;
        }
}
```

### ```20.0.3.103 nginx.conf```

```conf
http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '"$http_host" "$host" "$http_x_real_ip" '
                      '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  logs/access.log  main;
    
    server {
        listen       80;
        server_name  localhost;
        
        location /test/ {
            proxy_pass http://www.baidu.com/;
        }
}
```

### 客户端 ```20.0.3.107``` 访问

```bash
curl http://20.0.3.101/test/
```

- ```20.0.3.101``` 日志:
   ```
   "20.0.3.101" "20.0.3.101" "-" 20.0.3.107 - - [07/Mar/2023:03:30:57 -0500] "GET /test/ HTTP/1.1" 302 154 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.0.0 Safari/537.36 Edg/110.0.1587.50" "-"
   ```
- ```20.0.3.102``` 日志:
   ```
   "20.0.3.102" "20.0.3.102" "20.0.3.107" 20.0.3.101 - - [07/Mar/2023:03:30:57 -0500] "GET /test/ HTTP/1.0" 302 154 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.0.0 Safari/537.36 Edg/110.0.1587.50" "20.0.3.107"
   ```
- ```20.0.3.103``` 日志:
   ```
   "20.0.3.103" "20.0.3.103" "20.0.3.107" 20.0.3.102 - - [07/Mar/2023:03:30:57 -0500] "GET /test/ HTTP/1.0" 302 154 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/110.0.0.0 Safari/537.36 Edg/110.0.1587.50" "20.0.3.107, 20.0.3.101"
   ```

分析:

1. 第一层反向代理通过 ```proxy_set_header X-Real-IP $remote_addr``` 把真实客户端 IP 写入到请求头 ```X-Real-IP```
2. ```$http_x_real_ip``` 获取真实的客户端 IP
3. ```$remote_addr``` 获取上一层请求的服务端 IP，可以是客户端 IP，也可以是反向代理服务端 IP:
   - 如果是客户端发送请求到 nginx 服务端，那么 nginx 服务端 ```$remote_addr``` 获取的就是客户端 IP
   - 如果是 nginx 服务端 A 转发请求到 nginx 服务端 B，那么 nginx 服务端 B ```$remote_addr``` 获取的就是服务端 A 的 IP
5. ```proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for``` 把请求头中的 ```X-Forwarded-For``` 与 ```$remote_addr``` 用逗号合并起来，如果请求头中没有 ```X-Forwarded-For```，则 ```$proxy_add_x_forwarded_for``` 为 ```$remote_addr```。 每经过一层反向代理，就在请求头 ```X-Forwarded-For``` 后追加上游服务器 IP。
   - 公式:
       - ```X-Forwarded-For``` = 上游服务器的 ```$http_x_forwarded_for``` + ```,``` + 上游服务器的 ```$remote_addr```
   - 格式:
       - ```real client ip, proxy ip 1, ... ,proxy ip N```
   - 示例:
       1. 在第一层反向代理服务器 ```20.0.3.101``` 中，```$remote_addr``` 是 ```20.0.3.107```，```$http_x_real_ip``` 是空，```$http_x_forwarded_for``` 是空（当前为第一层反向代理服务器，其上游服务器为空，所以 ```$http_x_forwarded_for``` 的值为空）
       2. 在第二层反向代理服务器 ```20.0.3.102``` 中，```$remote_addr``` 是 ```20.0.3.101```，```$http_x_real_ip``` 是 ```20.0.3.107``` ，```$http_x_forwarded_for``` 是 ```20.0.3.107```（即第一层反向代理服务器的 ```$http_x_forwarded_for``` + 第一层反向代理服务器的 ```$remote_addr```）
       3. 在第三层反向代理服务器 ```20.0.3.103``` 中，```$remote_addr``` 是 ```20.0.3.102```，```$http_x_real_ip``` 是 ```20.0.3.107``` ，```$http_x_forwarded_for``` 是 ```20.0.3.107, 20.0.3.101```（即第二层反向代理服务器的 ```$http_x_forwarded_for``` + 第二层反向代理服务器的 ```$remote_addr```）

## 总结

### remote_addr

表示发出请求的远程主机的 IP 地址，```remote_addr``` 代表客户端的 IP，但它的值不是由客户端提供的，而是服务端根据客户端的 IP 指定的。

### x_forwarded_for

简称 XFF 头，它代表客户端，也就是 HTTP 的请求端真实的 IP，只有在通过了 HTTP 代理或者负载均衡服务器时才会添加该项。

### X-Real-IP

当有多个代理时候，可以在第一个反向代理上配置 ```proxy_set_header X-Real-IP $remote_addr``` 获取真实客户端 IP

### 区别

- ```X-Forwarded-For``` 一般是每一个非透明代理转发请求时会将上游服务器的 IP 地址追加到 ```X-Forwarded-For``` 的后面，使用英文逗号分割
- ```X-Forwarded-For``` 是多个 IP 地址，而 ```X-Real-IP``` 是一个
