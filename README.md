# nginx

## nginx  功能整理

```conf
server {
    listen       80;
    server_name  example.com;

    location / {
        root   /var/www/html;
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
        autoindex_format html;
    }
}
```

说明：

- ```autoindex on;```：开启目录浏览功能
- ```autoindex_exact_size off;```：设置为 off 时，以可读的方式显示文件大小，单位为 KB、MB 或者 GB；如果设置为 on，则显示文件的确切大小，单位是 bytes
- ```autoindex_localtime on;```：设置为 on 时，以服务器的文件时间作为显示的时间；默认为 off，显示的文件时间为 GMT 时间
- ```autoindex_format html;```：以网页的风格展示目录内容
