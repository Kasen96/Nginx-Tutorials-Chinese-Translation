# Configuring HTTPS servers 中文翻译

* [HTTPS 服务器优化](#HTTPS-服务器优化)
* [SSL 证书链](#SSL-证书链)
* [单个 HTTP/HTTPS 服务器](#单个-HTTP/HTTPS-服务器)
* [基于名称的 HTTPS 服务器](#基于名称的-HTTPS-服务器)
  * [具有多个名称的 SSL 证书](#具有多个名称的-SSL-证书)
  * [服务器名称指示](#服务器名称指示)
* [兼容性](#兼容性)

翻译自 [Configuring HTTPS servers](https://nginx.org/en/docs/http/configuring_https_servers.html) 。

---

要配置 HTTPS 服务器，必须在 [server](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) 块的[监听套接字](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)上启用 *ssl* 参数，并指定[服务器证书](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_certificate)和[私钥](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_certificate_key)文件的位置：

```nginx
server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.crt;
    ssl_certificate_key www.example.com.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ...
}
```

服务器证书是一个公共实体。它被发送到每一个连接到服务器的客户端。私钥是一个安全实体，应该存储在一个限制访问的文件中，但是它必须可以被 nginx 的主进程读取。私钥也可以和证书存放在同一个文件中：

```nginx
    ssl_certificate     www.example.com.cert;
    ssl_certificate_key www.example.com.cert;
```

在这种情况下，文件的访问权限也应该受到限制。虽然证书和密钥存储在一个文件中，但只有证书会被发送到客户端。

指令 [ssl_protocols](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_protocols) 和 [ssl_ciphers](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_ciphers) 可以用来限制连接，使其只包含 SSL/TLS 的加强版本和密码。默认情况下，nginx 使用 `ssl_protocols TLSv1 TLSv1.1 TLSv1.2` 和 `ssl_ciphers HIGH:!aNULL:!MD5` ，所以一般不需要明确配置它们。需要注意的是，这些指令的默认值是经过多次[修改](https://nginx.org/en/docs/http/configuring_https_servers.html#compatibility)的。

# HTTPS 服务器优化

SSL 操作会消耗额外的 CPU 资源。在多处理器系统中应该运行多个 [worker processes](https://nginx.org/en/docs/ngx_core_module.html#worker_processes)，数量不应少于可用的 CPU 核数。最耗费 CPU 的操作是 SSL 握手。有两种方法可以尽量减少每个客户端的这些操作次数：第一种是启用 [keepalive](https://nginx.org/en/docs/http/ngx_http_core_module.html#keepalive_timeout) 连接，通过一个连接发送多个请求；第二种是重用 SSL 会话参数以避免并行和后续连接的 SSL 握手。会话存储在 worker 之间共享的 SSL 会话缓存中，并由 [ssl_session_cache](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_cache) 指令配置。1 MB 缓存大约包含 4000 个会话。默认的缓存超时为 5 分钟。可以通过使用 [ssl_session_timeout](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_session_timeout) 指令来增加。下面是为具有 10 MB 共享会话缓存的多核系统优化的示例配置：

```nginx
worker_processes auto;

http {
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    server {
        listen              443 ssl;
        server_name         www.example.com;
        keepalive_timeout   70;

        ssl_certificate     www.example.com.crt;
        ssl_certificate_key www.example.com.key;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;
        ...
```

# SSL 证书链

有些浏览器可能会抱怨由知名证书颁发机构签署的证书，而另一些浏览器可能会接受该证书而不会出现问题。发生这种情况的原因是，发证机构使用中间证书签署了服务器证书，而该证书并不存在于与特定浏览器一起分发的知名可信证书机构的证书库中。在这种情况下，权威机构提供了链式证书捆绑包，这些证书应与已签名的服务器证书串联在一起。服务器证书必须出现在组合文件中的链接证书之前：

```bash
$ cat www.example.com.crt bundle.crt > www.example.com.chained.crt
```

由此产生的文件应该在 [ssl_certificate](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_certificate) 指令中使用：

```nginx
server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.chained.crt;
    ssl_certificate_key www.example.com.key;
    ...
}
```

如果服务器证书和捆绑包的顺序不对，nginx 将无法启动并显示错误信息：

```bash
SSL_CTX_use_PrivateKey_file(" ... /www.example.com.key") failed
   (SSL: error:0B080074:x509 certificate routines:
    X509_check_private_key:key values mismatch)
```

因为 nginx 尝试将私有密钥与捆绑包的第一个证书一起使用，而不是服务器证书。

浏览器通常会存储他们收到的中间证书，这些证书由受信任的权威机构签署，因此主动使用的浏览器可能已经拥有所需的中间证书，可能不会抱怨发送的证书没有链式捆绑。为了确保服务器发送完整的证书链，可以使用 *openssl* 命令行，例如：

```bash
$ openssl s_client -connect www.godaddy.com:443
...
Certificate chain
 0 s:/C=US/ST=Arizona/L=Scottsdale/1.3.6.1.4.1.311.60.2.1.3=US
     /1.3.6.1.4.1.311.60.2.1.2=AZ/O=GoDaddy.com, Inc
     /OU=MIS Department/CN=www.GoDaddy.com
     /serialNumber=0796928-7/2.5.4.15=V1.0, Clause 5.(b)
   i:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc.
     /OU=http://certificates.godaddy.com/repository
     /CN=Go Daddy Secure Certification Authority
     /serialNumber=07969287
 1 s:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc.
     /OU=http://certificates.godaddy.com/repository
     /CN=Go Daddy Secure Certification Authority
     /serialNumber=07969287
   i:/C=US/O=The Go Daddy Group, Inc.
     /OU=Go Daddy Class 2 Certification Authority
 2 s:/C=US/O=The Go Daddy Group, Inc.
     /OU=Go Daddy Class 2 Certification Authority
   i:/L=ValiCert Validation Network/O=ValiCert, Inc.
     /OU=ValiCert Class 2 Policy Validation Authority
     /CN=http://www.valicert.com//emailAddress=info@valicert.com
...
```

当使用 [SNI](https://nginx.org/en/docs/http/configuring_https_servers.html#sni) 测试配置时，指定 `-servername` 选项很重要，因为 *openssl* 默认不使用 SNI。

> In this example the subject (“s”) of the www.GoDaddy.com server certificate #0 is signed by an issuer (“i”) which itself is the subject of the certificate #1, which is signed by an issuer which itself is the subject of the certificate #2, which signed by the well-known issuer ValiCert, Inc. whose certificate is stored in the browsers’ built-in certificate base (that lay in the house that Jack built).

在这个例子中，`www.GoDaddy.com` 服务器证书 `#0` 的主体( `s` )是由发行人( `i` )签署的，该发行人本身就是证书 `#1` 的主体，而证书 `#1` 由本身是证书 `#2` 主体的发行人签发，证书 `#2` 由著名的 ValiCert 公司签发，发行的证书存储在浏览器的内置证书库中。

如果尚未添加证书捆绑包，则仅显示服务器证书 `#0` 。

# 单个 HTTP/HTTPS 服务器

可以配置一个同时处理 HTTP 和 HTTPS 请求的服务器：

```nginx
server {
    listen              80;
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.crt;
    ssl_certificate_key www.example.com.key;
    ...
}
```

在 0.7.14 之前，不能选择性地为单个监听套接字启用 SSL，如上图所示。只能使用 [ssl](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl) 指令为整个服务器启用，这使得无法设置单个 HTTP/HTTPS 服务器。为了解决这个问题，增加了 [listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 指令的 *ssl* 参数。因此不鼓励在现代版本中使用 [ssl](https://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl) 指令。

# 基于名称的 HTTPS 服务器

配置两个或多个侦听单个 IP 地址的 HTTPS 服务器时，会出现一个常见问题：

```nginx
server {
    listen          443 ssl;
    server_name     www.example.com;
    ssl_certificate www.example.com.crt;
    ...
}

server {
    listen          443 ssl;
    server_name     www.example.org;
    ssl_certificate www.example.org.crt;
    ...
}
```

在这种配置下，无论请求的服务器名称是什么，浏览器都会收到默认服务器的证书，如 `www.example.com` 。这是由 SSL 协议的行为造成的。SSL 连接是在浏览器发送 HTTP 请求之前建立的，而 nginx 并不知道所请求的服务器名称。因此，它可能只提供默认服务器的证书。

最古老、最稳健的解决方法是为每个 HTTPS 服务器分配一个独立的 IP 地址：

```nginx
server {
    listen          192.168.1.1:443 ssl;
    server_name     www.example.com;
    ssl_certificate www.example.com.crt;
    ...
}

server {
    listen          192.168.1.2:443 ssl;
    server_name     www.example.org;
    ssl_certificate www.example.org.crt;
    ...
}
```

## 具有多个名称的 SSL 证书

还有其他的方式，允许在几个 HTTPS 服务器之间共享一个 IP 地址。然而，所有这些方法都有其缺点。一种方法是使用一个证书并在 *SubjectAltName* 证书字段中包含几个名字，例如，`www.example.com` 和 `www.example.org` 。但是，SubjectAltName 字段的长度是有限的。

另一种方法是使用带有通配符名称的证书，例如，`*.example.org` 。通配符证书可以保证指定域名的所有子域的安全，但只在一个级别上。该证书匹配 `www.example.org` ，但不匹配 `example.org` 和 `www.sub.example.org` 。这两种方法也可以结合使用。证书可以在 SubjectAltName 字段中包含确切的名称和通配符名称，例如 `example.org` 和 `*.example.org` 。

最好将一个有多个名字的证书文件及其私钥文件放在 *http* 级别的配置中，在所有服务器中继承它们的单一内存拷贝：

```nginx
ssl_certificate     common.crt;
ssl_certificate_key common.key;

server {
    listen          443 ssl;
    server_name     www.example.com;
    ...
}

server {
    listen          443 ssl;
    server_name     www.example.org;
    ...
}
```

## 服务器名称指示

在一个 IP 地址上运行多个 HTTPS 服务器的更通用的解决方案是 [TLS 服务器名称指示扩展](http://en.wikipedia.org/wiki/Server_Name_Indication)（SNI，RFC 6066），它允许浏览器在 SSL 握手过程中传递一个请求的服务器名称，因此，服务器将知道它应该为连接使用哪个证书。目前大多数现代浏览器都[支持](http://en.wikipedia.org/wiki/Server_Name_Indication#Support) SNI，不过一些老的或特殊的客户端可能不会使用。

在 SNI 中只能传递域名，但是，如果请求中包含 IP 地址，一些浏览器可能会错误地传递服务器的 IP 地址作为其名称。我们不应该依赖这一点。

为了在 nginx 中使用 SNI，必须在已构建 nginx 二进制文件的 OpenSSL 库以及运行时动态链接的库中都支持它。如果使用配置选项 `--enable-tlsext` ，OpenSSL 从 *0.9.8f* 版本开始支持 SNI。从 OpenSSL 0.9.8j 开始，这个选项是默认启用的。如果 nginx 是在支持 SNI 的情况下构建的，那么 nginx 在使用 `-V` 开关运行时就会显示这个选项：

```bash
$ nginx -V
...
TLS SNI support enabled
...
```

但是，如果启用 SNI 的 nginx 动态链接到一个不支持 SNI 的 OpenSSL 库，nginx 会显示警告：

```bash
nginx was built with SNI support, however, now it is linked
dynamically to an OpenSSL library which has no tlsext support,
therefore SNI is not available
```

# 兼容性

- 从 0.8.21 和 0.7.62 开始，SNI 支持状态已经由 `-V` 开关显示。
- 从 0.7.14 开始，[listen](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen) 指令的 *ssl* 参数就已经被支持了。在 0.8.21 之前，它只能和 *default* 参数一起指定。
- 从 0.5.23 开始支持 SNI。
- 从 0.5.6 开始支持共享 SSL 会话缓存。
- 1.9.1 及以后的版本：默认的 SSL 协议是 TLSv1、TLSv1.1 和 TLSv1.2（如果 OpenSSL 库支持）。
- 0.7.65、0.8.19 及以后的版本：默认的 SSL 协议是 SSLv3、TLSv1、TLSv1.1 和 TLSv1.2（如果 OpenSSL 库支持）。
- 0.7.64、0.8.18 及更早的版本：默认的 SSL 协议是 SSLv2、SSLv3 和 TLSv1。
- 1.0.5 及以后的版本：默认 SSL 密码为 `HIGH:!aNULL:!MD5` 。
- 0.7.65、0.8.20 及以后版本：默认 SSL 密码为 `HIGH:!ADH:!MD5` 。
- 0.8.19 版本：默认 SSL 密码为 `ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM` 。
- 0.7.64、0.8.18 及更早的版本：默认的 SSL 密码为 `ALL:!ADH:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP` 。

作者：Igor Sysoev

编辑：Brian Mercer