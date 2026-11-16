# Calico 网络策略实验

## Calico 是什么，有什么作用？

[Calico](https://www.tigera.io/project-calico/) 是一个为 Kubernetes 提供网络和网络安全功能的开源项目，由 Tigera 开发维护，广泛应用于生产环境中。

**Calico 的核心作用：**

1. **解决 Pod 间的网络连接问题**：在 Kubernetes 集群中为每个 Pod 分配独立的 IP 地址，实现 Pod 之间可以相互通信
2. **实现网络隔离与访问控制**：通过 NetworkPolicy 定义哪些 Pod 可以访问哪些 Pod，防止未授权访问
3. **提升集群安全性**：限制恶意 Pod 对其他服务的访问，实现微观分段（Microsegmentation）防御

简单来说，**Calico 就是 Kubernetes 的"网络保安"**——它既确保 Pod 能够正常通信，又能在需要时严格控制谁能访问谁。

## Calico 的核心特性

- **高性能网络连接**：使用 BGP 路由和 Linux 原生转发，提供低延迟、高吞吐量的容器网络
- **灵活的网络策略**：支持 Kubernetes NetworkPolicy 和 Calico 扩展策略，精细控制 Pod 之间的通信
- **安全隔离**：通过 NetworkPolicy 实现微观分段（Microsegmentation），防止横向移动攻击
- **可视化与观察性**：集成网络流量监控和策略管理工具，便于故障排查
- **多种工作负载支持**：不仅支持 Kubernetes Pod，还支持虚拟机、容器和物理机的混合部署

## Calico 如何为 Pod 分配 IP

在常见的 Calico 安装中，Calico CNI 插件负责接入 Pod 网络，`calico-ipam` 负责从配置的地址池中分配 Pod IP。也有集群使用 `host-local` 等其他 IPAM 插件，因此实际分配方式取决于集群安装配置。

大致流程如下：

1. Kubernetes 创建 Pod 后，容器运行时调用 CNI 插件配置 Pod 网络。
2. Calico IPAM 从可用的 **IPPool** 中选取一个地址分配给 Pod，并记录该地址的使用情况。
3. Calico 配置 Pod 的网络接口和路由；随后由集群的路由机制负责让其他节点能够到达这个 Pod IP。

Calico IPAM 通常会把 IPPool 划分为较小的 **IPAM block**，并把 block 关联到节点，以便高效分配和汇总路由。默认 block 可容纳 64 个地址（IPv4 `/26`），但可以按 IPPool 配置调整。IPPool 还可以通过节点选择器等设置，控制哪些节点或工作负载使用某个地址范围。

**IP 分配和跨节点路由是相关但不同的事情。** Calico 可通过 BGP 等方式传播 Pod 路由；如果底层网络不能直接路由 Pod 网段，也可以配置 VXLAN 或 IP-in-IP 封装。具体使用哪种模式由集群网络和 Calico 配置决定。

| 地址类型 | 通常由谁管理 | 用途 |
| --- | --- | --- |
| Pod IP | Calico IPAM（或集群配置的其他 IPAM） | Pod 网络接口的地址；Pod 重建后地址可能改变 |
| Service ClusterIP | Kubernetes 控制面 | Service 的稳定虚拟 IP；不是普通 Pod IP 池分配的地址 |
| Node IP | 节点操作系统、云平台及 Kubernetes 节点配置 | 节点自身的地址；Calico 可使用它进行节点间路由或隧道通信 |

集群规划 Pod 网段时，应确保它与节点网段、Service CIDR 及需要互通的外部网络不冲突。日常访问应用应使用 Service DNS，而不要依赖某个 Pod IP 长期不变。

## Calico 和 Istio 的关系与重叠

Calico 和 [Istio](https://istio.io/latest/) 都能参与服务间流量的安全控制，但定位不同，通常是互补使用，而不是二选一：

| 对比 | Calico | Istio |
| --- | --- | --- |
| 主要定位 | Kubernetes 网络插件，提供 Pod 网络连接、IP 地址管理和网络策略 | 服务网格，管理网格内服务之间的流量与身份 |
| 常见策略层级 | 网络层（L3/L4）：Pod/Namespace 标签、IP、协议和端口 | 服务与请求层（主要是 L7）：服务身份、HTTP 方法、路径等；也支持部分 L4 规则 |
| 典型能力 | 即使工作负载未加入服务网格，也可控制 Pod 入站和出站网络流量 | 提供服务间 mTLS、基于身份的授权、流量路由和遥测 |

**重叠点**是两者都可以允许或拒绝服务间通信，尤其在限制端口或工作负载之间的访问时，看起来会有相似效果。区别在于 Calico 通常从 Pod 网络和标签出发控制“哪些网络端点可以互通”；Istio 通常从服务身份和请求属性出发控制“哪些服务或请求可以调用接口”。

**配合方式**通常是用 Calico 建立集群网络层的基础隔离，再用 Istio 对已纳入网格的服务实施更细的身份和应用层访问控制。一次请求要同时满足沿途的网络策略和 Istio 策略；如果 Calico 在网络层已经拒绝，Istio 的允许规则不能使它通过。反过来，网络层允许也不代表 Istio 一定允许该请求。

Istio 的 CNI 插件用于配置服务网格的流量拦截，不等同于替代 Calico 的通用集群网络插件。若同时使用两者，需要根据 Istio 部署模式和版本确认 Calico 策略不会阻断 Istio 控制面、网格流量或健康检查所需的连接。

## 适用场景

- Kubernetes 集群的网络基础架构建设
- 微服务安全隔离与访问控制
- 多租户环境的网络隔离
- 合规性要求的网络策略实现（如零信任架构）

## 相关资源

- [Calico 官方文档](https://docs.tigera.io/calico/latest/)
- [Calico IP 地址管理（IPAM）](https://docs.tigera.io/calico/latest/networking/ipam/)
- [Calico Overlay 网络与封装](https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip/)
- [Kubernetes NetworkPolicy 文档](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Calico 安装指南](https://docs.tigera.io/calico/latest/getting-started/kubernetes/)
- [Istio 安全与授权文档](https://istio.io/latest/docs/concepts/security/)
- [Istio CNI 文档](https://istio.io/latest/docs/setup/additional-setup/cni/)

## 动手实验

本实验依次完成以下四个 Lab，后面的步骤会复用前面创建的资源和策略：

| 实验 | 目标 |
| --- | --- |
| Lab 1 | 查看 Pod IP、IPAM 配置并验证网络连通 |
| Lab 2 | 使用命名空间和 Pod 标签控制 Web 访问 |
| Lab 3 | 为应用命名空间添加默认拒绝入站策略 |
| Lab 4 | 在默认拒绝基础上只开放 API 到数据库层的访问 |

> **请只在专用实验集群中操作。** NetworkPolicy 可能中断业务流量。实验只创建自己的 Namespace 和策略，不要在已有集群上替换 CNI。集群需要已安装并正常运行 Calico，且启用了 NetworkPolicy；安装前请按集群发行版查看[官方安装说明](https://docs.tigera.io/calico/latest/getting-started/kubernetes/)和版本兼容性。

### 准备环境

需要有权限创建 Namespace、Pod、Deployment、Service 和 NetworkPolicy 的 `kubectl`。查看 IPPool 和 Operator 配置还需要相应的集群资源读取权限。实验中的 DNS 名称使用 `cluster.local`；如果集群使用其他 DNS 域名，请相应替换。

确认当前集群和节点状态：

```sh
kubectl config current-context
kubectl get nodes
kubectl get pods -A
```

在输出中确认 `calico-node`、`calico-kube-controllers` 等 Calico 组件处于 `Running` 状态。组件名称和所在命名空间会随安装方式有所不同。

确认实验命名空间不存在；如果已存在，请先确认其中资源和策略，不要直接复用或删除不属于本实验的资源。创建三个命名空间并标记受信任的客户端命名空间：

```sh
kubectl create namespace calico-lab-app
kubectl create namespace calico-lab-trusted
kubectl create namespace calico-lab-untrusted
kubectl label namespace calico-lab-trusted lab-access=trusted
```

创建 Web 服务：

```sh
kubectl -n calico-lab-app create deployment web --image=nginx
kubectl -n calico-lab-app expose deployment web --port=80
kubectl -n calico-lab-app rollout status deployment/web --timeout=120s
```

`kubectl create deployment web` 默认会为 Deployment 的 Pod 选择器和 Pod 模板设置 `app=web` 标签。因此，下面 NetworkPolicy 中的 `podSelector` 能选中 Web Pod。这个标签是该命令生成资源时的默认值，并非所有 Pod 都会自动拥有；可以用 `kubectl -n calico-lab-app get pods --show-labels` 查看实际标签。

创建三个客户端 Pod，用于后续向 Web 服务发送请求：

```sh
kubectl -n calico-lab-trusted run frontend --image=curlimages/curl --restart=Never --labels=role=frontend --command -- sleep 3600
```

在 `calico-lab-trusted` 命名空间创建名为 `frontend`、标签为 `role=frontend` 的 Pod，模拟允许访问 Web 的前端客户端。

- `-n calico-lab-trusted`：指定 Pod 所在的命名空间。
- `run frontend`：创建名为 `frontend` 的 Pod。
- `--image=curlimages/curl`：使用带有 `curl` 的镜像，供后续测试连接。
- `--restart=Never`：创建独立 Pod，不自动重启容器。
- `--labels=role=frontend`：设置 Pod 标签，供后续网络策略选择。
- `--command -- sleep 3600`：让容器运行约一小时，以便后续通过 `kubectl exec` 执行 `curl`。容器退出后需要重新创建 Pod。

```sh
kubectl -n calico-lab-trusted run other --image=curlimages/curl --restart=Never --labels=role=other --command -- sleep 3600
```

在同一命名空间创建名为 `other`、标签为 `role=other` 的 Pod，模拟来自受信任命名空间但不具备前端标签的客户端。其余参数与上一条命令相同。

```sh
kubectl -n calico-lab-untrusted run attacker --image=curlimages/curl --restart=Never --labels=role=attacker --command -- sleep 3600
```

在 `calico-lab-untrusted` 命名空间创建名为 `attacker`、标签为 `role=attacker` 的 Pod，模拟不应获准访问的客户端；它只是普通测试 Pod，不会执行攻击。其余参数与第一条命令相同。

```sh
kubectl -n calico-lab-trusted wait --for=condition=Ready pod --all --timeout=120s
```

等待 `calico-lab-trusted` 中的所有 Pod 达到 `Ready` 状态，最多等待 120 秒。

```sh
kubectl -n calico-lab-untrusted wait --for=condition=Ready pod --all --timeout=120s
```

同样等待 `calico-lab-untrusted` 中的所有 Pod 就绪，最多等待 120 秒。

这里的“受信任”只表示实验中给 `calico-lab-trusted` 命名空间设置了 `lab-access=trusted` 标签，并不自动赋予或限制网络访问。Lab 2 创建 NetworkPolicy 后，策略才会根据命名空间和 Pod 标签决定是否允许访问。Pod 运行时间约为一小时；若实验中客户端已退出，需要重新创建后再继续。

### Lab 1：验证 Pod IP 分配与 Pod 间连通

先查看应用和客户端的 Pod IP、所在节点、Service ClusterIP 和 Endpoint：

```sh
kubectl -n calico-lab-app get pods -o wide
kubectl -n calico-lab-trusted get pods -o wide
kubectl get nodes -o wide
kubectl -n calico-lab-app get service,endpoints
```

**预期结果：** 不同 Pod 有各自的 Pod IP，`NODE` 列显示所在节点。Service 的 ClusterIP 是单独的虚拟 IP，Node IP 是节点自身的地址。

#### 可选：安排跨节点测试

对比上面输出中 `web` 和 `frontend` 的 `NODE`：若不同，下面的 Pod IP 测试就是跨节点通信；若相同，测试仍可验证同节点连通，但不能证明跨节点连通。

要让三个 curl 客户端都避开 Web 所在节点，请在至少有两个可调度节点时按[辅助文档](./ref/calico-same-and-cross-node.md)删除并重新创建带反亲和性的客户端。这会短暂中断客户端，但不会修改 Web Deployment；完成后返回这里，并重新检查 Pod 节点和 IP。只有一个节点时跳过此步骤，后续 Lab 仍可继续。

#### 查看 IPAM 配置

使用 `calico-ipam` 时，可以将上面 Pod 的 IP 与启用的 IPPool CIDR 对照；如果使用 `host-local` IPAM，地址分配来源不同。查看当前 IPPool：

```sh
kubectl get ippools
kubectl get ippools -o yaml
```

重点查看 `spec.cidr`（地址范围）、`spec.blockSize`（IPAM block 大小）、`spec.nodeSelector`（适用节点）、`spec.ipipMode` / `spec.vxlanMode`（封装方式）和 `spec.disabled`。字段可能因版本或配置而省略。使用 Calico IPAM 时，实验 Pod 的 IP 应在适用于该 Pod 和节点的池中；多个池时，需结合节点选择器和其他分配配置判断。

如果集群由 Tigera Operator 管理，还可以查看安装配置中的网络设置：

```sh
kubectl get installation.operator.tigera.io default -o yaml
```

检查 `spec.calicoNetwork.ipPools` 和 `spec.calicoNetwork.ipam`。如果提示资源不存在，集群可能不是由 Operator 管理，或 Installation 资源名称不同；这不代表 Calico 没有运行。此时以当前 IPPool 资源及集群实际使用的安装清单为准。

如果有权限登录节点，可以只读查看该节点上的 CNI 配置，确认 IPAM 插件类型。路径和文件名会因发行版和安装方式不同而变化，下面是常见示例：

```sh
sudo ls /etc/cni/net.d
sudo cat /etc/cni/net.d/10-calico.conflist
```

查看 Calico 插件配置中的 `ipam.type`：`calico-ipam` 表示由 Calico IPAM 分配地址，`host-local` 则使用不同的分配方式。不要修改 CNI 配置来完成实验；分享配置内容前应移除凭据等敏感信息。

如果环境已安装并配置 `calicoctl`，还可以查看 IPAM block 与节点关联信息：

```sh
calicoctl ipam show --show-blocks
```

#### 验证网络连通

从客户端通过 Service DNS 访问 Web 服务：

```sh
kubectl -n calico-lab-trusted exec frontend -- curl --max-time 5 -sS -o /dev/null -w "%{http_code}\n" http://web.calico-lab-app.svc.cluster.local
```

再查看 `web` Pod 的 IP：

```sh
kubectl -n calico-lab-app get pod -l app=web -o wide
```

将输出中的 Pod IP 代入下面命令，直接访问 Pod（不要输入占位符本身）：

```sh
kubectl -n calico-lab-trusted exec frontend -- curl --max-time 5 -sS -o /dev/null -w "%{http_code}\n" http://<WEB_POD_IP>
```

**预期结果：** Service 访问和直接访问 Pod IP 都返回 `200`。这验证了客户端 Pod 到服务 Pod 的跨命名空间网络连通；只有确认两者位于不同节点时，才能证明跨节点连通。若失败，先确认 Pod 就绪、Service 有 Endpoint，并检查 DNS、集群网络和已有策略。

### Lab 2：用 NetworkPolicy 控制访问

先让不受信任客户端也使用 `role=frontend` 标签，以便单独验证命名空间选择条件：

```sh
kubectl -n calico-lab-untrusted label pod attacker role=frontend --overwrite
```

此时 `frontend` 和 `attacker` 的 Pod 标签相同，但分别处于受信任和不受信任的命名空间；`other` 则处于受信任命名空间，但 Pod 标签不同。标签本身不会阻止访问。

先确认没有其他限制这些流量的策略时，三种客户端都能访问 Web：

```sh
kubectl -n calico-lab-trusted exec frontend -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://web.calico-lab-app.svc.cluster.local
kubectl -n calico-lab-trusted exec other -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://web.calico-lab-app.svc.cluster.local
kubectl -n calico-lab-untrusted exec attacker -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://web.calico-lab-app.svc.cluster.local
```

三条命令都应返回 `200`。创建 `allow-trusted-frontend.yaml`，内容如下：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-trusted-frontend
  namespace: calico-lab-app
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              lab-access: trusted
          podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 80
```

逐项理解这份策略：

- `metadata.name` 和 `metadata.namespace`：策略名为 `allow-trusted-frontend`，只在 `calico-lab-app` 命名空间内生效。
- `spec.podSelector`：选择此命名空间中带有 `app=web` 标签的 Pod，也就是上面 Deployment 创建的 Web Pod。它决定“保护谁”，不是“谁可以访问”。
- `policyTypes: Ingress`：策略控制所选 Pod 的入站流量，不控制它们的出站流量。
- `ingress.from`：决定“谁可以访问”。来源 Pod 必须位于带有 `lab-access=trusted` 标签的命名空间，并且自身带有 `role=frontend` 标签。同一个 `from` 项里的两个选择器是“并且”关系，必须同时匹配。
- `ports`：只允许访问目标 Pod 的 TCP 80 端口。

因此，受信任命名空间里的 `frontend` Pod 符合两个来源条件；同一命名空间里没有 `role=frontend` 标签的 `other` 不符合 Pod 条件；带有前端标签但位于不受信任命名空间的 `attacker` 不符合命名空间条件。策略创建后，目标 Web Pod 会受到入站隔离，只能通过适用的 NetworkPolicy 允许的流量访问。NetworkPolicy 的允许规则会累加；如果其他策略还允许了额外来源，这份策略不会撤销那些许可。

应用并检查策略：

```sh
kubectl apply -f learn-calico/allow-trusted-frontend.yaml
kubectl -n calico-lab-app describe networkpolicy allow-trusted-frontend
```

以上 `kubectl apply` 命令从仓库根目录执行。

重新执行上面的三条访问命令：

| 客户端 | 预期结果 | 原因 |
| --- | --- | --- |
| 受信任 `frontend` | 返回 `200` | 命名空间和 Pod 标签都匹配 |
| 受信任 `other` | 请求失败 | Pod 标签不匹配 |
| 不受信任 `attacker` | 请求失败 | Pod 标签匹配，但命名空间不匹配 |

失败的请求通常会超时。同一个 `from` 条目中的 `namespaceSelector` 与 `podSelector` 必须同时匹配（逻辑 AND）；如果拆成两个条目，则会变成满足任意一个即可（逻辑 OR）。策略生效可能需要短暂时间；如果结果不符，检查标签和其他策略。

该策略只限制目标 Web Pod 的入站流量，不限制其出站流量。

### Lab 3：默认拒绝未明确授权的流量

创建 `default-deny-ingress.yaml`，为应用命名空间中的所有 Pod 添加默认拒绝入站策略：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: calico-lab-app
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

它能“默认拒绝”，不是因为 YAML 里有特殊的 `defaultDeny` 字段，而是因为 NetworkPolicy 的规则语义：`podSelector: {}` 选中 `calico-lab-app` 命名空间中的所有 Pod，`policyTypes: Ingress` 让这些 Pod 进入入站隔离状态；策略又没有配置任何 `ingress` 允许规则，所以这条策略本身不允许任何入站流量。没有被任何适用策略允许的入站连接就会被拒绝。这个策略不限制出站流量；同时，其他 NetworkPolicy 中已经允许的入站流量仍然有效，因为允许规则会累加。

应用策略并验证只有 Lab 2 明确允许的受信任 frontend 仍能访问 Web：

```sh
kubectl apply -f default-deny-ingress.yaml
kubectl -n calico-lab-app get networkpolicy

# 受信任 frontend 仍可访问 Web（由 Lab 2 的策略允许）
kubectl -n calico-lab-trusted exec frontend -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://web.calico-lab-app.svc.cluster.local

# 其他客户端不能访问 Web
kubectl -n calico-lab-trusted exec other -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://web.calico-lab-app.svc.cluster.local
kubectl -n calico-lab-untrusted exec attacker -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://web.calico-lab-app.svc.cluster.local
```

**预期结果：** frontend 返回 `200`，other 和 attacker 的请求失败。默认拒绝策略本身不允许任何入站流量；NetworkPolicy 的允许规则是累加的，所以 Lab 2 中的允许策略继续生效。由于 Lab 2 已经隔离 Web 的入站流量，本步骤的 Web 访问结果不会变化；下一 Lab 将验证新增的数据库层也受到默认拒绝保护。

### Lab 4：按应用层实施微观分段

增加一个数据库层示例服务和代表 API 层的探测 Pod：

```sh
kubectl -n calico-lab-app create deployment db --image=nginx
kubectl -n calico-lab-app expose deployment db --port=80
kubectl -n calico-lab-app run api-probe --image=curlimages/curl --restart=Never --labels=app=api --command -- sleep 3600
kubectl -n calico-lab-app rollout status deployment/db --timeout=120s
kubectl -n calico-lab-app wait --for=condition=Ready pod/api-probe --timeout=120s
```

先确认默认拒绝生效：API 探测 Pod 此时还没有获准访问数据库。

```sh
kubectl -n calico-lab-app exec api-probe -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://db
```

**预期结果：** 请求超时或失败。若成功，检查是否存在其他允许该流量的策略，再继续。

创建 `allow-api-to-db.yaml`，只允许 API 探测 Pod 访问数据库层：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: calico-lab-app
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api
      ports:
        - protocol: TCP
          port: 80
```

应用策略并验证允许和拒绝的访问：

```sh
# API 探测 Pod 可访问数据库
kubectl apply -f allow-api-to-db.yaml
kubectl -n calico-lab-app exec api-probe -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://db

# 受信任前端不能绕过应用层直接访问数据库
kubectl -n calico-lab-trusted exec frontend -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://db.calico-lab-app.svc.cluster.local

# 受信任 frontend 仍可访问 Web
kubectl -n calico-lab-trusted exec frontend -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://web.calico-lab-app.svc.cluster.local

# 不受信任客户端不能访问 Web
kubectl -n calico-lab-untrusted exec attacker -- curl --max-time 3 -sS -o /dev/null -w "%{http_code}\n" http://web.calico-lab-app.svc.cluster.local
```

**预期结果：** API 探测 Pod 访问数据库和 frontend 访问 Web 返回 `200`；frontend 直连数据库及不受信任客户端访问 Web 会超时或失败。这样即使前端 Pod 被攻陷，也不能直接访问数据库层。

> 这里的 `db` 使用 Nginx 仅作为网络连通性测试目标，不是实际数据库。`api-probe` 用来模拟具有 `app=api` 标签的后端工作负载。

### 清理实验资源

实验完成后，先确认当前上下文是实验集群，并检查实验命名空间：

```sh
kubectl config current-context
kubectl get namespace calico-lab-app calico-lab-trusted calico-lab-untrusted --ignore-not-found
```

**删除命名空间会删除其中所有资源，包括与实验无关的资源；如果有其他资源，请勿继续。**

确认后删除实验命名空间及其中的 Deployment、Pod、Service 和 NetworkPolicy：

```sh
kubectl delete namespace calico-lab-app calico-lab-trusted calico-lab-untrusted --ignore-not-found
```

验证命名空间已删除：

```sh
kubectl get namespace calico-lab-app calico-lab-trusted calico-lab-untrusted --ignore-not-found
```

预期不再列出这三个命名空间。如果仍显示 `Terminating`，等待删除完成后再检查；若长时间未完成，可用 `kubectl describe namespace <命名空间名>` 查看原因。

如果不再需要，可以手动删除实验时在本地创建的 `default-deny-ingress.yaml`、`allow-api-to-db.yaml` 和可选的 `curl-anti-affinity.yaml`；仓库提供的 `allow-trusted-frontend.yaml` 无需删除。删除前请确认文件不含需要保留的内容。无需卸载 Calico 或修改集群网络配置。

### 遇到问题时

1. **Pod 未就绪：** 使用 `kubectl describe pod` 和 `kubectl logs` 检查事件、镜像拉取和容器状态。
2. **Service 不可达：** 检查 Pod 状态、标签、Service selector、Endpoint 和 DNS。
3. **策略结果不符：** 检查策略的 Namespace、`podSelector`、来源选择器、端口和协议；同时确认没有其他 NetworkPolicy 允许或限制相关流量。
4. **Calico 组件异常：** 检查 Calico Pod 状态和日志，并参考[官方故障排查文档](https://docs.tigera.io/calico/latest/operations/troubleshoot/)。

## 延伸阅读

- [Calico NetworkPolicy 文档](https://docs.tigera.io/calico/latest/network-policy/)
- [Kubernetes NetworkPolicy 文档](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Calico 网络文档](https://docs.tigera.io/calico/latest/networking/)
