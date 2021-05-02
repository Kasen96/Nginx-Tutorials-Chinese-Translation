Server names 翻译
===

* [通配符名称](#通配符名称)
* [正则表达式名称](#正则表达式名称)
* [杂项名称](#杂项名称)
* [国际化名称](#国际化名称)
* [优化](#优化)
* [兼容性](#兼容性)

翻译自 [Server names](https://nginx.org/en/docs/http/server_names.html)。

---

服务器名称是使用 [server_name](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 指令定义的，它决定了一个给定请求使用哪个 [server](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块。参见 "[How nginx processes a request](https://nginx.org/en/docs/http/request_processing.html)"。它们可以使用确切的名称、通配符名称或正则表达式来定义：

```nginx
server {
    listen       80;
    server_name  example.org  www.example.org;
    ...
}

server {
    listen       80;
    server_name  *.example.org;
    ...
}

server {
    listen       80;
    server_name  mail.*;
    ...
}

server {
    listen       80;
    server_name  ~^(?<user>.+)\.example\.net$;
    ...
}
```

当通过名称搜索虚拟服务器时，如果名称匹配多个指定的变体，例如通配符名称和正则表达式都匹配，将选择第一个匹配的变体，顺序如下：

1. 确切的名称
2. 以星号开头的最长通配符名称，如 `*.example.org`
3. 以星号结尾的最长通配符名称，如 `mail.*`
4. 第一个匹配的正则表达式(按照在配置文件中出现的顺序)

# 通配符名称

通配符名称只能在名称的开头或结尾处包含星号，并且只能在点的边上包含星号。名字 `www.*.example.org` 和 `w*.example.org` 是无效的。但是，这些名称可以使用正则表达式来指定，例如，`~^www\..+\.example\.org$` 和 `~^w.*\.example\.org$` 。一个星号可以匹配多个名称部分。`*.example.org` 不仅匹配 `www.example.org` 也可以匹配 `www.sub.example.org` 。

形式为 `.example.org` 的特殊通配符名称可以用来匹配确切的名称 `example.org` 和通配符名称 `*.example.org` 。

# 正则表达式名称

nginx 使用的正则表达式与 Perl 编程语言 (PCRE) 的正则表达式兼容。要使用正则表达式，服务器名称必须以波浪号字符开头：

```nginx
server_name  ~^www\d+\.example\.net$;
```

否则它将被视为确切名称，或者如果表达式包含星号，将被视为通配符名称（并且很可能被视为无效的名称）。不要忘记设置 `^` 和 `$` 锚点。它们在语法上不是必需的，但在逻辑上是必需的。另外请注意域名中的点应以反斜杠转义。 包含字符 `{` 和 `}` 的正则表达式应加引号：

```nginx
server_name  "~^(?<name>\w\d{1,3}+)\.example\.net$";
```

否则，nginx 将无法启动并显示错误消息：

```bash
directive "server_name" is not terminated by ";" in ...
```

命名的正则表达式捕获可以在以后用作变量：

```nginx
server {
    server_name   ~^(www\.)?(?<**domain**>.+)$;

    location / {
        root   /sites/**$domain**;
    }
}
```

PCRE 库使用以下语法支持命名捕获：

- `?<name>`   Perl 5.10 兼容语法，自 PCRE-7.0 起受支持
- `?'name'`   Perl 5.10 兼容语法，自 PCRE-7.0 起受支持
- `?P<name>` Python 兼容语法，自 PCRE-4.0 起受支持

如果 nginx 无法启动并显示错误消息：

```bash
pcre_compile() failed: unrecognized character after (?< in ...
```

这意味着 PCRE 库较旧，应尝试使用语法 `?P<name>`。 捕获也可以以数字形式使用：

```nginx
server {
    server_name   ~^(www\.)?(.+)$;

    location / {
        root   /sites/**$2**;
    }
}
```

但是，这种用法应限于简单的情况（如上述情况），因为数字参考很容易被覆盖。

# 杂项名称

有些服务器名称需要被特殊对待。

如果要求在一个非默认的 [server](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块中处理没有 “Host” 头字段的请求，则应指定一个空名称：

```nginx
server {
    listen       80;
    server_name  example.org  www.example.org  "";
    ...
}
```

如果在 [server](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块中未定义 [server_name](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name)，则 nginx 使用空名字作为服务器名称。

> 在这种案例中 nginx 0.8.48 以下的版本使用机器的主机名作为服务器名。

如果服务器名称定义为 `$hostname` (0.9.4)，则使用计算机的主机名。

如果有人使用 IP 地址而不是服务器名称发出请求，则 “Host” 请求头字段将包含 IP 地址，并且可以使用 IP 地址作为服务器名称来处理请求：

```nginx
server {
    listen       80;
    server_name  example.org
                 www.example.org
                 ""
                 192.168.1.1
                 ;
    ...
}
```

在通用（catch-all）服务器示例中，可以看到一个奇怪的名称  `_` ：

```nginx
server {
    listen       80  default_server;
    server_name  _;
    return       444;
}
```

这个名字并没有什么特别之处，它只是无数无效域名中的一个，与任何实际名称都没有交集。 同样可以使用其他无效名称，例如 `--` 和 `!@＃`。

nginx 在 0.6.25 之前的版本都支持  `*` 这个特殊的名字，而这个名字被错误地解释为一个通用的名字。它从来没有作为一个万能或通配符服务器名称的功能。相反，它提供了现在由 [server_name_in_redirect](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name_in_redirect) 指令提供的功能。特殊名称 `*` 现在已被废弃，应使用 [server_name_in_redirect](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name_in_redirect) 指令。需要注意的是，没有办法使用 [server_name](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 指令来指定通用名称或默认服务器。这是 [listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 指令的属性，而不是 [server_name](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 指令的属性。参见 "[How nginx processes a request](https://nginx.org/en/docs/http/request_processing.html)"。可以定义在端口 `*:80` 和 `*:8080` 上监听的服务器，并指示其中一个是端口 `*:8080` 的默认服务器，而另一个是端口 `*:80` 的默认服务器：

```nginx
server {
    listen       80;
    listen       8080  default_server;
    server_name  example.net;
    ...
}

server {
    listen       80  default_server;
    listen       8080;
    server_name  example.org;
    ...
}
```

# 国际化名称

国际化域名（[IDNs](https://en.wikipedia.org/wiki/Internationalized_domain_name)）应在 [server_name](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_name) 指令中使用 ASCII（Punycode）表示。

```nginx
server {
    listen       80;
    server_name  xn--e1afmkfd.xn--80akhbyknj4f;  # пример.испытание
    ...
}
```

# 优化

确切的名称、以星号开头的通配符名称和以星号结尾的通配符名称都存储和监听端口绑定的三个哈希表中。在配置阶段就对哈希表的大小进行了优化，这样就可以在最少的 CPU 缓存丢失下找到一个名字。设置哈希表的细节在独立的[文件](https://nginx.org/en/docs/hash.html)中提供。

首先搜索确切的名称哈希表。如果没有找到名字，则搜索以星号开头的通配符名字的哈希表。如果没有找到名字，则搜索以星号结尾的通配符名字的哈希表。

搜索通配符名称哈希表比搜索精确名称哈希表慢，因为名称是按域名部分搜索的。请注意，特殊的通配符形式 `.example.org` 是存储在通配符名称哈希表中，而不是存储在精确名称哈希表中。

正则表达式是依次测试的，因此是最慢的方法，而且是不可扩展的。

由于这些原因，最好尽可能使用准确的名称。例如，如果一个服务器最常被请求的名字是 `example.org` 和 `www.example.org`，那么显式定义它们会更有效率：

```nginx
server {
    listen       80;
    server_name  example.org  www.example.org  *.example.org;
    ...
}
```

比起使用简化形式：

```nginx
server {
    listen       80;
    server_name  .example.org;
    ...
}
```

如果定义了大量的服务器名，或者定义了异常长的服务器名，那么在 *http* 层调整 [server_names_hash_max_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_max_size) 和 [server_names_hash_bucket_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_bucket_size) 指令就变得很有必要。[server_names_hash_bucket_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_bucket_size) 指令的默认值可能等于 32，或 64，或其他值，这取决于 CPU 缓存行大小。如果默认值为 32，并且服务器名称定义为 `too.long.server.name.example.org`，那么 nginx 将无法启动并显示错误信息：

```bash
could not build the server_names_hash,
you should increase server_names_hash_bucket_size: 32
```

在这种情况下，指令值应增加到 2 的下一个幂：

```nginx
http {
    server_names_hash_bucket_size  64;
    ...
```

如果定义了大量的服务器名称，会出现另一条错误信息：

```bash
could not build the server_names_hash,
you should increase either server_names_hash_max_size: 512
or server_names_hash_bucket_size: 32
```

在这种情况下，首先尝试将 [server_names_hash_max_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_max_size) 设置为一个接近服务器名称数量的数字。只有当这样做没有帮助，或者 nginx 的启动时间过长时，才可以尝试增加 [server_names_hash_bucket_size](https://nginx.org/en/docs/http/ngx_http_core_module.html#server_names_hash_bucket_size) 。

如果一个服务器是监听端口的唯一服务器，那么 nginx 根本不会测试服务器名称（也不会为监听端口建立哈希表）。但是，有一个例外。如果一个服务器名是一个带捕获的正则表达式，那么 nginx 必须执行该表达式才能获得捕获。

# 兼容性

- 从 0.9.4 开始支持特殊的服务器名称 `$hostname` 。
- 从 0.8.48 开始，默认的服务器名是一个空名 ""。
- 从 0.8.25 开始支持命名的正则表达式服务器名称捕获。
- 从 0.7.40 开始支持正则表达式服务器名称捕获。
- 从 0.7.12 开始支持空服务器名 ""。
- 从 0.6.25 开始，支持使用通配符服务器名或正则表达式作为第一个服务器名。
- 从 0.6.7 开始支持正则表达式服务器名称。
- 从 0.6.0 开始支持通配符形式 `example.*` 。
- 从 0.3.18 开始支持 `.example.org` 这一特殊形式。
- 从 0.1.13 开始支持通配符形式 `*.example.org` 。

作者：Igor Sysoev

编辑：Brian Mercer