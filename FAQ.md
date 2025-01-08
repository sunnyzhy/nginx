## 1 The 'Access-Control-Allow-Origin' header contains multiple values

- 原因
  nignx 设置了跨域后，gateway 也设置了跨域，在服务上部署是在同一个域内，但是前端在本地调试，会和服务上的域重复

  出现这个问题是因为跨域选项设置多了，查看下服务端和 nginx 端是不是重复配置了 ```Access-Control-Allow-Origin:*```

- 解决
  将 nginx 的域注释掉即可，仅留 gateway 的跨域设置

## location 本地目录时访问报 403

1. 执行命令 ```ps aux | grep nginx```，如果 ```master process``` 和 ```worker process``` 的用户名不一样。
2. 在 nginx.conf 的第一行添加 ```user  root;```
3. 重启 nginx
