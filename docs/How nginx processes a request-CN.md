How nginx processes a request 中文翻译
===

* [基于名称的虚拟服务器](#基于名称的虚拟服务器)
* [如何防止用未定义的服务器名称处理请求](#如何防止用未定义的服务器名称处理请求)
* [基于名称和 IP 的混合虚拟服务器](#基于名称和-IP-的混合虚拟服务器)
* [一个简单的 PHP 网站配置](#一个简单的-PHP-网站配置)

翻译自 [How nginx processes a request](https://nginx.org/en/docs/http/request_processing.html)。

---

# 基于名称的虚拟服务器

nginx 首先确定哪个服务器应处理请求。 让我们从一个简单的配置开始，其中三个虚拟服务器都在端口 `*:80` 上侦听：

```nginx
server {
    listen      80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      80;
    server_name example.com www.example.com;
    ...
}
```

在这个配置中，nginx 只测试请求的头字段 "Host"，以确定请求应该被路由到哪个服务器。如果它的值不符合任何服务器名称，或者请求中根本不包含这个请求头字段，那么 nginx 将把请求发送到这个端口的默认服务器中。在上面的配置中，第一个是默认服务器，这是 nginx 的标准默认行为。也可以通过 [listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 指令中的 `default_server` 参数，明确设置哪个服务器为默认服务器：

```nginx
server {
    listen      80 default_server;
    server_name example.net www.example.net;
    ...
}
```

> `default_server` 参数从 0.8.21 版本开始就有了。在更早期版本中，应该使用 `default` 参数。

注意，默认的服务器是 listen 端口的属性，而不是 server 名称的属性。后面会有更多的介绍。

# 如何防止用未定义的服务器名称处理请求

如果不允许没有 "Host" 头部字段的请求，可以定义一个仅丢弃请求的服务器：

```nginx
server {
    listen      80;
    server_name "";
    return      444;
}
```

这里，服务器名称设置为空字符串并将匹配没有 "Host" 头字段的请求，同时返回一个特殊的 nginx 非标准代码 *444* 来关闭连接。

> 从 0.8.48 版本开始，这是 server 名称的默认设置，所以可以省略 `server_name ""` 。在早期的版本中，机器的 hostname 被用作默认的服务器名。

# 基于名称和 IP 的混合虚拟服务器

让我们看看一个更复杂的配置，一些虚拟服务器监听不同的地址：

```nginx
server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      192.168.1.1:80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      192.168.1.2:80;
    server_name example.com www.example.com;
    ...
}
```

在这个配置中，nginx 首先根据 [server](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块的 [listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 指令测试请求的 IP 地址和端口。然后，它根据与 IP 地址和端口相匹配的 [server](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块的 [server_name](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 条目测试请求的 "Host" 头字段。如果找不到服务器名，该请求将由默认服务器处理。例如，在 `192.168.1.1:80` 端口上收到的 `www.example.com` 请求将由 `192.168.1.1:80` 端口的默认服务器处理，也就是第一个服务器，因为这个端口没有定义 `www.example.com`。

前面已经说过，默认服务器是监听端口的一个属性，不同的端口可以定义不同的默认服务器：

```nginx
server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      192.168.1.1:80 default_server;
    server_name example.net www.example.net;
    ...
}

server {
    listen      192.168.1.2:80 default_server;
    server_name example.com www.example.com;
    ...
}
```

# 一个简单的 PHP 网站配置

现在让我们来看看 nginx 是如何为一个典型的、简单的 PHP 站点选择处理请求的 *location* ：

```nginx
server {
    listen      80;
    server_name example.org www.example.org;
    root        /data/www;

    location / {
        index   index.html index.php;
    }

    location ~* \.(gif|jpg|png)$ {
        expires 30d;
    }

    location ~ \.php$ {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME
                      $document_root$fastcgi_script_name;
        include       fastcgi_params;
    }
}
```

nginx 首先搜索最具体的前缀位置，而不考虑列出的顺序。在上面的配置中，唯一的前缀位置是 `/`，由于它与任何请求相匹配，它将被用于最后的手段。然后 nginx 按照配置文件中列出的顺序检查正则表达式给出的位置。搜索将在遇到第一个匹配的表达式之后停止，nginx 将使用这个位置。如果没有正则表达式与请求匹配，那么 nginx 使用前面找到的最具体的前缀位置。

请注意，所有类型的位置都只测试请求行中没有参数的 URI 部分。这样做是因为查询字符串中的参数可能以多种方式给出，例如：

```http
/index.php?user=john&page=1
/index.php?page=1&user=john
```

此外，任何人都可能要求查询字符串中的任何东西：

```http
/index.php?page=1&something+else&user=john
```

现在让我们看看在上面的配置中如何处理请求：

- 请求 */logo.gif* 先由 `/` 匹配，再由正则表达式 `\.(gif|jpg|png)$` 匹配，因此这个请求由后一个location 处理。使用指令 `root /data/www` 将请求映射到文件 */data/www/logo.gif*，并将该文件发送到客户端。
- 请求 */index.php* 也是先由 `/` 匹配，然后由正则表达式 `\.(php)$` 匹配。因此，它由后一个 location 处理，并将请求传递给在 `localhost:9000` 上监听的 FastCGI 服务器。[fastcgi_param](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_param) 指令将 FastCGI 参数 `SCRIPT_FILENAME` 设置为 */data/www/index.php*，之后 FastCGI 服务器执行该文件。变量 `$document_root` 等于 [root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root) 指令的值，变量 `$fastcgi_script_name` 等于请求 URI，即 */index.php*。
- 请求 */about.html* 只与 `/` 相匹配，因此它在这个 location 进行处理。使用指令 `root /data/www` 将请求映射到文件 */data/www/about.html*，并将该文件发送给客户端。
- 处理 */* 的请求比较复杂。它只与 `/` 匹配，因此，它由这个 location 处理。然后，[index](https://nginx.org/en/docs/http/ngx_http_index_module.html#index) 指令根据其参数和 `root /data/www` 指令测试判断是否存在 *index* 文件。如果文件 */data/www/index.html* 不存在，而文件 */data/www/index.php* 存在，那么该指令会做一个内部重定向到 */index.php*，然后 nginx 会像客户端发送请求一样重新搜索这些 location。正如我们之前所看到的，重定向的请求最终将由 FastCGI 服务器处理。

作者：Igor Sysoev

编辑：Brian Mercer