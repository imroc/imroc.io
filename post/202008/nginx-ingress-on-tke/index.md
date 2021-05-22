---
title: "Nginx Ingress on TKE 部署最佳实践"

# Date published
date: 2020-08-07T11:00:00+08:00

# Date updated
lastmod: 2020-08-07T11:00:00+08:00

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

开源的 Ingress Controller 的实现使用量最大的莫过于 Nginx Ingress 了，功能强大且性能极高。Nginx Ingress 有多种部署方式，本文将介绍 Nginx Ingress 在 TKE 上的一些部署方案，这几种方案的原理、各自优缺点以及一些选型和使用上的建议。

## 什么是 Nginx Ingress ?

在介绍如何部署 Nginx Ingress 之前，我们先简单了解下什么是 Nginx Ingress。

Nginx Ingress 是 Kubernetes Ingress 的一种实现，它通过 watch Kubernetes 集群的 Ingress 资源，将 Ingress 规则转换成 Nginx 的配置，然后让 Nginx 来进行 7 层的流量转发:

![](https://imroc.cc/assets/blog/nginx-ingress-on-tke-1.jpg)

实际 Nginx Ingress 有两种实现：

1. [https://github.com/kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx)
1. [https://github.com/nginxinc/kubernetes-ingress](https://github.com/nginxinc/kubernetes-ingress)

第一种是 Kubernetes 开源社区的实现，第二种是 Nginx 官方的实现，我们通常用的是 Kubernetes 社区的实现，这也是本文所关注的重点。

## 有哪些部署方案 ?

那么如何在 TKE 上部署 Nginx Ingress 呢？主要有三种方案，下面分别介绍下这几种方案及其部署方法。

### 方案一： Deployment + LB

在 TKE 上部署 Nginx Ingress 最简单的方式就是将 Nginx Ingress Controller 以 Deployment 的方式部署，并且为其创建 LoadBalancer 类型的 Service(可以是自动创建 CLB 也可以是绑定已有 CLB)，这样就可以让 CLB 接收外部流量，然后转发到 Nginx Ingress 内部：

![](https://imroc.cc/assets/blog/nginx-ingress-on-tke-2.jpg)

当前 TKE 上 LoadBalancer 类型的 Service 默认实现是基于 NodePort，CLB 会绑定各节点的 NodePort 作为后端 rs，将流量转发到节点的 NodePort，然后节点上再通过 Iptables 或 IPVS 将请求路由到 Service 对应的后端 Pod，这里的 Pod 就是 Nginx Ingress Controller 的 Pod。后续如果有节点的增删，CLB 也会自动更新节点 NodePort 的绑定。

这是最简单的一种方式，可以直接通过下面命令安装:

```bash
kubectl create ns nginx-ingress
kubectl apply -f https://raw.githubusercontent.com/TencentCloudContainerTeam/manifest/master/nginx-ingress/nginx-ingress-deployment.yaml -n nginx-ingress
```

### 方案二：Daemonset + HostNetwork + LB

方案一虽然简单，但是流量会经过一层 NodePort，会多一层转发。这种方式有一些缺点：

1. 转发路径较长，流量到了 NodePort 还会再经过 Kubernetes 内部负载均衡，通过 Iptables 或 IPVS 转发到 Nginx，会增加一点网络耗时。
2. 经过 NodePort 必然发生 SNAT，如果流量过于集中容易导致源端口耗尽或者 conntrack 插入冲突导致丢包，引发部分流量异常。
3. 每个节点的 NodePort 也充当一个负载均衡器，CLB 如果绑定大量节点的 NodePort，负载均衡的状态就分散在每个节点上，容易导致全局负载不均。
4. CLB 会对 NodePort 进行健康探测，探测包最终会被转发到 Nginx Ingress 的 Pod，如果 CLB 绑定的节点多，Nginx Ingress 的 Pod 少，会导致探测包对 Nginx Ingress 造成较大的压力。

我们可以让 Nginx Ingress 使用 hostNetwork，CLB 直接绑节点 IP + 端口(80,443)， 这样就不用走 NodePort；由于使用 hostNetwork，Nginx Ingress 的 pod 就不能被调度到同一节点避免端口监听冲突。通常做法是提前规划好，选取部分节点作为边缘节点，专门用于部署 Nginx Ingress，为这些节点打上 label，然后 Nginx Ingress 以 DaemonSet 方式部署在这些节点上。下面是架构图：

![](https://imroc.cc/assets/blog/nginx-ingress-on-tke-3.jpg)

安装步骤：

1. 将规划好的用于部署 Nginx Ingress 的节点打上 label: `kubectl label node 10.0.0.3 nginx-ingress=true`(注意替换节点名称)。
2. 将 Nginx Ingress 部署在这些节点上:
```bash
kubectl create ns nginx-ingress
kubectl apply -f https://raw.githubusercontent.com/TencentCloudContainerTeam/manifest/master/nginx-ingress/nginx-ingress-daemonset-hostnetwork.yaml -n nginx-ingress
```
3. 手动创建 CLB，创建 80 和 443 端口的 TCP 监听器，分别绑定部署了 Nginx Ingress 的这些节点的 80 和 443 端口。

### 方案三：Deployment + LB 直通 Pod

方案二虽然相比方案一有一些优势，但同时也引入了手动维护 CLB 和 Nginx Ingress 节点的运维成本，需要提前规划好 Nginx Ingress 的节点，增删 Nginx Ingress 节点时需要手动在 CLB 控制台绑定和解绑节点，也无法支持自动扩缩容。<br />如果你的网络模式是 VPC-CNI，那么所有的 Pod 都使用的弹性网卡，弹性网卡的 Pod 是支持 CLB 直接绑 Pod 的，可以绕过 NodePort，并且不用手动管理 CLB，支持自动扩缩容:

![](https://imroc.cc/assets/blog/nginx-ingress-on-tke-4.jpg)

如果你的网络模式是 Global Router(大多集群都是这种模式)，你可以为集群开启 VPC-CNI 的支持，即两种网络模式混用，在集群信息页可打开：

![](https://imroc.cc/assets/blog/nginx-ingress-on-tke-5.jpg)

确保集群支持 VPC-CNI 之后，可以使用下面命令安装 Nginx Ingress:

```bash
kubectl create ns nginx-ingress
kubectl apply -f https://raw.githubusercontent.com/TencentCloudContainerTeam/manifest/master/nginx-ingress/nginx-ingress-deployment-eni.yaml -n nginx-ingress
```

## 如何选型？

前面介绍了 Nginx Ingress 在 TKE 上部署的三种方案，也说了各种方案的优缺点，这里做一个简单汇总下，给出一些选型建议：
1. 方案一较为简单通用，但在大规模和高并发场景可能有一点性能问题。如果对性能要求不那么严格，可以考虑使用这种方案。
2. 方案二使用 hostNetwork 性能好，但需要手动维护 CLB 和 Nginx Ingress 节点，也无法实现自动扩缩容，通常不太建议用这种方案。
3. 方案三性能好，而且不需要手动维护 CLB，是最理想的方案。它需要集群支持 VPC-CNI，如果你的集群本身用的 VPC-CNI 网络插件，或者用的 Global Router 网络插件并开启了 VPC-CNI 的支持(两种模式混用)，那么建议直接使用这种方案。

## 如何支持内网 Ingress ?

方案二由于是手动管理 CLB，自行创建 CLB 时可以选择用公网还是内网；方案一和方案三默认会创建公网 CLB，如果要用内网，可以改下部署 YAML，给 `nginx-ingress-controller` 这个 Service 加一个 key 为 `service.kubernetes.io/qcloud-loadbalancer-internal-subnetid`，value 为内网 CLB 所被创建的子网 id 的 annotation，示例:

``` yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxxxxx # value 替换为集群所在 vpc 的其中一个子网 id
  labels:
    app: nginx-ingress
    component: controller
  name: nginx-ingress-controller
```

## 如何复用已有 LB ?

方案一和方案三默认会自动创建新的 CLB，Ingress 的流量入口地址取决于新创建出来的 CLB 的 IP 地址。如果业务对入口地址有依赖，比如配置了 DNS 解析到之前的 CLB IP，不希望切换 IP；或者想使用包年包月的 CLB (默认创建是按量计费)，那么也可以让 Nginx Ingress 绑定已有的 CLB。

操作方法同样也是修改下部署 yaml，给 `nginx-ingress-controller` 这个 Service 加一个 key 为 `service.kubernetes.io/tke-existed-lbid`，value 为 CLB ID 的 annotation，示例:

``` yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/tke-existed-lbid: lb-6swtxxxx # value 替换为 CLB 的 ID
  labels:
    app: nginx-ingress
    component: controller
  name: nginx-ingress-controller
```

## Nginx Ingress 公网带宽有多大？

有同学可能会问：我的 Nginx Ingress 的公网带宽到底有多大？能否支撑住我服务的并发量？

这里需要普及一下，腾讯云账号有带宽上移和非带宽上移两种类型：
1. 非带宽上移，是指带宽在云主机(CVM)上管理。
2. 带宽上移，是指带宽上移到了 CLB 或 IP 上管理。

具体来讲，如果你的账号是非带宽上移类型，Nginx Ingress 使用公网 CLB，那么 Nginx Ingress 的公网带宽是 CLB 所绑定的 TKE 节点的带宽之和；如果使用方案三，CLB 直通 Pod，也就是 CLB 不是直接绑的 TKE 节点，而是弹性网卡，那么此时 Nginx Ingress 的公网带宽是所有 Nginx Ingress Controller Pod 被调度到的节点上的带宽之和。

如果你的账号是带宽上移类型就简单了，Nginx Ingress 的带宽就等于你所购买的 CLB 的带宽，默认是 10Mbps (按量计费)，你可以按需调整下。

由于历史遗留原因，以前注册的账号大多是非带宽上移类型，参考 [这里](https://cloud.tencent.com/document/product/684/39903) 来区分自己账号的类型。

## 如何创建 Ingress ?

目前还没有完成对 Nginx Ingress 的产品化支持，所以如果是在 TKE 上自行了部署 Nginx Ingress，想要使用 Nginx Ingress 来管理 Ingress，目前是无法通过在 TKE 控制台(网页) 上进行操作的，只有通过 YAML 的方式来创建，并且需要给每个 Ingress 都指定 Ingress Class 的 annotation，示例:

``` yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/ingress.class: nginx # 这里是重点
spec:
  rules:
  - host: *
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-v1
          servicePort: 80
```

## 如何监控？

通过上面的方法安装的 Nginx Ingress，已经暴露了 metrics 端口，可以被 Prometheus 采集。如果集群内安装了 prometheus-operator，可以使用下面的 ServiceMonitor 来采集 Nginx Ingress 的监控数据:

``` yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-ingress-controller
  namespace: nginx-ingress
  labels:
    app: nginx-ingress
    component: controller
spec:
  endpoints:
  - port: metrics
    interval: 10s
  namespaceSelector:
    matchNames:
    - nginx-ingress
  selector:
    matchLabels:
      app: nginx-ingress
      component: controller
```

这里也给个原生 Prometheus 配置的示例:

``` yaml
    - job_name: nginx-ingress
      scrape_interval: 5s
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - nginx-ingress
      relabel_configs:
      - action: keep
        source_labels:
        - __meta_kubernetes_service_label_app
        - __meta_kubernetes_service_label_component
        regex: nginx-ingress;controller
      - action: keep
        source_labels:
        - __meta_kubernetes_endpoint_port_name
        regex: metrics
```

有了数据后，我们再给 grafana 配置一下面板来展示数据，Nginx Ingress 社区提供了面板： https://github.com/kubernetes/ingress-nginx/tree/master/deploy/grafana/dashboards


我们直接复制 json 导入到 grafana 即可导入面板。其中，`nginx.json` 是展示 Nginx Ingress 各种常规监控的面板：

![](https://imroc.cc/assets/blog/nginx-ingress-on-tke-6.jpg)

`request-handling-performance.json` 是展示 Nginx Ingress 性能方面的监控面板：

![](https://imroc.cc/assets/blog/nginx-ingress-on-tke-7.jpg)

## 总结

本文梳理了 Nginx Ingress 在 TKE 上部署的三种方案以及许多实用的建议，对于想要在 TKE 上使用 Nginx Ingress 的同学是一个很好的参考。由于 Nginx Ingress 的使用需求量较大，我们也正在做 Nginx Ingress 的产品化支持， 可以实现一键部署，集成日志和监控能力，并且会对其进行性能优化。相信在不久的将来，我们就能够在 TKE 上更简单高效的使用 Nginx Ingress 了，敬请期待吧！

## 参考资料

1. TKE Service YAML 示例: https://cloud.tencent.com/document/product/457/45489#yaml-.E7.A4.BA.E4.BE.8B
2. TKE Service 使用已有 CLB: https://cloud.tencent.com/document/product/457/45491
3. 区分腾讯云账户类型: https://cloud.tencent.com/document/product/684/39903