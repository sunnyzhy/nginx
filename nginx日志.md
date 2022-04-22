# nginx 日志

## access_log

访问日志主要记录客户端的请求。

### 语法

```conf
access_log path [format [buffer=size] [gzip[=level]] [flush=time] [if=condition]]; # 设置访问日志
access_log off; # 关闭访问日志
```

参数说明:

- path: 指定日志的存放位置。可以是相对路径，也可是绝对路径。
- format: 指定日志的格式。默认使用预定义的 combined。
- buffer: 用来指定日志写入时的缓存大小。默认是 64k。
- gzip: 日志写入前先进行压缩。压缩率可以指定，从 1 到 9 数值越大压缩比越高，同时压缩的速度也越慢。默认是 1。
- flush: 设置缓存的有效时间。如果超过 flush 指定的时间，缓存中的内容将被清空。
- if: 条件判断。如果指定的条件计算为 0 空字符串，那么该请求不会写入日志。
- off: 当前作用域下的所有的请求日志都被关闭。

### 作用域

access_log 的作用域:

- http
- server
- location
- if in location
- limit_except

### 基本用法

```conf
access_log logs/gateway.access.log;
```

把日志写入相对路径 ```logs/gateway.access.log```，即 nginx 当前目录下的 logs 文件夹。日志格式使用默认的 ```combined```。

```conf
access_log logs/gateway.access.log buffer=32k gzip flush=1m;
```

把日志写入相对路径 ```logs/gateway.access.log```，日志格式使用默认的 combined，指定日志的缓存大小为 32k，日志写入前启用 gzip 进行压缩，压缩比使用默认值 1，缓存数据有效时间为 1 分钟。

## log_format

自定义日志格式。

### 语法

```conf
log_format name [escape=default|json] string ...;
```

参数说明:

- name: 格式名称，在 access_log 中引用。默认的日志格式为 ```combined```。
- escape: 设置变量中的字符编码方式是 json 还是 default，默认是 default。
- string: 要定义的日志格式内容。该参数可以有多个。参数中可以使用 nginx 变量。

    log_format 中常用的变量：

    |变量|含义|
    |--|--|
    |$bytes_sent|发送给客户端的总字节数|
    |$body_bytes_sent|发送给客户端的字节数，不包括响应头的大小|
    |$connection|连接序列号|
    |$connection_requests|当前通过连接发出的请求数量|
    |$msec|日志写入时间，单位为秒，精度是毫秒|
    |$pipe|如果请求是通过 http 流水线发送，则其值为 ```p```，否则为 ```.```|
    |$request_length|请求长度（包括请求行，请求头和请求体）|
    |$request_time|请求处理时长，单位为秒，精度为毫秒，从读入客户端的第一个字节开始，直到把最后一个字符发送张客户端进行日志写入为止|
    |$status|响应状态码|
    |$time_iso8601|标准格式的本地时间，形如 ```2017-05-24T18:31:27+08:00```|
    |$time_local|通用日志格式下的本地时间，如 ```24/May/2017:18:31:27 +0800```|
    |$http_referer|请求的 referer 地址|
    |$http_user_agent|客户端浏览器信息|
    |$remote_addr|客户端 IP 地址|
    |$http_x_forwarded_for|当前端有代理服务器时，设置 web 节点记录客户端地址的配置，此参数生效的前提是代理服务器也要进行相关的 x_forwarded_for 设置|
    |$request|完整的原始请求行，如 ```GET / HTTP/1.1```|
    |$remote_user|客户端用户名称，针对启用了用户认证的请求|
    |$request_uri|完整的请求地址，如 "https://www.baidu.com/"|

默认的日志格式 combined:

```conf
log_format combined '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
```

### 作用域

log_format 的作用域:

- http

### 基本用法

```conf
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"'
                  '"$upstream_status" "$upstream_addr"';
access_log logs/gateway.access.log main;
```

定义了一个名称为 main 的日志格式，并在 access_log 指令中引用了它。

## error_log

错误日志主要记录服务器和请求处理过程中的错误信息。

### 语法

```conf
error_log file [level];
Default:    
error_log logs/error.log error;
```

参数说明:

- file: 指定日志的写入位置。
- level: 指定日志的级别。level 取值范围是按级别从低到高排列为:
   - debug
   - info
   - notice
   - warn
   - error
   - crit
   - alert
   - emerg
   
   只有日志的错误级别等于或高于 level 指定的值才会写入错误日志中。默认值是 ```error```。

### 作用域

error_log 的作用域:

- main
- http
- mail
- stream
- server
- location

### 基本用法

```conf
error_log logs/gateway.error.log;
```

把错误日志写入相对路径 ```logs/gateway.error.log```，日志级别使用默认的 ```error```。

## rewrite_log

### 语法

```conf
rewrite_log on | off;
Default:    
rewrite_log off;
```

### 作用域

rewrite_log 的作用域:

- http
- server
- location
- if

### 作用

由 ```ngx_http_rewrite_module``` 模块提供。用来记录重写日志，对于调试重写规则建议开启。启用时将在***错误日志***中记录 ```notice``` 级别的重写日志。

## open_log_file_cache

每一条日志记录的写入都是先打开文件再写入记录，然后关闭日志文件。如果在日志文件的路径中使用了变量，如 ```access_log logs/$host/gateway.access.log main;```，为提高性能，可以使用 ```open_log_file_cache``` 设置日志文件描述符的缓存。

### 语法

```conf
open_log_file_cache max=N [inactive=time] [min_uses=N] [valid=time];
```

参数说明:

- max: 设置缓存中最多容纳的文件描述符数量，如果被占满，采用 LRU 算法将描述符关闭。
- inactive: 设置缓存存活时间，默认是 10s。
- min_uses: 在 inactive 时间段内，日志文件最少使用几次，该日志文件描述符记入缓存，默认是 1 次。
- valid: 设置多久对日志文件名进行检查，看是否发生变化，默认是 60s。
- off: 不使用缓存。默认为 off。

### 作用域

open_log_file_cache 的作用域:

- http
- server
- location

### 基本用法

```conf
open_log_file_cache max=1000 inactive=20s valid=1m min_uses=2;
```

设置缓存最多缓存 1000 个日志文件描述符，20s 内如果缓存中的日志文件描述符至少被被访问 2 次，才不会被缓存关闭。每隔 1 分钟检查缓存中的文件描述符的文件名是否还存在。
