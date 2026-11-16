# Calico Lab 辅助文档：将 curl 客户端调度到其他节点

本文配合[动手实验](../README.md#动手实验)使用。如果三个 curl 客户端和 Web Pod 都在同一个节点，可以重新创建客户端，并使用 Pod 反亲和性让它们避开 Web 所在节点。Web Deployment 和 Service 不需要修改。

## 同节点通信需要 CNI 吗？

需要。普通 Pod 创建时，容器运行时会调用 CNI 插件配置网络接口和 IP。同节点 Pod 通信也需要这些网络接口及节点上的转发；只是数据通常不需要离开节点。

```text
同节点：客户端 Pod → 节点网络转发和策略 → Web Pod
跨节点：客户端 Pod → 客户端节点 → 节点间网络 → Web 节点 → Web Pod
```

跨节点流量可能使用路由、VXLAN 或 IP-in-IP，具体取决于 Calico 配置。CNI 负责配置网络，而不是每个数据包都重新调用一次 CNI 插件。

## 1. 检查当前节点

```sh
kubectl config current-context
kubectl -n calico-lab-app rollout status deployment/web --timeout=120s
kubectl -n calico-lab-app get pods -l app=web -o wide --show-labels
kubectl -n calico-lab-trusted get pods -o wide --show-labels
kubectl -n calico-lab-untrusted get pods -o wide --show-labels
kubectl get nodes -L kubernetes.io/hostname
```

对比 `NODE` 列。如果 `web` 与某个客户端已经在不同节点，就可以用该客户端进行跨节点测试，无需重新创建。例如，`web` 在 `slave2`、`frontend` 在 `slave3`，两者已经跨节点。

以下步骤适用于需要将**三个客户端都调度到 Web 所在节点之外**的情况。确认 Web 已就绪并带有 `app=web` 标签，还要有至少一个不同于 Web 所在节点的可调度 `Ready` 节点。节点需带有 `kubernetes.io/hostname` 标签并满足资源、污点等条件；只有一个可用节点时无法完成跨节点实验。

## 2. 创建客户端配置

创建本地文件 `curl-anti-affinity.yaml`：

```yaml
apiVersion: v1
kind: List
items:
  - apiVersion: v1
    kind: Pod
    metadata:
      name: frontend
      namespace: calico-lab-trusted
      labels:
        role: frontend
    spec:
      restartPolicy: Never
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: web
              namespaces:
                - calico-lab-app
              topologyKey: kubernetes.io/hostname
      containers:
        - name: frontend
          image: curlimages/curl
          command: ["sleep", "3600"]
  - apiVersion: v1
    kind: Pod
    metadata:
      name: other
      namespace: calico-lab-trusted
      labels:
        role: other
    spec:
      restartPolicy: Never
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: web
              namespaces:
                - calico-lab-app
              topologyKey: kubernetes.io/hostname
      containers:
        - name: other
          image: curlimages/curl
          command: ["sleep", "3600"]
  - apiVersion: v1
    kind: Pod
    metadata:
      name: attacker
      namespace: calico-lab-untrusted
      labels:
        role: attacker
    spec:
      restartPolicy: Never
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: web
              namespaces:
                - calico-lab-app
              topologyKey: kubernetes.io/hostname
      containers:
        - name: attacker
          image: curlimages/curl
          command: ["sleep", "3600"]
```

每个客户端使用相同的反亲和性：`labelSelector` 和 `namespaces` 找到应用命名空间中的 Web Pod；`topologyKey` 指定以节点为范围。`requiredDuringSchedulingIgnoredDuringExecution` 要求调度器避开已有 Web Pod 的节点。三个客户端可以调度到同一个其他节点，也可以分布在不同节点。

这份配置恢复准备阶段的客户端标签。如果已完成 Lab 2，重新创建后需要重新执行 Lab 2 中给 `attacker` 设置 `role=frontend` 的命令，才能继续验证命名空间选择条件。

## 3. 删除并重新创建客户端

**这会中断三个客户端。** 先确认当前上下文以及以下 Pod 均为本实验创建的独立测试 Pod，且没有需要保留的数据；不要在生产环境照搬。独立 Pod 的亲和性不能通过修改现有 Pod 来完成迁移，因此需要重新创建。

删除受信任命名空间中的两个客户端：

```sh
kubectl -n calico-lab-trusted delete pod frontend other --wait=true
```

删除成功后，删除不受信任命名空间中的客户端：

```sh
kubectl -n calico-lab-untrusted delete pod attacker --wait=true
```

确认删除成功后，创建带反亲和性的客户端：

```sh
kubectl apply -f curl-anti-affinity.yaml
```

创建成功后，等待客户端就绪：

```sh
kubectl -n calico-lab-trusted wait --for=condition=Ready pod --all --timeout=120s
kubectl -n calico-lab-untrusted wait --for=condition=Ready pod --all --timeout=120s
```

若删除或创建失败，不要继续；如果等待超时，使用 `kubectl -n <命名空间> describe pod <Pod名>` 查看调度事件。没有符合条件的节点时，客户端会保持 `Pending`。

## 4. 验证并返回 Lab

```sh
kubectl -n calico-lab-app get pods -l app=web -o wide
kubectl -n calico-lab-trusted get pods -o wide
kubectl -n calico-lab-untrusted get pods -o wide
```

**预期结果：** 三个客户端都已就绪，且各自的 `NODE` 都不同于 Web Pod 的 `NODE`。它们的 IP 可能因重新创建而改变。查看当前 Web Pod 的 IP，回到 [Lab 1](../README.md#lab-1验证-pod-ip-分配与-pod-间连通)进行直接访问测试。

反亲和性只在调度时生效。如果 Web Pod 不存在，规则没有匹配对象；如果 Web 之后被重新调度，已运行的客户端也不会自动迁移，测试前应重新检查节点。

实验完成后，按[清理步骤](../README.md#清理实验资源)删除实验资源；若不再需要，可手动删除本地创建的 `curl-anti-affinity.yaml`。
