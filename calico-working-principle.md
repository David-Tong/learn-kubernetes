# Calico 的工作原理

Calico 是 Kubernetes 常用的容器网络与网络安全方案。它不只是一个 CNI 插件：在常见部署中，Calico 还负责 Pod IP 地址管理、节点间 Pod 流量转发，以及 NetworkPolicy 的执行。具体组件和数据转发方式取决于安装配置；不能只凭“集群安装了 Calico”就推断一定启用了某种 IPAM、BGP、隧道或策略功能。

## 先看整体

```text
Kubernetes API 中的 Pod、IPPool、NetworkPolicy 等期望状态
                         │
             Calico 控制组件和节点代理观察状态
                         │
       ┌─────────────────┴─────────────────┐
       │                                   │
   Pod 网络配置                       路由与访问策略配置
   CNI + IPAM                          Felix 等节点组件
       │                                   │
       └───────────────┬───────────────────┘
                       ▼
             节点上的 Linux 网络数据平面
                       │
        同节点直达，或路由 / 隧道到其他节点
```

可以把 Calico 分成两类工作：

- **控制面和节点代理**：观察集群期望状态，并将网络与策略配置分发、落实到节点。
- **数据面**：真正处理数据包的节点网络接口、路由和策略规则。通常数据包由节点内核转发；使用 eBPF 数据面时，转发和服务处理方式会不同。

## Pod 创建时，Calico 如何配置网络？

以普通 Linux Pod 和常见的 veth 网络为例：

1. Kubernetes 将 Pod 调度到某个节点，容器运行时为 Pod 准备网络命名空间。
2. 容器运行时调用该节点配置好的 CNI 插件，为 Pod 添加网络。
3. Calico CNI 调用配置的 IPAM 为 Pod 分配地址。若使用 `calico-ipam`，通常从适用的 Calico `IPPool` 分配；集群也可能配置其他 IPAM。
4. CNI 配置 Pod 网络接口、节点侧连接和必要的路由，然后向运行时返回网络配置。
5. Pod 删除时，运行时再次调用 CNI 清理网络，并释放相应地址和资源。

因此，**CNI 主要参与 Pod 网络的创建和清理，不是每个数据包经过时才调用一次**。Pod 网络建好后，数据包由节点数据面转发。

## Pod IP 如何在节点间可达？

IPAM 负责“给 Pod 分配哪个 IP”，路由机制负责“如何把发往这个 IP 的包送到目标节点”。二者相关，但不是同一件事。

```text
源 Pod
  │
  ▼
源节点上的 Pod 接口和路由
  │
  ├─ 目标 Pod 在本节点：经本机网络直接到达
  │
  └─ 目标 Pod 在其他节点：经节点路由或隧道到达目标节点
                                  │
                                  ▼
                         目标节点转发给目标 Pod
```

Calico 可根据集群网络和配置采用不同的节点间传输方式，例如：

- **BGP 路由**：节点之间交换到 Pod 网段或路由块的可达信息，使网络知道如何到达目标节点。是否运行 BGP、由谁传播路由，取决于部署模式。
- **Overlay 隧道**：使用 VXLAN 或 IP-in-IP 等方式，将 Pod 流量封装在节点间网络可传输的报文中，再由目标节点解封装。隧道会增加报文头，需要考虑 MTU。
- **底层网络集成或其他路由方式**：由云平台、数据中心网络或集群的其他路由配置提供可达性。

这些方式可能有不同前提，也可能组合使用。实际流量路径应以集群的 IPPool、节点和 Calico 配置为准。

## NetworkPolicy 如何生效？

NetworkPolicy 描述的是**哪些端点之间允许哪些流量**，本身并不转发数据包。Calico 的节点组件观察 Kubernetes API 中的策略和 Pod、Namespace 标签，再将允许或拒绝规则落实到节点数据面。

例如，策略可以选择某些目标 Pod，并按来源 Pod、来源命名空间、方向、协议和端口定义规则。数据包经过相关节点时，数据面依据这些规则判断是否放行。Calico 既可以执行 Kubernetes 标准 `NetworkPolicy`，也可以在相应配置下支持 Calico 自身的扩展策略。

需要注意：

- NetworkPolicy 需要网络方案实际执行；仅创建策略对象不代表流量一定受到控制。
- Kubernetes NetworkPolicy 的允许规则是累加的。一条策略不会撤销其他策略已经允许的流量。
- 被入站策略选中的 Pod 会受到入站隔离；没有匹配到任何适用允许规则的入站流量会被拒绝。出站策略同理，但需要单独配置。
- 同一条规则中的命名空间选择器和 Pod 选择器通常需要同时匹配；选择器的具体组合方式会影响策略范围。

配置和实验示例见 [Calico 网络策略实验](./learn-calico/README.md)。

## 常见组件各自做什么？

| 组件或资源 | 主要职责 |
| --- | --- |
| Calico CNI 插件 | 响应容器运行时的网络配置请求，为 Pod 接入网络 |
| IPAM / `IPPool` | 分配和管理 Pod IP 地址及地址池 |
| Felix | 节点代理，观察网络与策略状态，并配置节点上的接口、路由及策略数据面 |
| BGP 组件 | 在使用 BGP 的部署中传播路由信息；不启用 BGP 的部署不依赖它完成路由传播 |
| `calico-kube-controllers` | 运行 Calico 的 Kubernetes 控制器；负责的资源同步和控制逻辑取决于部署配置 |
| Typha | 可选的状态分发组件，常用于减少大量节点直接访问数据存储时的负载 |

组件名称、是否部署和职责细节可能随 Calico 版本、安装方式和数据面模式而异。

## Calico 和 Service 有什么关系？

Pod IP 网络和 Service 虚拟 IP 是不同层面的功能。访问 ClusterIP Service 时，通常由 `kube-proxy` 或集群配置的其他 Service 数据面选择后端并转发；某些 Calico eBPF 部署也可以接管 Service 数据面。随后，Pod 网络负责把流量送到后端 Pod。

所以，排查 Service 访问问题时，除了 Calico 网络和策略，也要检查 Service、EndpointSlice 以及集群实际使用的 Service 数据面。可参考 [Pod IP 和 Service IP](./pod-ip-and-service-ip.md)。

## 如何初步检查？

以下命令只读取集群状态；某些资源可能因安装方式不同或权限不足而无法查询：

```sh
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get ippools
kubectl get networkpolicy -A
```

- 节点和 Pod 状态可帮助确认 Pod 是否已调度、是否获得 IP。
- `IPPool` 可查看地址范围及相关配置；如果资源类型不存在，可能是集群未使用 Calico IPAM，或当前环境不提供该资源。
- NetworkPolicy 清单可帮助确认目标命名空间中的策略，但策略对象存在不等于已确认数据面行为。
- Calico 组件的名称和命名空间依安装方式而异；如有权限，可先用 `kubectl get pods -A` 查找相关组件，再查看其状态和日志。

不要在不了解现有配置的情况下修改 CNI 配置、IPPool、路由或策略；这类变更可能影响整个集群的 Pod 连通性。

## 延伸阅读

- [Calico 官方文档](https://docs.tigera.io/calico/latest/)
- [Calico 网络文档](https://docs.tigera.io/calico/latest/networking/)
- [Calico IP 地址管理](https://docs.tigera.io/calico/latest/networking/ipam/)
- [Calico NetworkPolicy 文档](https://docs.tigera.io/calico/latest/network-policy/)
- [Kubernetes 网络插件文档](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
