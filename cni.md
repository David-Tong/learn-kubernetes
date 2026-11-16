# Kubernetes CNI 概览

## 什么是 CNI？

CNI（Container Network Interface）是一套用于配置容器网络的规范和插件机制。容器运行时通过 CNI 调用网络插件，为容器或 Pod 的网络环境添加、检查或移除网络连接。

在 Kubernetes 中，CNI 通常负责将普通 Pod 接入集群网络。它定义的是运行时与插件之间的调用约定，并不规定所有插件必须使用同一种路由、封装或策略实现。

## Kubernetes 需要什么样的 Pod 网络？

Kubernetes 的网络模型期望每个普通 Pod 都能获得集群内可用的 IP，Pod 之间可以通信，而不需要应用自己配置网络地址转换。Pod 内的容器通常共享网络命名空间、IP 和端口空间。

**网络连通性不等于访问权限。** 即使网络能够把流量送到目标 Pod，NetworkPolicy 或其他安全机制仍可限制通信。使用 `hostNetwork: true` 的 Pod 共享节点网络命名空间，是普通 Pod 网络的例外。

## Pod 网络是如何创建的？

一个常见的 Linux Pod 创建过程如下：

1. Kubernetes 将 Pod 调度到节点，节点上的容器运行时准备 Pod 的网络命名空间。
2. 容器运行时根据节点的 CNI 配置调用插件，发出添加网络的请求。
3. 插件创建或配置网络接口，并通过 IPAM 插件等机制分配地址；在常见的 veth 方案中，一端位于 Pod 网络命名空间，另一端位于节点网络命名空间。
4. 插件完成必要的地址、路由等配置并返回结果，Pod 中的容器便可使用网络。
5. Pod 网络被移除时，运行时调用插件执行清理，包括释放相关网络资源和地址。

CNI 规范中常见的操作包括 `ADD`、`DEL` 和 `CHECK`；运行时和插件对具体操作的支持取决于其版本与配置。CNI 配置也可以串联多个插件，由它们分别完成网络接入、端口映射等工作。

## CNI 与集群网络组件的关系

| 组件或概念 | 主要职责 |
| --- | --- |
| 容器运行时 | 在节点上调用 CNI 插件配置和清理 Pod 网络 |
| CNI 插件 | 按 CNI 约定配置网络接口、地址和路由等 |
| IPAM | 管理和分配 IP 地址；例如 Calico IPAM 使用 IPPool |
| 网络数据平面 | 通过路由、封装等机制转发 Pod 流量，具体实现由网络方案决定 |
| NetworkPolicy | Kubernetes 定义的网络访问控制 API，需要网络方案支持和执行 |
| Service 数据平面 | 将 Service 流量导向后端，例如 kube-proxy 或其他实现 |
| CoreDNS | 为 Service 等资源提供集群 DNS 解析 |

同一个网络方案可能同时提供 CNI、IPAM、策略执行和 Service 数据平面功能，也可能与其他组件配合。**CNI 本身不等于 NetworkPolicy、Service 或 DNS。** Pod IP 与 Service ClusterIP 的区别可参考 [Pod IP 和 Service IP](./pod-ip-and-service-ip.md)。

## 跨节点通信

同一节点上的 Pod 通信通常经由节点上的虚拟网络设备；跨节点时，还需要把数据送到目标节点。不同网络方案可以采用不同方式：

- **路由**：节点或底层网络维护到 Pod 地址的路由，必要时可通过 BGP 等协议传播路由。
- **Overlay 封装**：例如 VXLAN 或 IP-in-IP，将 Pod 流量封装后通过节点网络传输，在目标端解封装。
- **云平台网络集成**：部分方案利用云平台网络能力为 Pod 提供地址和连接。

这些方式各有部署条件和运维取舍。例如，封装需要考虑额外报文头对 MTU 的影响；直接路由则需要底层网络能够处理相应路由。实际实现可能组合使用多种方式，不应根据插件名称直接假定流量路径。

同节点通信为什么仍需要 CNI，以及如何验证跨节点通信，可参考[同节点与跨节点 Pod 通信](./learn-calico/ref/calico-same-and-cross-node.md)。

## 常见网络方案

| 方案 | 简介 |
| --- | --- |
| Calico | 提供容器网络、IPAM 和网络策略等功能，支持多种路由和封装方式 |
| Cilium | 使用 eBPF 提供网络、安全和可观测性等能力 |
| Flannel | 主要提供简单的 Pod 网络连接；如需 NetworkPolicy，通常需要配合支持策略的组件 |
| 云平台 CNI | 将 Pod 网络与特定云平台的网络资源集成，具体能力取决于提供商和配置 |

选择时应结合 Kubernetes 发行版的支持情况、底层网络、NetworkPolicy 需求、性能和运维复杂度，而不是仅依据某一项功能。部署前请查看所选方案的版本兼容性与安装文档。

## 如何查看和排查？

先确认集群和 Pod 的基本状态：

```sh
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get pods -A
```

查看节点是否就绪、Pod 是否获得 IP，以及网络组件的 Pod 是否正常运行。组件名称和命名空间取决于安装方式。

如果某个 Pod 无法创建或网络异常，可以查看它的事件：

```sh
kubectl -n <命名空间> describe pod <Pod名>
```

执行前将占位符替换为实际名称。事件可能包含 CNI 调用、IP 分配或网络沙箱创建失败的线索。Pod 已运行但连接失败时，还应分别检查目标 Pod、Service 和 EndpointSlice，以及相关 NetworkPolicy；不应直接断定是 CNI 问题。

如果有权限登录节点，可以只读查看该节点上的 CNI 配置：

```sh
sudo ls /etc/cni/net.d
```

配置目录和文件名可能因发行版而异。查看实际配置文件时注意其中可能包含凭据，不要直接分享敏感内容，也不要在已有集群上随意修改或替换 CNI 配置。

## 继续阅读

- [Calico 介绍和动手实验](./learn-calico/README.md)
- [Pod IP 和 Service IP](./pod-ip-and-service-ip.md)
- [Kubernetes 网络插件文档](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [CNI 规范](https://www.cni.dev/docs/spec/)
