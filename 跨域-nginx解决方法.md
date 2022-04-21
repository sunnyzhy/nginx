# 跨域 - nginx 解决方法

## 场景

- 192.168.0.101: nginx 服务器的地址，网关的反向代理配置:
    ```conf
        server {
            listen       80;
            server_name  localhost;

            location /jwt {
              add_header Access-Control-Allow-Origin $http_origin always;
              add_header Access-Control-Allow-Headers '*';
              add_header Access-Control-Allow-Methods '*';
              add_header Access-Control-Allow-Credentials 'true';
              if ($request_method = 'OPTIONS') {
              	return 200;
              }

              proxy_pass http://127.0.0.1:8765;
              proxy_set_header host $host;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header referer "-";
              proxy_redirect default;
            }
        }
    ```
- 192.168.0.100: 本机地址，前端部署在本地，vue 配置: ```BASE_API: '"http://192.168.0.101"'```
- spring-cloud-starter-gateway 跨域配置:
    ```java
    @Configuration
    public class CorsConfig {
        @Bean
        public WebFilter corsFilter() {
            return (ServerWebExchange ctx, WebFilterChain chain) -> {
                ServerHttpRequest request = ctx.getRequest();
                if (CorsUtils.isCorsRequest(request)) {
                    HttpHeaders requestHeaders = request.getHeaders();
                    ServerHttpResponse response = ctx.getResponse();
                    HttpMethod requestMethod = requestHeaders.getAccessControlRequestMethod();
                    HttpHeaders headers = response.getHeaders();
                    headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_ORIGIN, requestHeaders.getOrigin());
                    headers.addAll(HttpHeaders.ACCESS_CONTROL_ALLOW_HEADERS, requestHeaders
                            .getAccessControlRequestHeaders());
                    if (requestMethod != null) {
                        headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_METHODS, requestMethod.name());
                    }
                    headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_CREDENTIALS, "true");
                    headers.add(HttpHeaders.ACCESS_CONTROL_EXPOSE_HEADERS, "*");
                    headers.add(HttpHeaders.ACCESS_CONTROL_MAX_AGE, "18000L");
                    if (request.getMethod() == HttpMethod.OPTIONS) {
                        response.setStatusCode(HttpStatus.OK);
                        return Mono.empty();
                    }
                }
                return chain.filter(ctx);
            };
        }
    }
    ```

## 步骤

1. 打开本地浏览器，输入 ```http://localhost```
2. 调用远程 nginx 服务器对外开放的接口 ```http://192.168.0.101/jwt/token``` 获取 token
    浏览器控制台返回错误信息:

    ```
    Access to XMLHttpRequest at 'http://192.168.0.101/jwt/token' from origin 'http://localhost' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header contains multiple values 'http://localhost, http://localhost', but only one is allowed.
    ```
3. 在本地浏览器中输入 ```http://192.168.0.100```
4. 再次调用远程 nginx 服务器对外开放的接口 ```http://192.168.0.101/jwt/token``` 获取 token
    浏览器控制台返回错误信息:

    ```
    Access to XMLHttpRequest at 'http://192.168.0.101/jwt/token' from origin '192.168.0.100' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header contains multiple values '192.168.0.100, 192.168.0.100', but only one is allowed.
    ```
5. 把 nginx 的跨域配置修改为 ```add_header Access-Control-Allow-Origin '*' always;```
6. 在本地浏览器中输入 ```http://localhost```
7. 再次调用远程 nginx 服务器对外开放的接口 ```http://192.168.0.101/jwt/token``` 获取 token
    浏览器控制台返回错误信息:

    ```
    Access to XMLHttpRequest at 'http://192.168.0.101/jwt/token' from origin 'http://localhost' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header contains multiple values 'http://localhost, *', but only one is allowed.
    ```
8. 在本地浏览器中输入 ```http://192.168.0.100```
9. 再次调用远程 nginx 服务器对外开放的接口 ```http://192.168.0.101/jwt/token``` 获取 token
    浏览器控制台返回错误信息:

    ```
    Access to XMLHttpRequest at 'http://192.168.0.101/jwt/token' from origin 'http://192.168.0.100' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header contains multiple values 'http://192.168.0.100, *', but only one is allowed.
    ```

注:

1. 从上述步骤可以看出， ```The 'Access-Control-Allow-Origin' header contains multiple values``` 所包含的值分别对应网关配置的 ```Access-Control-Allow-Origin``` 和 nginx 配置的  ```Access-Control-Allow-Origin```
2. 关于服务端设置 Access-Control-Allow-Origin 的头信息
   - ``add_header Access-Control-Allow-Origin '*' always;``: 允许所有的请求源
   - ``add_header Access-Control-Allow-Origin $http_origin always;``: 动态指定请求源，**推荐**

***原因***

- 根据 ```The 'Access-Control-Allow-Origin' header contains multiple values 'http://192.168.0.100, http://192.168.0.100', but only one is allowed.```，可以看出 ```Access-Control-Allow-Origin``` 头检测到了多个值，也就是重复配置跨域了，比如在该场景中，既在 nginx 里配置了跨域，也在网关里配置了跨域。

***解决办法***

- 删除 nginx 里的跨域配置
- 删除网关里的跨域配置，**推荐**

   在本地浏览器中输入 ```http://192.168.0.100```，可以看到 HTTP 的请求信息与响应信息如下:

   ```
   General
     Request URL: http://192.168.0.101/jwt/token
     Request Method: OPTIONS
     Status Code: 200 OK
     Remote Address: 192.168.0.101:80
     Referrer Policy: strict-origin-when-cross-origin

   Response Headers
     Access-Control-Allow-Credentials: true
     Access-Control-Allow-Headers: *
     Access-Control-Allow-Methods: *
     Access-Control-Allow-Origin: http://192.168.0.100
     Connection: keep-alive
     Content-Length: 0
     Content-Type: application/octet-stream
     Date: Thu, 21 Apr 2022 08:01:47 GMT
     Server: nginx/1.13.6

   Request Headers
     Accept: */*
     Accept-Encoding: gzip, deflate
     Accept-Language: zh-CN,zh;q=0.9
     Access-Control-Request-Headers: access-token,content-type
     Access-Control-Request-Method: POST
     Connection: keep-alive
     Host: 192.168.0.101
     Origin: http://192.168.0.100
     Referer: http://192.168.0.100
     Sec-Fetch-Mode: cors
     User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.45 Safari/537.36
   ```

***分析***

1. 从 ```General``` 里的 ```Status Code: 200 OK``` 可以看出跨域成功
2. 从 ```Request Headers``` 里的 ```Origin: http://localhost``` 可以看出，源为 ```http://192.168.0.100```
3. 从 ```General``` 里的 ```Request URL: http://192.168.0.101/jwt/token``` 可以看出，目标为 ```http://192.168.0.101```
4. 从 ```Response Headers``` 里的 ```Access-Control-Allow-Origin: http://192.168.0.100``` 可以看出，服务端允许源 ```http://192.168.0.100``` 跨域
