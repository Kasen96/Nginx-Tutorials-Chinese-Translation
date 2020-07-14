Using nginx as HTTP load balancer 中文翻译
===

* [介绍](#介绍)
* [负载均衡方法](#负载均衡方法)
* [默认负载均衡配置](#默认负载均衡配置)
* [连接最少的负载均衡](#连接最少的负载均衡)
* [会话持久性](#会话持久性)
* [加权负载平衡](#加权负载平衡)
* [健康检查](#健康检查)
* [扩展阅读](#扩展阅读)

翻译自 [Using nginx as HTTP load balancer](https://nginx.org/en/docs/http/load_balancing.html) 。

---

# 介绍

跨多个应用实例的负载均衡是一种常用于优化资源利用率，最大限度地提高吞吐量，减少延迟，并确保容错配置的技术。

这可以将 nginx 作为一个非常高效的 HTTP 负载均衡器，并将流量分配到多个应用服务器上，从而提高 Web 应用的性能、可扩展性和可靠性。

# 负载均衡方法

nginx 支持以下负载平衡机制（或方法）：

- **round-robin** —— 对应用服务器的请求是以轮询的方式分发的，
- **least-connected** —— 下一个请求被分配给活跃连接数最少的服务器，
- **ip-hash** —— 一个哈希函数用于决定下一个请求应该选择哪个服务器（基于客户端的 IP 地址）。

# 默认负载均衡配置

使用 nginx 进行负载均衡的最简单配置如下：

```nginx
http {
    upstream myapp1 {
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}
```

在上面的例子中，有 3 个相同的应用程序实例运行在 *srv1* - *srv3* 上。当没有特别配置负载均衡方式时，默认为轮询。所有的请求都被[代理](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)到服务器组 `myapp1`，nginx 使用 HTTP 负载均衡来分配请求。

nginx 中的反向代理实现包括 HTTP、HTTPS、FastCGI、uwsgi、SCGI、memcached 和 gRPC 的负载均衡。

要为 HTTPS 而不是 HTTP 配置负载均衡，只需使用 "*https*" 作为协议。

在为 FastCGI、uwsgi、SCGI、memcached 或 gRPC 设置负载均衡时，分别使用 [fastcgi_pass](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_pass)、[uwsgi_pass](https://nginx.org/en/docs/http/ngx_http_uwsgi_module.html#uwsgi_pass)、[scgi_pass](https://nginx.org/en/docs/http/ngx_http_scgi_module.html#scgi_pass)、[memcached_pass](https://nginx.org/en/docs/http/ngx_http_memcached_module.html#memcached_pass) 和 [grpc_pass](https://nginx.org/en/docs/http/ngx_http_grpc_module.html#grpc_pass) 指令。

# 连接最少的负载均衡

另一个负载均衡原则是最少连接。在一些请求需要较长的时间才能完成的情况下，最少连接可以更公平地控制应用实例的负载。

在使用最少连接的负载均衡时，nginx 会尽量避免让繁忙的应用服务器承受过多的请求，而是将新的请求分配给一个不太繁忙的服务器。

当在服务器组配置中使用 [least_conn](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#least_conn) 指令时，nginx 中的最少连接负载均衡被激活：

```nginx
    upstream myapp1 {
        least_conn;
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }
```

# 会话持久性

请注意，在轮询或最小连接负载均衡的情况下，每个后续客户端的请求都有可能被分配到不同的服务器上。不能保证同一个客户端总是被分配到同一个服务器上。

如果需要将客户端与特定的应用服务器绑定 —— 换句话说，使客户端的会话 "粘性 "或 "持久"，即总是试图选择特定的服务器 —— 可以使用 ip-hash 负载均衡机制。

使用 ip-hash，客户端的 IP 地址被用作散列密钥，以确定应该为客户端的请求选择服务器组中的哪个服务器。这种方法可以确保来自同一客户端的请求总是被导向同一个服务器，除非这个服务器不可用。

要配置 ip-hash 负载均衡，只需在服务器（上游）组配置中添加 [ip_hash](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#ip_hash) 指令即可：

```nginx
upstream myapp1 {
    ip_hash;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
```

# 加权负载平衡

还可以通过使用服务器权重来进一步影响 nginx 负载均衡算法。

在上面的例子中，没有配置服务器权重，这意味着所有指定的服务器对于特定的负载均衡方法来说都是相等的。

特别是轮询，这也意味着服务器之间的请求分配大致相同 —— 前提是有足够多的请求，并且以统一的方式处理请求并足够快地完成请求。

当为服务器指定[权重](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server)参数时，权重将作为负载均衡决策的一部分。

```nginx
    upstream myapp1 {
        server srv1.example.com weight=3;
        server srv2.example.com;
        server srv3.example.com;
    }
```

在这种配置下，每 5 个新的请求将被分配到不同的应用实例中，如下所示：3 个请求会被分配到 *srv1*，一个请求会被分配到 *srv2*，另一个会被分配到 *srv3*。

在最近版本的 nginx 中，同样可以将权重与最少连接和 ip-hash 负载均衡结合使用。

# 健康检查

nginx 中的反向代理实现包括带内（或被动）服务器健康检查。如果某台服务器响应出现错误，nginx 会将这台服务器标记为故障，并会暂时避免选择这台服务器进行后续的入站请求。

[max_fails](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 指令设置在 [fail_timeout](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 期间应与服务器进行的连续不成功尝试通信的次数。默认情况下，[max_fails](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 被设置为 1 。当它被设置为 0 时，该服务器的健康检查将被禁用。[fail_timeout](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 参数还定义了服务器被标记为失败的时间。在 [fail_timeout](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 间隔之后，nginx 将开始优雅地探测服务器的实时客户端请求。如果探测成功，服务器就会被标记为实时服务器。

# 扩展阅读

此外，还有更多的指令和参数来控制 nginx 中的服务器负载均衡，例如 [proxy_next_upstream](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_next_upstream)、[backup](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server)、[down](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#server) 和 [keepalive](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#keepalive)。更多信息请查看我们的[参考文档](https://nginx.org/en/docs/) 。

Last but not least, [application load balancing](https://www.nginx.com/products/nginx/load-balancing/), [application health checks](https://www.nginx.com/products/application-health-checks/), [activity monitoring](https://www.nginx.com/products/live-activity-monitoring/) and [on-the-fly reconfiguration](https://www.nginx.com/products/on-the-fly-reconfiguration/) of server groups are available as part of our paid NGINX Plus subscriptions.

The following articles describe load balancing with NGINX Plus in more detail:

- [Load Balancing with NGINX and NGINX Plus](https://www.nginx.com/blog/load-balancing-with-nginx-plus/)
- [Load Balancing with NGINX and NGINX Plus part 2](https://www.nginx.com/blog/load-balancing-with-nginx-plus-part2/)