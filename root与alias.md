# root 与 alias

nginx 指定文件路径有两种方式 root 和 alias，指令的使用方法和作用域：

```
[root]
语法：root path
默认值：root html
配置段：http、server、location、if

[alias]
语法：alias path
配置段：location
```

root 与 alias 主要区别在于 nginx 如何解释 location 后面的 uri，这会使两者分别以不同的方式将请求映射到服务器文件上。

**root 的处理结果是：root 路径 ＋ location 路径**

**alias 的处理结果是：使用 alias 路径替换 location 路径**

alias 是一个目录别名的定义，root 则是最上层目录的定义。

## 以 root 方式设置资源路径

```conf
location /img {
    root       /var/www/image;
    autoindex  on;
}
```

当输入 ```localhost:80/img/``` 时，会访问本机的 ```/var/www/image/img/``` 目录

## 以 alias 方式设置资源路径

```conf
location /img {
    alias      /var/www/image/;
    autoindex  on;
}
```

当输入 ```localhost:80/img/``` 时，会访问本机的 ```/var/www/image/``` 目录

## 注意

1. 使用 alias 时，目录名后面一定要加 "/"
2. alias 在使用正则匹配时，必须捕捉要匹配的内容并在指定的内容处使用
3. alias 只能位于 location 块中。（root 可以不放在 location 中）
