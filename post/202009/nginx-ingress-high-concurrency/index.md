---
title: "Nginx Ingress 高并发实践"
# Date published
date: 2020-09-02T20:50:00+08:00

# Date updated
lastmod: 2020-09-02T20:50:00+08:00

draft: false

# Show this page in the Featured widget?
featured: false

authors:
- admin

tags:
- kubernetes

---

{{% toc %}}

## 概述

Nginx Ingress Controller 基于 Nginx 实现了 Kubernetes Ingress API，Nginx 是公认的高性能网关，但如果不对其进行一些参数调优，就不能充分发挥出高性能的优势。之前我们在 [Nginx Ingress on TKE 部署最佳实践](https://imroc.cc/posts/nginx-ingress-on-tke/) 一文中讲了 Nginx Ingress 在 TKE 上部署最佳实践，涉及的部署 YAML 其实已经包含了一些性能方面的参数优化，只是没有提及，本文将继续展开介绍针对 Nginx Ingress 的一些全局配置与内核参数调优的建议，可用于支撑我们的高并发业务。

## 内核参数调优

我们先看下如何对 Nginx Ingress 进行内核参数调优，设置内核参数的方法可以用 initContainers 的方式，后面会有示例。

### 调大连接队列的大小

进程监听的 socket 的连接队列最大的大小受限于内核参数  `net.core.somaxconn`，在高并发环境下，如果队列过小，可能导致队列溢出，使得连接部分连接无法建立。要调大 Nginx Ingress 的连接队列，只需要调整 somaxconn 内核参数的值即可，但我想跟你分享下这背后的相关原理。

进程调用 listen 系统调用来监听端口的时候，还会传入一个 backlog 的参数，这个参数决定 socket 的连接队列大小，其值不得大于 somaxconn 的取值。Go 程序标准库在 listen 时，默认直接读取 somaxconn 作为队列大小，但 Nginx 监听 socket 时没有读取 somaxconn，而是有自己单独的参数配置。在 `nginx.conf` 中 listen 端口的位置，还有个叫 backlog 参数可以设置，它会决定 nginx listen 的端口的连接队列大小。

``` nginx.conf
server {
    listen  80  backlog=1024;
    ...
```

如果不设置，backlog 在 linux 上默认为 511:

```
backlog=number
   sets the backlog parameter in the listen() call that limits the maximum length for the queue of pending connections. By default, backlog is set to -1 on FreeBSD, DragonFly BSD, and macOS, and to 511 on other platforms.
```

也就是说，即便你的 somaxconn 配的很高，nginx 所监听端口的连接队列最大却也只有 511，高并发场景下可能导致连接队列溢出。

不过这个在 Nginx Ingress 这里情况又不太一样，因为 Nginx Ingress Controller 会自动读取 somaxconn 的值作为 backlog 参数写到生成的 nginx.conf 中: https://github.com/kubernetes/ingress-nginx/blob/controller-v0.34.1/internal/ingress/controller/nginx.go#L592

也就是说，Nginx Ingress 的连接队列大小只取决于 somaxconn 的大小，这个值在 TKE 默认为 4096，建议给 Nginx Ingress 设为 65535: `sysctl -w net.core.somaxconn=65535`。

### 扩大源端口范围

高并发场景会导致 Nginx Ingress 使用大量源端口与 upstream 建立连接，源端口范围从 `net.ipv4.ip_local_port_range` 这个内核参数中定义的区间随机选取，在高并发环境下，端口范围小容易导致源端口耗尽，使得部分连接异常。TKE 环境创建的 Pod 源端口范围默认是 32768-60999，建议将其扩大，调整为 1024-65535: `sysctl -w net.ipv4.ip_local_port_range="1024 65535"`。

### TIME_WAIT 复用

如果短连接并发量较高，它所在 netns 中 TIME_WAIT 状态的连接就比较多，而 TIME_WAIT 连接默认要等 2MSL 时长才释放，长时间占用源端口，当这种状态连接数量累积到超过一定量之后可能会导致无法新建连接。

所以建议给 Nginx Ingress 开启 TIME_WAIT 重用，即允许将 TIME_WAIT 连接重新用于新的 TCP 连接: `sysctl -w net.ipv4.tcp_tw_reuse=1`

### 调大最大文件句柄数

Nginx 作为反向代理，对于每个请求，它会与 client 和 upstream server 分别建立一个连接，即占据两个文件句柄，所以理论上来说 Nginx 能同时处理的连接数最多是系统最大文件句柄数限制的一半。

系统最大文件句柄数由 `fs.file-max` 这个内核参数来控制，TKE 默认值为 838860，建议调大: `sysctl -w fs.file-max=1048576`。

### 配置示例

给 Nginx Ingress Controller 的 Pod 添加 initContainers 来设置内核参数:

```yaml
      initContainers:
      - name: setsysctl
        image: busybox
        securityContext:
          privileged: true
        command:
        - sh
        - -c
        - |
          sysctl -w net.core.somaxconn=65535
          sysctl -w net.ipv4.ip_local_port_range="1024 65535"
          sysctl -w net.ipv4.tcp_tw_reuse=1
          sysctl -w fs.file-max=1048576
```

## 全局配置调优

除了内核参数需要调优，Nginx 本身的一些配置也需要进行调优，下面我们来详细看下。

### 调高 keepalive 连接最大请求数

Nginx 针对 client 和 upstream 的 keepalive 连接，均有 keepalive_requests 这个参数来控制单个 keepalive 连接的最大请求数，且默认值均为 100。当一个 keepalive 连接中请求次数超过这个值时，就会断开并重新建立连接。

如果是内网 Ingress，单个 client 的 QPS 可能较大，比如达到 10000 QPS，Nginx 就可能频繁断开跟 client 建立的 keepalive 连接，然后就会产生大量 TIME_WAIT 状态连接。我们应该尽量避免产生大量 TIME_WAIT 连接，所以，建议这种高并发场景应该增大 Nginx 与 client 的 keepalive 连接的最大请求数量，在 Nginx Ingress 的配置对应 `keep-alive-requests`，可以设置为 10000，参考: [https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#keep-alive-requests](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#keep-alive-requests)

同样的，Nginx 与 upstream 的 keepalive 连接的请求数量的配置是 `upstream-keepalive-requests`，参考: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-requests 但是，一般情况应该不必配此参数，如果将其调高，可能导致负载不均，因为 Nginx 与 upstream 保持的 keepalive 连接过久，导致连接发生调度的次数就少了，连接就过于 "固化"，使得流量的负载不均衡。

### 调高 keepalive 最大空闲连接数

Nginx 针对 upstream 有个叫 keepalive 的配置，它不是 keepalive 超时时间，也不是 keepalive 最大连接数，而是 keepalive 最大空闲连接数。

它的默认值为 32，在高并发下场景下会产生大量请求和连接，而现实世界中请求并不是完全均匀的，有些建立的连接可能会短暂空闲，而空闲连接数多了之后关闭空闲连接，就可能导致 Nginx 与 upstream 频繁断连和建连，引发 TIME_WAIT 飙升。在高并发场景下可以调到 1000，参考: [https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-connections](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-connections)

### 调高单个 worker 最大连接数

`max-worker-connections` 控制每个 worker 进程可以打开的最大连接数，TKE 环境默认 16384，在高并发环境建议调高，比如设置到 65536，这样可以让 nginx 拥有处理更多连接的能力，参考: [https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#max-worker-connections](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#max-worker-connections)

### 配置示例

Nginx 全局配置通过 configmap 配置(Nginx Ingress Controller 会 watch 并自动 reload 配置):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-controller
# nginx ingress 性能优化: https://www.nginx.com/blog/tuning-nginx/
data:
  # nginx 与 client 保持的一个长连接能处理的请求数量，默认 100，高并发场景建议调高。
  # 参考: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#keep-alive-requests
  keep-alive-requests: "10000"
  # nginx 与 upstream 保持长连接的最大空闲连接数 (不是最大连接数)，默认 32，在高并发下场景下调大，避免频繁建联导致 TIME_WAIT 飙升。
  # 参考: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-connections
  upstream-keepalive-connections: "200"
  # 每个 worker 进程可以打开的最大连接数，默认 16384。
  # 参考: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#max-worker-connections
  max-worker-connections: "65536"
```

## 总结

本文分享了对 Nginx Ingress 进行性能调优的方法及其原理的解释，包括内核参数与 Nginx 本身的配置调优，更好的适配高并发的业务场景，希望对大家有所帮助。

## 参考资料

* Nginx Ingress on TKE 部署最佳实践: https://mp.weixin.qq.com/s/NAwz4dlsPuJnqfWYBHkfGg
* Nginx Ingress 配置参考:  https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/
* Tuning NGINX for Performance: https://www.nginx.com/blog/tuning-nginx/
* ngx_http_upstream_module 官方文档: http://nginx.org/en/docs/http/ngx_http_upstream_module.html