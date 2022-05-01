这是实现「 Kubernetes 集群零停机时间更新」系列文章的第三部分。在本系列的第二部分中，我们通过利用 Pod 生命周期钩子实现了应用程序Pod的正常终止，从而减轻了由于 Pod 未处理完已存请求而直接关机而导致的停机时间。但是，我们还了解到，在启动关闭序列后，Pod 会拒绝为新到来的流量提供服务，但实际情况是 Pod 仍然可能会继续接收到新流量。这意味着最终客户端可能会收到错误消息，因为它们的请求被路由到了不再能为流量提供服务的Pod。理想情况下，我们希望 Pod 在启动关闭后立即停止接收流量。为了减轻这种情况，我们必须首先了解为什么会发生Pod开始关闭时仍然会接收到新流量这个问题。

这篇文章中的很多信息都是从「 Kubernetes in Action」一书中学到的。除了在这里介绍的信息外，本书还提供了在 Kubernetes 上运行应用程序的最佳实践，因此强烈建议您阅读此书。

## Pod关闭序列

在上篇文章「[如何优雅地关闭Pod](https://mp.weixin.qq.com/s/fbQ2vdJ4bDJdOrhNFChOsQ)」中我们介绍了 Pod 被驱逐的生命周期，逐出序列的第一步是开始删除 Pod ，这会引发一系列事件，最终导致 Pod 从系统中删除。但是，上篇文章里我们没有谈论到的是，如何从上层的 Service 控制器中注销 Pod，使得 Pod 能停止接收流量。

>译注：我的理解是要在Pod真正停止运行前，要先把它从Service上拿掉，也就是摘流。

那么，是什么情况会导致 Pod 从 Service 中注销掉呢？要了解这一点，我们需要更深入一层，来了解从集群中删除Pod时都发生了什么。

通过 Kubernetes 的 API 将 Pod 从群集中删除后，该 Pod 在元数据服务器中被标记为要删除。这会向所有相关子系统发送一个 Pod 删除通知，然后处理该通知：

> 这里说的元数据服务器，指的是Kubernetes APIServer，而子系统则是Kubernetes的一些核心组件。

- Pod 所在节点上的`kubelet`将启动上一篇文章中描述的 Pod 关闭序列。
- 所有节点上运行的`kube-proxy`守护程序将从 iptables 中删除 Pod的 IP 地址。
- 端点控制器将从有效端点列表中删除该 Pod，反映到我们这个例子中来就是从 Service 管控的端点（endpoint）列表中移除这个 Pod 端点。

我们无需了解每个系统的详细信息。这里的重点是涉及多个子系统，这些系统可能在不同的节点上运行，而上面列举的这些操作是会并行发生的。因此，在将 Pod 从所有活动列表中删除之前，Pod 很有可能已经开始执行 preStop 钩子并接收到了 TERM 信号。这就是即使 Pod 在启动关闭序列后，仍继续接收到流量的原因。

## 摘流方案

从表面上看，我们可以将上面那些事件序列串联起来，禁止他们并行进行，直到从所有相关子系统注销了要删除的 Pod 之后，再开始 Pod 的关闭序列。但是，由于 Kuberenetes 系统的分布式性质，在实践中很难做到这一点。如果节点之一遇到网络阻隔会怎样？是否要无限期地等待事件传播？如果该节点重新恢复联机怎么办？如果你的Kubernetes 集群有成千上万个节点怎么办？

不幸的是，现实情况是并不存在防止所有中断的完美解决方案。但是，我们可以做的是在 Pod 关闭序列中引入足够的延迟，以捕获99％的情况。为此，我们在preStop挂钩中引入了一个 sleep指令，以延迟 Pod 关闭序列。接下来，让我们看看在我们的例子中它是如何工作的。

我们会更新一直以来使用的资源定义文件，使用sleep 命令引入延迟来作为要执行的 preStop 钩子的一部分。在「 Kubernetes in Action」中，作者 `Lukša` 建议使用5到10秒的延迟，因此在这里我们将使用5秒延迟作为 preStop 钩子的一部分：

```yaml
lifecycle:
  preStop:
    exec:
      # Introduce a delay to the shutdown sequence to wait for the
      # pod eviction event to propagate. Then, gracefully shutdown nginx.
      command: ["sh", "-c", "sleep 5 && /usr/sbin/nginx -s quit"]

```

现在，让我们推演一下这个示例关闭序列中会发生什么。像上一篇文章的分析一样，我们将使用`kubectl drain`逐出节点上的 Pod。这将会发送一个Pod deletion 事件，该事件会同时通知给 kubelet 和 Endpoint Controller（端点控制器，这里指的是 Pod 上层的 Service控制器）。在此，我们假设 preStop 钩子在 Service 从自己可用列表中移除 Pod 之前启动。

![驱逐节点上的Pod，会发送一个Pod Deletion事件](https://cdn.learnku.com/uploads/images/202103/15/6964/3Yp5J5lbV3.png!large)

在 preStop 钩子执行时，首先会延迟5秒执行第二条关闭Nginx的命令。在这个期间，Service 将会从自己的可用列表中移除 Pod。

![关闭程序被延迟的同时Service会从列表中去掉要关闭的Pod](https://cdn.learnku.com/uploads/images/202103/15/6964/hncAy2msiJ.png!large)

在此延迟期间，Pod 仍处于运行状态，因此即使其接收到新的连接请求，它仍能够处理连接。此外，在将 Pod 从Service 中移除后，客户端的连接请求，将不会路由到将要关闭的 Pod 上。因此，在这种情况下，假如 Service 在延迟期间内处理完这些事件，集群将不会有停机时间。

最后，preStop 钩子进程从休眠中醒来并执行关闭 Nginx 容器，从节点中删除容器：

![](https://cdn.learnku.com/uploads/images/202103/15/6964/7mhFKu6w9g.png!large)

![](https://cdn.learnku.com/uploads/images/202103/15/6964/BckYUTPtGq.png!large)

此时，我们就可以安全地在Node1上进行任何升级，包括重启节点加载新的内核版本。如果我们已经启动了一个新节点来容纳Node1运行的工作负载，那么我们也可以关闭Node1节点。

## 重新创建Pod

如果你已经看到了这里，你可能想知道如何重新创建最初被调度到维护节点上的 Pod。现在，我们知道了如何正常关闭 Pod，但是如果要维持运行中的 Pod 的数量，该怎么办？这就要靠 Deployment 控制器发挥作用了。

Deployment 控制器负责在集群上维护指定的期望状态，如果你能回想起我们的资源定义文件里的内容，你会发现我们不是直接创建的 Pod。 取而代之的是我们通过给 Deployment 提供创建Pod的模板，让它来自动为我们管理 Pod。下面是定义中的 Pod 模板部分的内容：

```yaml
template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15
        ports:
        - containerPort: 80
```

模板指定了创建一个运行`nginx:1.15`镜像的容器的Pod，指定 Pod 携带的标签为`app:nginx`，向外暴露80端口。

除了Pod模板之外，我们还为 Deployment 资源提供了一个配置，用于指定应维护的 Pod 副本数：

```yaml
spec:
  replicas: 2
```

这会通知 Deployment 控制器它应始终维持有两个 Pod 运行在集群上。每当运行的 Pod 数量下降时，Deployment 控制器都会自动创建一个新的Pod来替换它。因此，在我们这个例中，当我们使用 `kubectl drain` 操作从节点上驱逐 Pod 时，Deployment 控制器会在其他可用节点上自动重新创建 Pod，保持当前状态与定义里指定的期望状态一直。

## 总结

总而言之，在 preStop 钩子有足够的延迟和正常终止的情况下，我们现在能在单个节点上正常关闭 Pod 了。利用Deployment，我们可以自动重新创建被关闭的 Pod。但是，如果我们想一次替换集群中的所有节点怎么办？

如果我们天真地重启所有节点，因为服务负载均衡器可能没有可用的Pod，而导致系统停机。更糟糕的是，对于有状态的系统，这样操作可能会让仲裁机制失效。

为了处理这种情况，Kubernetes 提供了一个称为`PodDisruptionBudgets`的功能，该功能指定了在任何给定时间点 Kubernetes 可以承受的关闭的 Pod 数量的上线。在本系列的下一也是最后一部分，我们将介绍如何使用它来控制同时发生的节点驱逐事件的数量。

### 推荐阅读

- [如何优雅地关闭Kubernetes集群中的Pod](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247487192&idx=1&sn=52fc08998b5f4c4cd33be5052d6bbe28&chksm=fa80df4fcdf756594bb4a07e5680e32eb1cce9eb1ce5b2eee11d50b8098840b27fa07a5b07b3&token=1308265118&lang=zh_CN#rd)
- [Deployment应用详解](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485643&idx=1&sn=6460bf2e170e4b2e8ebb2882bfe7c60f&chksm=fa80d95ccdf7504ad9b5e3ba7ad3dad6a25347a7b0aad4636523cb1ba878cebbc480bf2153a0&token=1308265118&lang=zh_CN#rd)





