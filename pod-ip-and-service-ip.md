# Pod IP 和 Service IP

在 Kubernetes 集群中，每个普通 Pod 都有用于通信的 **Pod IP**；Service 则可以提供一个相对稳定的 **ClusterIP**，让客户端不必追踪后端 Pod 的变化。

本文以 Calico IPAM 和 ClusterIP Service 为例，介绍地址如何分配、流量如何转发，以及如何查看实际配置。

## 1. 先看区别

| | Pod IP | Service ClusterIP |
| --- | --- | --- |
| 作用 | 标识具体的 Pod 网络端点 | 为一组后端提供集群内的访问入口 |
| 地址来源 | 由集群配置的 IPAM 分配；Calico IPAM 使用 Calico IPPool | 由 Kubernetes 控制面从 Service CIDR 分配 |
| 地址变化 | Pod 被替换后可能变化 | Service 存续期间通常不变；删除重建后可能变化 |
| 通信方式 | 网络将流量路由到目标 Pod | Service 数据平面选择后端并转发流量 |

例如，集群可能有：

```text
Pod 地址池：     10.0.0.0/16
Pod IP：         10.0.1.5、10.0.2.6

Service CIDR：  10.96.0.0/12
Service IP：    10.96.8.20
```

这些只是示例，实际网段以集群配置为准。Pod 和 Service 地址范围应避免与彼此、节点网段以及需要互通的外部网络重叠。

**为什么 Service IP 不在 Calico IPPool 中？** 因为 ClusterIP 使用 Kubernetes 的 Service CIDR，而普通 Pod 使用网络插件的 IPAM 地址范围；它们是不同的分配机制。

## 2. Pod IP 如何分配？

当集群使用 Calico IPAM 时，容器运行时在创建 Pod 网络时调用 CNI 插件，Calico IPAM 从适用的 **IPPool** 中分配地址。Calico 将地址池划分为较小的 block，并与节点关联，以便分配地址和汇总路由。Pod 删除或被替换后，地址可能释放并重新分配，不应依赖它长期不变。

Calico 也可以与其他 IPAM 配合使用，例如 `host-local`。因此，安装了 Calico 并不意味着所有 Pod IP 一定来自 Calico IPPool；还需要看集群实际的 CNI 和 IPAM 配置。

### Pod IP 在哪里？

对于使用普通网络的 Linux Pod，IP 通常配置在 Pod 自己的网络命名空间中的接口上，而不是节点物理网卡上。以 Calico 常见的 veth 模式为例：

```text
节点网络命名空间                  Pod 网络命名空间
┌─────────────────────┐         ┌─────────────────────┐
│ cali...             │── veth ─│ eth0: 10.0.1.5      │
│ 节点路由和策略       │         │ Pod 内的容器        │
└─────────────────────┘         └─────────────────────┘
```

veth 是一对相连的虚拟接口，接口名和实现因网络插件及配置而异。Pod 内的容器通常共享网络命名空间和 IP。因此，在节点上直接运行 `ip addr` 看不到 Pod 接口，并不表示 Pod IP 不存在。使用 `hostNetwork: true` 的 Pod 共享节点网络命名空间，是一个例外。

### 如何查看 Pod IP 和 IPPool？

查看 Pod 地址和所在节点：

```sh
kubectl get pods -A -o wide
```

输出的 `IP` 是 Pod IP，`NODE` 是 Pod 所在节点。

集群安装了 Calico IPPool 资源且当前用户有读取权限时，查看地址池及其配置：

```sh
kubectl get ippools
kubectl get ippools -o yaml
```

重点查看 `spec.cidr`（地址范围）、`spec.blockSize`（block 大小）、`spec.nodeSelector`（适用节点）和 `spec.disabled`（是否禁用）。字段可能因配置而省略。使用 Calico IPAM 时，可将普通 Pod 的 IP 与适用的 IPPool CIDR 对照；多个池时还需检查池的选择条件。

若容器镜像中包含 `ip` 命令，可以进入 Pod 的网络命名空间查看接口。执行前请将占位符替换为实际名称：

```sh
kubectl -n <命名空间> exec <Pod名> -- ip addr
```

如果提示找不到 `ip` 命令，可能只是镜像未包含该工具，并不表示 Pod 没有 IP。更多 IPAM 配置和实践步骤见 [Calico 实验](./learn-calico/README.md)。

## 3. Service IP 如何分配？

Kubernetes 控制面从 **Service CIDR** 中为使用 ClusterIP 的 Service 分配虚拟 IP。Service 为客户端提供稳定入口，后端 Pod 更新时，客户端通常仍使用同一个 Service 名称或 ClusterIP。

ClusterIP 不代表一个独立的 Pod 网卡。具体实现取决于集群：例如 kube-proxy 的 iptables 或 IPVS 模式，或者其他组件实现的 eBPF 数据平面。某些实现会将 ClusterIP 配置到节点上的虚拟接口，因此不能一概认为 Service IP 永远不会出现在网卡上。

NodePort 和 LoadBalancer 类型的 Service 通常也有 ClusterIP，同时提供额外的访问方式。Headless Service 的 `clusterIP` 为 `None`，不会分配 ClusterIP；`ExternalName` Service 使用 DNS 名称映射，也不分配 ClusterIP。

### 如何查看 Service IP 和 Service CIDR？

查看各 Service 当前的 ClusterIP：

```sh
kubectl get services -A -o wide
```

查看输出中的 `CLUSTER-IP` 列。在支持 ServiceCIDR API 且有读取权限的集群中，还可以查看分配范围：

```sh
kubectl get servicecidrs
kubectl get servicecidrs -o yaml
```

查看输出中的 `CIDRs` 列或 YAML 中的 `spec.cidrs`。集群可能有多个 ServiceCIDR，也可能同时配置 IPv4 和 IPv6。如果提示资源类型不存在，可能是集群版本或配置不支持该 API；请查看集群发行版或云平台提供的网络配置。不能仅凭已经分配的 Service IP 推断整个 CIDR。

## 4. 访问 Service 时流量怎么走？

```text
客户端 Pod
    │ 访问 Service ClusterIP:端口
    ▼
Service 数据平面选择一个可用后端
    │
    ▼
目标 Pod IP:端口
```

Kubernetes 使用 EndpointSlice 等资源记录 Service 后端地址和端口。Service 数据平面根据这些信息选择后端，并将流量导向目标 Pod。后端发生变化时，EndpointSlice 会随之更新。

跨节点流量还需要集群网络将数据送到目标 Pod。Calico 可以使用路由、VXLAN 或 IP-in-IP 等方式，具体路径取决于集群配置。

查看 Service 和它的后端，执行前请替换占位符：

```sh
kubectl -n <命名空间> describe service <服务名>
kubectl -n <命名空间> get endpointslices -l kubernetes.io/service-name=<服务名> -o wide
```

第一条命令显示 Service IP、端口和后端；第二条命令显示对应的 EndpointSlice 和后端地址。

## 5. 查看控制面组件的网络参数

如果控制面组件作为 Pod 运行且有查看权限，可以从其启动参数了解集群配置。先查看组件名称：

```sh
kubectl -n kube-system get pods
```

找到 `kube-controller-manager` 的 Pod 名称并替换占位符：

```sh
kubectl -n kube-system describe pod <controller-manager的Pod名>
```

输出的 `Command` 部分可能包含以下参数（地址仅为示例）：

```text
--allocate-node-cidrs=true
--cluster-cidr=10.0.0.0/16
--service-cluster-ip-range=10.96.0.0/12
```

- `--cluster-cidr`：控制器管理器配置的集群 Pod 网络范围。
- `--allocate-node-cidrs=true`：控制器管理器为节点分配 Pod CIDR。可通过 `kubectl get nodes -o yaml` 查看节点的 `spec.podCIDR`；Calico IPAM 分配 Pod IP 时不依赖该字段。
- `--service-cluster-ip-range`：控制器管理器配置的 Service 地址范围，不是 Calico IPPool。

也可以查看 API server 的启动参数，查找其 `--service-cluster-ip-range`：

```sh
kubectl -n kube-system describe pod <apiserver的Pod名>
```

将启动参数与 `kubectl get servicecidrs`、`kubectl get ippools` 的结果对照。ServiceCIDR 资源可能有多个，启动参数不一定列出所有后来添加的范围。控制面组件的名称和部署方式因集群而异；托管集群可能不暴露这些 Pod 或启动配置，请以发行版或云平台文档为准。

## 继续阅读

- [Calico IP 分配和实验](./learn-calico/README.md)
- [Kubernetes Service 文档](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes EndpointSlice 文档](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
