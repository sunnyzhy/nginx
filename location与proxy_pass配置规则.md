# location 与 proxy_pass 配置规则

后台 Controller:

```java
@RestController
public class NginxController {
    @GetMapping(value = "/test")
    public String test1() {
        return "/test";
    }

    @GetMapping(value = "/test/1")
    public String test2() {
        return "/test/1";
    }

    @GetMapping(value = "/test/1/2")
    public String test3() {
        return "/test/1/2";
    }

    @GetMapping(value = "/test1")
    public String test4() {
        return "/test1";
    }

    @GetMapping(value = "/test1/2")
    public String test5() {
        return "/test1/2";
    }

    @GetMapping(value = "/")
    public String test6() {
        return "/";
    }

    @GetMapping(value = "/1")
    public String test7() {
        return "/1";
    }

    @GetMapping(value = "/1/2")
    public String test8() {
        return "/1/2";
    }
}
```

## proxy_pass 的末尾不加 /

nginx 转发请求的伪代码:

```java
     if (url.indexOf(location) > 0) {
         String[] arr = url.split(location);
         return proxy_pass + location + (arr.length == 1 ? "" : url.split(location)[1]);
     } else {
         if ((url + "/").indexOf(location) > 0) {
             return "301";
         }
         return "404";
     }
```

### location 的末尾不加 /

类似于模糊匹配，只要请求的路由前缀匹配上了 location ，nginx 就会转发请求。

示例:

```conf
   location /test {
         proxy_pass http://localhost:8080;
     }
```

|请求|nginx 转发的请求|响应|
|---|---|---|
|```http://localhost/test```|```http://localhost:8080/test```|/test|
|```http://localhost/test/1```|```http://localhost:8080/test/1```|/test/1|
|```http://localhost/test/1/2```|```http://localhost:8080/test/1/2```|/test/1/2|
|```http://localhost/test1```|```http://localhost:8080/test1```|/test1|
|```http://localhost/test1/2```|```http://localhost:8080/test1/2```|/test1/2|

### location 的末尾加了 /

类似于精确匹配，只有请求的路由完全匹配上了 location 首尾 / 之间的路由 ，nginx 才会转发请求; 否则，返回 404 。

示例:

```conf
   location /test/ {
         proxy_pass http://localhost:8080;
     }
```

|请求|nginx 转发的请求|响应|
|---|---|---|
|```http://localhost/test```|先经过 301 重定向到 ```http://localhost/test/``` , 再转发到 ```http://localhost:8080/test```|/test|
|```http://localhost/test/```|```http://localhost:8080/test```|/test|
|```http://localhost/test/1```|```http://localhost:8080/test/1```|/test/1|
|```http://localhost/test/1/2```|```http://localhost:8080/test/1/2```|/test/1/2|
|```http://localhost/test1```||404|
|```http://localhost/test1/2```||404|

## proxy_pass 的末尾加了 /

nginx 转发请求的伪代码:

```java
     if (url.indexOf(location) > 0) {
         String[] arr = url.split(location);
         return proxy_pass + (arr.length == 1 ? "" : url.split(location)[1]);
     } else {
         if ((url + "/").indexOf(location) > 0) {
             return "301";
         }
         return "404";
     }
```

### location 的末尾不加 /

类似于模糊匹配，nginx 在转发请求的时候，会舍弃匹配到的 location 。

示例:

```conf
   location /test {
         proxy_pass http://localhost:8080/;
     }
```

|请求|nginx 转发的请求|响应|
|---|---|---|
|```http://localhost/test```|```http://localhost:8080/```|/|
|```http://localhost/test/1```|```http://localhost:8080//1```|404|
|```http://localhost/test/1/2```|```http://localhost:8080//1/2```|404|
|```http://localhost/test1```|```http://localhost:8080/1```|/1|
|```http://localhost/test1/2```|```http://localhost:8080/1/2```|/1/2|

### location 的末尾加了 /

类似于精确匹配，nginx 在转发请求的时候，会舍弃匹配到的 location 。

示例:

```conf
   location /test/ {
         proxy_pass http://localhost:8080/;
     }
```

|请求|nginx 转发的请求|响应|
|---|---|---|
|```http://localhost/test```|先经过 301 重定向到 ```http://localhost/test/``` , 再转发到 ```http://localhost:8080/```|/|
|```http://localhost/test/```|```http://localhost:8080/```|/|
|```http://localhost/test/1```|```http://localhost:8080/1```|/1|
|```http://localhost/test/1/2```|```http://localhost:8080/1/2```|/1/2|
|```http://localhost/test1```||404|
|```http://localhost/test1/2```||404|

区别:

- location 的末尾不加 ```/``` , 类似于模糊匹配
- location 的末尾加了 ```/``` , 类似于精确匹配
- proxy_pass 的末尾不加 ```/``` , ```proxy_pass + location + (arr.length == 1 ? "" : url.split(location)[1])``` , 转发的请求不舍弃 location
- proxy_pass 的末尾加了 ```/``` , ```proxy_pass + (arr.length == 1 ? "" : url.split(location)[1])``` , 转发的请求会舍弃 location
