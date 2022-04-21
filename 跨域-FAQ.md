# 跨域 - FAQ

**注: 服务端包含了 nginx**

## Q1. Access to XMLHttpRequest at 'http://192.168.0.101/jwt/token' from origin 'http://192.168.0.100' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.

***原因***

- 根据 ```No 'Access-Control-Allow-Origin' header is present on the requested resource.``` 可以看出，因为被请求的资源没有设置 ```Access-Control-Allow-Origin```，所以从 ```http://192.168.0.100``` 发起的请求不被允许

***解决办法***

- 在服务端设置 ```Access-Control-Allow-Origin``` 的头信息
   - ```add_header Access-Control-Allow-Origin '*' always;```: 允许所有的请求源
   - ```add_header Access-Control-Allow-Origin $http_origin always;```: 动态指定请求源，**推荐**

***以下 FAQ 里的 ```add_header Access-Control-Allow-Origin '*' always;``` 仅为示例，主要是为了说明 origin 的来源、withCredentials 的用法等。在实际配置中，还是用推荐的 ```add_header Access-Control-Allow-Origin $http_origin always;``` 配置。***

## Q2. Access to XMLHttpRequest at 'http://192.168.0.101/jwt/token' from origin 'http://192.168.0.100' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header contains multiple values 'http://192.168.0.100, \*', but only one is allowed.

***原因***

- 根据 ```The 'Access-Control-Allow-Origin' header contains multiple values 'http://192.168.0.100, *', but only one is allowed.``` 可以看出 ```Access-Control-Allow-Origin``` 头检测到了多个值，也就是重复配置跨域了，比如在该场景中，既在 nginx 里配置了跨域，也在网关里配置了跨域。

***解决办法***

- 删除 nginx 里的跨域配置
- 删除网关里的跨域配置，**推荐**

## Q3. Access to XMLHttpRequest at 'http://192.168.0.101/jwt/token' from origin 'http://192.168.0.100' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: The value of the 'Access-Control-Allow-Origin' header in the response must not be the wildcard '\*' when the request's credentials mode is 'include'. The credentials mode of requests initiated by the XMLHttpRequest is controlled by the withCredentials attribute.

***原因***

- 服务端设置了 ```Access-Control-Allow-Origin: *```, 并且开启了 ```Access-Control-Allow-Credentials: true```，那意味着将 cookie 开放给了所有的请求源。
   ```conf
   add_header Access-Control-Allow-Origin '*' always;
   add_header Access-Control-Allow-Credentials 'true';
   ```

***解决办法***

**如果服务端同时设置了 ```Access-Control-Allow-Origin: *``` 和 ````Access-Control-Allow-Credentials: true```，那么客户端无需再设置 ```withCredentials: true```**

删除客户端请求时设置的 ```withCredentials: true```:

```ts
const service = axios.create({
  baseURL: import.meta.env.VITE_APP_BASE_API,
  timeout: 10000 //,
  // withCredentials: true
})
```

或

```ts
// axios.defaults.withCredentials = true
```

***注: 该解决方法同样适用于服务端的以下配置:***

```conf
add_header Access-Control-Allow-Origin $http_origin always;
add_header Access-Control-Allow-Credentials 'true';
```

***也就是说，不管服务端的配置是 ```add_header Access-Control-Allow-Origin '*' always;add_header Access-Control-Allow-Credentials 'true';``` ，还是 ```add_header Access-Control-Allow-Origin $http_origin always;add_header Access-Control-Allow-Credentials 'true';``` ，只要删除客户端的配置 ```withCredentials: true```，就能解决跨域问题。***

## Q4. Access to XMLHttpRequest at 'http://192.168.0.101/jwt/token' from origin 'http://192.168.0.100'' has been blocked by CORS policy: Request header field access-token is not allowed by Access-Control-Allow-Headers in preflight response.

***相当于把 Q3 问题里的 ```Access-Control-Allow-Origin '*'``` 里的通配符 ```*``` 改成了 ```$http_origin```（因为 Q3 的错误提示说明了 "不能使用通配符 *"）***

服务端:

```conf
add_header Access-Control-Allow-Origin $http_origin always;
add_header Access-Control-Allow-Credentials 'true';
add_header Access-Control-Allow-Headers '*';
add_header Access-Control-Allow-Methods '*';
```

客户端:

```ts
const service = axios.create({
  baseURL: import.meta.env.VITE_APP_BASE_API,
  timeout: 10000,
  withCredentials: true
})
```

***原因***

- 如果服务端同时设置了 ```Access-Control-Allow-Origin: $http_origin``` 和 ```Access-Control-Allow-Credentials 'true'```, 客户端也设置了 ```withCredentials: true```，那么服务端设置的 ```Access-Control-Allow-Headers: *``` 就会失效，此时 ```*``` 不再是通配符，而是普通的字符串。

***解决办法***

把 ```access-token``` 添加到配置服务端的 ```Access-Control-Allow-Headers``` 中，对于提示的其他不允许的字段，也都明确地添加进来:

```conf
add_header Access-Control-Allow-Origin $http_origin always;
add_header Access-Control-Allow-Headers 'content-type,access-token';
add_header Access-Control-Allow-Methods '*';
add_header Access-Control-Allow-Credentials 'true';
```
