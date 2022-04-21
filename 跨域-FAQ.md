# 跨域 - FAQ

## Access to XMLHttpRequest at 'http://192.168.0.100/jwt/token' from origin 'http://localhost' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.

***原因***

- 根据 ```No 'Access-Control-Allow-Origin' header is present on the requested resource.``` 可以看出，因为被请求的资源没有设置 ```Access-Control-Allow-Origin```，所以从 ```http://localhost``` 发起的请求不被允许

***解决办法***

- 服务端设置 ```Access-Control-Allow-Origin``` 的头信息
   - ```add_header Access-Control-Allow-Origin '*' always;```: 允许所有的请求源
   - ```add_header Access-Control-Allow-Origin $http_origin always;```: 动态指定请求源，**推荐**

***以下 FAQ 里的 ```Access-Control-Allow-Origin = *``` 仅为示例，主要是为了说明 origin 的来源、withCredentials 的用法等。在实际配置中，还是用推荐的 ```add_header Access-Control-Allow-Origin $http_origin always;``` 配置。***

## Access to XMLHttpRequest at 'http://192.168.0.100/jwt/token' from origin 'http://localhost' has been blocked by CORS policy: The 'Access-Control-Allow-Origin' header contains multiple values 'http://localhost, \*', but only one is allowed.

***原因***

- 根据 ```The 'Access-Control-Allow-Origin' header contains multiple values 'http://localhost, *', but only one is allowed.``` 可以看出 ```Access-Control-Allow-Origin``` 头检测到了多个值，也就是重复配置跨域了，比如在该场景中，既在 nginx 里配置了跨域，也在网关里配置了跨域。

***解决办法***

- 删除 nginx 里的跨域配置
- 删除网关里的跨域配置，**推荐**

## Access to XMLHttpRequest at 'http://192.168.0.100/jwt/token' from origin 'http://192.168.0.100:9090' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: The value of the 'Access-Control-Allow-Origin' header in the response must not be the wildcard '\*' when the request's credentials mode is 'include'. The credentials mode of requests initiated by the XMLHttpRequest is controlled by the withCredentials attribute.

***原因***

- 如果服务端开启了 ```Access-Control-Allow-Credentials = true```, 并且设置了 ```Access-Control-Allow-Origin = *```，那意味着将 cookie 开放给了所有的网站。假设当前是 A 网站,并且在 cookie 里写入了身份凭证，用户同时打开了 B 网站, 那么 B 网站给服务器发的所有请求都是以 A 用户的身份进行的, 这将导致跨域问题。

**注: 此处的服务端包含 nginx**

***解决办法***

**如果服务端设置了 ```"Access-Control-Allow-Origin": "*"```，则客户端无需再设置 ```withCredentials: true```**

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
