How nginx processes a TCP/UDP session 中文翻译
===

翻译自 [How nginx processes a TCP/UDP session](https://nginx.org/en/docs/stream/stream_processing.html) 。

---

一个来自客户端的 TCP/UDP 会话按连续的步骤进行处理，称为**阶段**：

- Post-accept

    接受客户端连接后的第一个阶段。[ngx_stream_realip_module](https://nginx.org/en/docs/stream/ngx_stream_realip_module.html) 模块在此阶段被调用。

- Pre-access

    初步检查访问情况。[ngx_stream_limit_conn_module](https://nginx.org/en/docs/stream/ngx_stream_limit_conn_module.html) 模块在此阶段被调用。

- Access

    在实际数据处理前限制客户端访问。[ngx_stream_access_module](https://nginx.org/en/docs/stream/ngx_stream_access_module.html) 模块在此阶段被调用。

- SSL

    TLS/SSL 终止。[ngx_stream_ssl_module](https://nginx.org/en/docs/stream/ngx_stream_ssl_module.html) 模块在此阶段被调用。

- Preread

    读取初始数据字节到 [preread_buffer](https://nginx.org/en/docs/stream/ngx_stream_core_module.html#preread_buffer_size)，以便 [ngx_stream_ssl_preread_module](https://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html) 等模块在处理前分析数据。

- Content

    强制性阶段，在这个阶段数据被实际处理，通常是[代理](https://nginx.org/en/docs/stream/ngx_stream_proxy_module.html)到[上游](https://nginx.org/en/docs/stream/ngx_stream_upstream_module.html)服务器，或者将指定的值[返回](https://nginx.org/en/docs/stream/ngx_stream_return_module.html)给客户端。

- Log

    记录客户端会话处理结果的最后一个阶段。[ngx_stream_log_module](https://nginx.org/en/docs/stream/ngx_stream_log_module.html) 模块在此阶段被调用。