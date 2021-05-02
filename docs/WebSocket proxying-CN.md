WebSocket proxying 中文翻译
===

翻译自 [WebSocket proxying](https://nginx.org/en/docs/http/websocket.html) 。

---

要将客户端和服务器之间的连接从 HTTP/1.1 变成 WebSocket，就需要使用 HTTP/1.1 中可用的[协议切换](https://tools.ietf.org/html/rfc2616#section-14.42)机制。

但有一个微妙的地方：由于 `Upgrade` 是一个 [hop-by-hop](https://tools.ietf.org/html/rfc2616#section-13.5.1) 的头，所以它不是从客户端传递到代理服务器。对于正向代理，客户端可以使用 *CONNECT* 方法来规避这个问题。但这在反向代理中是行不通的，因为客户端不知道任何代理服务器，需要在代理服务器上进行特殊处理。

从 1.3.13 版本开始，nginx 实现了一种特殊的操作模式，即如果代理服务器返回的响应代码为 101（交换协议），而客户端通过请求中的 `Upgrade` 头要求进行协议切换，则允许在客户端和代理服务器之间建立隧道。

如上所述，包括 `Upgrade` 和 `Connection` 在内的 hop-by-hop 头信息不会从客户端传递给代理服务器，因此为了让代理服务器知道客户端将协议切换到 WebSocket 的意图，必须显式传递这些头信息：

```nginx
location /chat/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
}
```

一个更复杂的例子，在向代理服务器发出的请求中，`Connection` 头字段的值取决于客户端请求头中是否存在 `Upgrade` 字段：

```nginx
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        ...

        location /chat/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
        }
    }
```

默认情况下，如果代理服务器在 60 秒内没有传输任何数据，连接将被关闭。可以使用  [proxy_read_timeout](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_read_timeout) 指令增加超时时间。另外，还可以配置代理服务器定期发送 WebSocket ping 帧，以重置超时时间并检查连接是否仍然有效。