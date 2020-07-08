Nginx Beginner's Guide 中文翻译
============================

* [启动、停止和重新加载配置](#启动停止和重新加载配置)
* [配置文件结构](#配置文件结构)
* [提供静态内容](#提供静态内容)
* [设置一个简单的代理服务器](#设置一个简单的代理服务器)
* [设置 FastCGI 代理](#设置-FastCGI-代理)

翻译自 [Nginx Beginner's Guide](https://nginx.org/en/docs/beginners_guide.html)。

本指南对 nginx 做了一个基本的介绍，并介绍了一些可以用它来完成的简单任务。这里假设读者的机器上已经安装了 nginx，如果还没有安装，请参见安装 nginx [页面](https://nginx.org/en/docs/install.html)。本指南介绍了如何启动和停止 nginx，以及重新加载它的配置，解释了配置文件的结构，并介绍了如何设置 nginx 来提供静态内容，如何将 nginx 配置为代理服务器，以及如何将它与 FastCGI 应用程序连接。

nginx 有一个主进程和几个工作进程。主进程的主要目的是读取和评估配置，并维护工作进程。工作进程进行实际的请求处理，nginx 采用基于事件的模型和依赖于操作系统的机制来有效地在工作进程之间分配请求。工作进程的数量在配置文件中定义，对于给定的配置来说，可以是固定的，也可以根据可用的 CPU 核心数自动调整（详见 [worker_processes](https://nginx.org/en/docs/ngx_core_module.html#worker_processes)）。

nginx 及其模块的工作方式是在配置文件中决定的。默认情况下，配置文件被命名为 *nginx.conf* 并放在路径为 `/usr/local/nginx/conf` ，`/etc/nginx` ，或 `/usr/local/etc/nginx` 的目录下。

# 启动、停止和重新加载配置

想要启动 nginx，运行可执行文件。一旦启动了 nginx，就可以通过使用 `-s` 参数调用可执行文件来对其进行控制。使用以下语法：

```bash
nginx -s signal
```

其中 `signal` 也可以是下列之一：

- `stop` — 快速关机
- `quit` — 优雅关机
- `reload` — 重新载入配置文件
- `reopen` — 重新打开日志文件

例如，要在等待工作进程完成对当前请求的服务的过程中停止 nginx，可以执行以下命令：

```bash
nginx -s quit
```

> 该命令应该在启动 nginx 的同一用户下执行。

在配置文件中所做的更改不会立即被应用，直到向 nginx 发送重载配置的命令或者重启 nginx 。要重新加载配置，请执行：

```bash
nginx -s reload
```

一旦主进程收到重载配置的信号，它就会检查新配置文件的语法有效性，并尝试应用其中提供的配置。如果成功，主进程启动新的工作进程，并向旧的工作进程发送消息，要求它们关闭。否则，主进程将回滚更改，并继续使用旧配置工作。旧工作进程在收到关闭命令后，会停止接受新的连接，并继续为当前的请求提供服务并持续到所有这些请求都得到服务。在此之后，旧的工作进程退出。

也可以借助 Unix 工具（如 *kill* 工具）向 nginx 进程发送信号。在下面这个例子中，一个信号被直接发送给一个具有给定进程 ID 的进程。nginx 主进程的进程 ID 默认写在 `/usr/local/nginx/logs` 或 `/var/run` 目录下的 *nginx.pid* 中。例如，如果主进程 ID 是 *1628*，要发送 `QUIT` 信号使 nginx 优雅的关闭，执行：

```bash
kill -s QUIT 1628
```

为了获得所有正在运行的 nginx 进程的列表，可以使用 *ps* 工具，例如，如下所示：

```bash
ps -ax | grep nginx
```

关于向 nginx 发送信号的更多信息，参阅 [Controlling nginx](https://nginx.org/en/docs/control.html)。

# 配置文件结构

nginx 由模块组成，这些模块由配置文件中指定的指令控制。指令分为简单指令（simple directives）和块指令（block directives）。一个简单的指令由名称和参数组成，用空格隔开并以分号（`;`）结束。块指令的结构与简单指令相同，但它的结尾不使用分号，而是用一组由括号（`{` 和 `}`）包围的附加指令。如果一个块指令在括号内有其他指令，则称为上下文（context）（示例：[events](https://nginx.org/en/docs/ngx_core_module.html#events)，[http](https://nginx.org/en/docs/http/ngx_http_core_module.html#http)，[server](https://nginx.org/en/docs/http/ngx_http_core_module.html#server)，和 [location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location)）。

放在配置文件中任何上下文之外的指令都被认为是在[主上下文（main context）](https://nginx.org/en/docs/ngx_core_module.html)中。events 和 http 指令位于主上下文中，server 位于 http 中，location 位于 server 中。

`#` 之后的一行内剩余内容会被视为注释。

# 提供静态内容

Web 服务器的一个重要任务是提供文件（如图像或静态 HTML 页面）。你将实现一个例子，根据请求的不同，文件将从不同的本地目录提供：*/data/www*（包含 HTML 文件）和 */data/images*（包含图像）。这就需要编辑配置文件，并在 [http](https://nginx.org/en/docs/http/ngx_http_core_module.html#http) 块内设置一个带有两个 [location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) 区块的 [server](https://nginx.org/en/docs/http/ngx_http_core_module.html#server) 区块。

首先，创建 */data/www* 目录，并将 *index.html* 文件与任何文本内容放入其中，并创建 */data/images* 目录，将一些图片放入其中。

接着，打开配置文件。默认的配置文件已经包含了几个 server 块的例子，大部分都被注释掉了。现在注释掉所有的区块，然后开始一个新的 server 区块。

```bash
http {
    server {
    }
}
```

一般来说，配置文件可以包括若干个 server 块，并以它们[监听](https://nginx.org/en/docs/http/ngx_http_core_module.html#listen)的端口和[服务器名称](https://nginx.org/en/docs/http/server_names.html)来[区分](https://nginx.org/en/docs/http/request_processing.html)。一旦 nginx 决定由哪个 server 处理请求，它就会根据 server 块中定义的 location 指令参数测试请求头中指定的 URI 。

将下面的 location 块添加到 server 块中：

```bash
location / {
    root /data/www;
}
```

这个 location 块指定了 "`/`" 前缀与来自请求的 URI 进行比较。对于匹配的请求，URI 将被添加到 [root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root) 指令中指定的路径，即 */data/www*，生成本地文件系统中请求文件的路径。如果有几个匹配的 location 块，nginx 会选择前缀最长的那个。上面的 location 块提供了最短的前缀，长度为 1 ，所以只有当所有其他 location 块都不能提供匹配的时候，才会使用这个区块。

接着添加第二个 location 块：

```bash
location /images/ {
    root /data;
}
```

它将匹配以 */images/* 开头的请求（`location /` 也匹配此类请求，但前缀更短）。

server 块的最终配置应该是这样的：

```bash
server {
    location / {
        root /data/www;
    }

    location /images/ {
        root /data;
    }
}
```

这已经是一个在标准端口 80 上监听的服务器的工作配置，可以在本地机器上访问 `http://localhost` 。为了响应以 */images/* 为开头的 URI 的请求，服务器将从 */data/images* 目录中发送文件。例如，在响应 `http://localhost/images/example.png` 的请求时，nginx 会发送 */data/images/example.png* 文件。如果这个文件不存在，nginx 会发送一个 404 error 的响应。不是以 */images/* 开头的 URI 请求将被映射到 */data/www* 目录。例如，在响应 `http://localhost/some/example.html` 请求时，nginx 将发送 */data/www/some/example.html* 文件。

要应用新的配置，如果还没有启动 nginx，请启动 nginx，或者向 nginx 的主进程发送重载信号，执行以下命令：

```bash
nginx -s reload
```

> 如果有什么事情没有达到预期的效果，你可以尝试在 `/usr/local/nginx/logs` 或 `/var/log/nginx` 目录下的 *access.log* 和 *error.log* 文件中查找原因。

# 设置一个简单的代理服务器

nginx 经常被使用的一个方法是把它设置为代理服务器，也就是一台服务器接收请求，把请求传递给代理服务器，从代理服务器上获取响应，然后发送给客户端。

我们将配置一个基本的代理服务器，它从本地目录中发送带有文件的图片请求，并将所有其他请求发送到代理服务器。在这个例子中，这两个服务器将被定义在单个 nginx 实例上。

首先，定义代理服务器，在 nginx 的配置文件中增加一个 server 块，内容如下：

```bash
server {
    listen 8080;
    root /data/up1;

    location / {
    }
}
```

这将是一个简单的监听 8080 端口的服务器（之前由于使用标准端口 80，所以没有指定 listen 指令），并将所有请求映射到本地文件系统的 */data/up1* 目录。创建这个目录，并将 *index.html* 文件放入其中。注意，root 指令是放在 server 上下文中的。当选择用于服务请求的 location 块不包括自己的 root 指令时，就会使用这种 root 指令。

接下来，使用上一节的服务器配置，并将其修改为代理服务器配置。在第一个 location 块中，放入[proxy_pass](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) 指令，并在参数中指定代理服务器的协议、名称和端口（在我们的例子中，它是 `http://localhost:8080`）。

```bash
server {
    location / {
        proxy_pass http://localhost:8080;
    }

    location /images/ {
        root /data;
    }
}
```

我们将修改第二个 location 块，目前将带有 */images/* 前缀的请求映射到 */data/images* 目录下的文件，使其与具有典型文件扩展名的图片请求相匹配。修改后的 location 块看起来像这样：

```bash
location ~ \.(gif|jpg|png)$ {
   root /data/images;
}
```

参数是一个正则表达式，匹配所有以 *.gif*、*.jpg* 或 *.png* 结尾的 URI 。正则表达式应该在前面加上`~`。相应的请求将被映射到 */data/images* 目录。

当 nginx 选择一个 location 块来服务一个请求时，它首先检查指定前缀的 [location](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) 指令，记住前缀最长的 location，然后检查正则表达式。如果与正则表达式匹配，nginx 就选择这个 location，否则，就选择之前记住的那个。

代理服务器的最终配置如下：

```bash
server {
    location / {
        proxy_pass http://localhost:8080/;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

该服务器将过滤以 *.gif*、*.jpg* 或 *.png* 结尾的请求，并将其映射到 */data/images* 目录（通过在 root 指令的参数中添加 URI），并将所有其他请求传递给上面配置的代理服务器。

要应用新的配置，按照前面章节的描述向 nginx 发送重载信号。

还有[很多](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)指令可以用来进一步配置代理连接。

# 设置 FastCGI 代理

nginx 可以用来路由请求到 FastCGI 服务器，这些服务器运行的应用程序是由各种框架和编程语言（如 PHP）构建的。

最基本的能与一台 FastCGI 服务器一起工作的 nginx 配置包括使用 [fastcgi_pass](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_pass) 指令代替 proxy_pass 指令，以及使用 [fastcgi_param](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_param) 指令来设置传递给 FastCGI 服务器的参数。假设 FastCGI 服务器可以在 `localhost:9000` 上访问。以上一节的代理配置为基础，用 fastcgi_pass 指令替换 proxy_pass 指令，并将参数改为 `localhost:9000`。在 PHP 中，SCRIPT_FILENAME 参数用于确定脚本名称，QUERY_STRING 参数用于传递请求参数。最终的配置将是：

```bash
server {
    location / {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param QUERY_STRING    $query_string;
    }

    location ~ \.(gif|jpg|png)$ {
        root /data/images;
    }
}
```

这将设置一个服务器，通过 FastCGI 协议将所有请求（静态图像请求除外）路由到运行在 `localhost:9000` 上的代理服务器。