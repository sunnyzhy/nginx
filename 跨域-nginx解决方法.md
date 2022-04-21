# 跨域 - nginx解决方法

## 场景

1. 本机 ip 地址为 192.168.0.100，网关的 ip 地址为 192.168.0.101
2. nginx 跨域配置:
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

              proxy_pass http://192.168.0.101:8765;
              proxy_set_header host $host;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header X-Real-IP $remote_addr;
              proxy_set_header referer "-";
              proxy_redirect default;
            }
        }
    ```
3. spring-cloud-starter-gateway 跨域配置:
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

## 出现的问题

浏览器控制台返回错误信息:

```
Access to XMLHttpRequest at 'http://192.168.0.100/jwt/token' from origin 'http://localhost' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header contains multiple values 'http://localhost, http://localhost', but only one is allowed.
```

## 第一种分析

通过 ```localhost``` 访问 nginx。

***原因***

- 根据 ```Access to XMLHttpRequest at 'http://192.168.0.100/jwt/token' from origin 'http://localhost' has been blocked by CORS policy```，可以看出源 ```http://localhost``` 访问目标 ```http://192.168.0.100```，不满足同源策略，因而造成跨域问题。

***解决办法***

- 通过 ```192.168.0.100``` 访问 nginx。

## 第二种分析（推荐）

spring-cloud-starter-gateway 跨域配置:

- 当 ```headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_ORIGIN, requestHeaders.getOrigin());``` 的时候，返回 ```The 'Access-Control-Allow-Origin' header contains multiple values 'http://localhost, http://localhost', but only one is allowed.```
- 当 ```headers.add(HttpHeaders.ACCESS_CONTROL_ALLOW_ORIGIN, "*");``` 的时候，返回 ```The 'Access-Control-Allow-Origin' header contains multiple values '*, http://localhost', but only one is allowed.```

可以看出， 不管是 ```'http://localhost, http://localhost'```，还是 ```'http://localhost, *'```，分别对应网关配置的 ```Access-Control-Allow-Origin``` 和 nginx 配置的  ```Access-Control-Allow-Origin```

***原因***

- 根据 ```The 'Access-Control-Allow-Origin' header contains multiple values 'http://localhost, http://localhost', but only one is allowed.```，可以看出 ```Access-Control-Allow-Origin``` 头检测到了多个值，也就是重复配置跨域了，比如在该场景中，既在 nginx 里配置了跨域，也在网关里配置了跨域。

***解决办法***

- 删除 nginx 里的跨域配置
- 删除网关里的跨域配置，**推荐**

***效果***

- 既能通过 ```localhost``` 访问 nginx
- 也能通过 ```192.168.0.100``` 访问 nginx
