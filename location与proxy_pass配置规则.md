# location 与 proxy_pass 配置规则

## location 配置

### 1 基本介绍

location 配置用于匹配请求的 URL，即 ngnix 中的 $request_uri 变量，其配置格式如下：

```
location [ 空格 | = | ~ | ~* |^~|!~ | !~* ] /uri/
```

### 2 loacation 匹配顺序

#### 2.1 location 匹配格式规则前缀

- = 开头: 表示精确匹配
- ^~ 开头: 注意这不是一个正则表达式，它的目的是优于正则表达式的匹配；如果该 location 是最佳匹配，则不再进行正则表达式检测。
- ~ 开头: 表示区分大小写的正则匹配;
- ~* 开头: 表示不区分大小写的正则匹配
- !~ && !~\*: 表示区分大小写不匹配的正则和不区分大小写的不匹配的正则

#### 2.2 匹配的搜索顺序优先级

优先级从上到下依次递减:

1. 首先匹配 =
2. 其次匹配 ^~
3. 再其次按照配置文件的顺序进行正则匹配
4. 最后是交给 / 进行通用匹配

```
(location =) > (location 完整路径) > (location ^~ 路径) > (location ~,~* 正则顺序) > (location 部分起始路径) > (/)
```

### 3 匹配模式及顺序举例

|location匹配符|描述|
|--|--|
|location = /uri|= 开头表示精确匹配，只有完全匹配上才能生效。|
|location ^~ /uri|^~ 开头对 URL 路径进行前缀匹配，并且在正则之前。|
|location ~ pattern|~ 开头表示区分大小写的正则匹配。|
|location ~* pattern|~* 开头表示不区分大小写的正则匹配。|
|location /uri|不带任何修饰符，也表示前缀匹配，但是在正则匹配之后，如果没有正则命中，命中最长的规则。|
|location /|通用匹配，任何未匹配到其它 location 的请求都会匹配到，相当于 switch 中的 default。|

### 4 location 是否以"／"结尾

在 ngnix 中 location 进行的是模糊匹配:

- 不以"/"结尾时，```location /abc/def``` 可以匹配 /abc/defghi 请求，也可以匹配 /abc/def/ghi 等
- 以"/"结尾时，```location /abc/def/``` 不能匹配 /abc/defghi 请求，只能匹配 /abc/def/anything 等

## proxy_pass 配置

-  proxy_pass 配置的 uri 以 / 结尾时，相当于绝对路径， Nginx 不会把 location 中匹配的路径部分加入代理 uri
   ```conf
   location /proxy/ {
     proxy_pass http://127.0.0.1:8080/;
   }
   ```
   上述配置，当访问 http://127.0.0.1/proxy/test.html 的时候，代理 URL 是 http://127.0.0.1:8080/test.html


-  proxy_pass 配置的 uri 不是以 / 结尾时，Nginx 则会把匹配的路径部分加入代理 uri
   ```conf
   location /proxy/ {
     proxy_pass http://127.0.0.1:8080;
   }
   ```
   上述配置，当访问 http://127.0.0.1/proxy/test.html 的时候，代理 URL 是 http://127.0.0.1:8080/proxy/test.html

参考:

[https://www.hangge.com/blog/cache/detail_2979.html](https://www.hangge.com/blog/cache/detail_2979.html 'Nginx - 反向代理location与proxy_pass配置规则总结')
