这是我们实现 Kubernetes 集群零停机时间更新的系列文章的第四部分也是最后一部分。在前两篇文章 「[ 如何优雅地关闭Kubernetes集群中的Pod](http://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247487192&idx=1&sn=52fc08998b5f4c4cd33be5052d6bbe28&chksm=fa80df4fcdf756594bb4a07e5680e32eb1cce9eb1ce5b2eee11d50b8098840b27fa07a5b07b3&token=2067195606&lang=zh_CN#rd) 」和「 [借助 Pod 删除事件的传播实现 Pod 摘流](http://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247487210&idx=1&sn=d73ca74b2e1bc1f969ac094ea44a5b28&chksm=fa80df7dcdf7566bce2f91da94cf33222e8b1bdea147f69403c4139d520641f3c09acc62c927&token=2067195606&lang=zh_CN#rd)」中，我们重点介绍了如何正常关闭集群中现有的Pod。我们介绍了如何使用 preStop 钩子正确关闭Pod，以及为什么在 Pod 关闭序列中增加延迟以等待删除事件在群集中传播很重要。这些可以处理一个Pod的终止，但不能保证我们在需要关闭多个 Pod时还能让服务正常运行。在本文中，我们将使用 Kubernetes 提供`PodDisruptionBudgets` 或者简称`PDB`来减轻这种风险。

>PDB是Kubernetes中用来保证集群中始终有指定的Pod副本数处于可用状态，它与Deployment中指定的**maxUnavailable**的区别是，后者是用来使用 Deployment 对应用进行滚动更新时保障最少可服务副本数的！而 ReplicaSet Controller，也并不能给保证集群中始终有几个可服务副本，它是负责尽快的让实际副本数跟期望副本数相同的，不会保证中间某些时刻的实际副本数。**Kubernetes 的 PDB 是用来保证应用在每个时刻最少可用Pod副本数的，对那些Voluntary（自愿的）Disruption做好Budgets(预算方案)**。
>
>PDB是针对Voluntary Disruption场景设计的，属于Kubernetes可控的范畴之一，而不是为Involuntary Disruption（非自愿中断设计）设计的，自愿中断主要是一些系统维护和升级更新的操作，而非自愿中断一般都是些硬件和网络故障导致的中断。一些集群会对Node进行自动管理，因此需要使用PDB来保障应用的HA。





## PDB：预算可容忍的故障数

Pod 中断预算（PDB）是一种在给定时间可容忍的中断数量（故障预算）的指标。每当计算出服务中的 Pod 中断会导致服务降至PDB以下时，操作就会暂停，直到可以维持PDB为止。这意味着在等待更多 Pod 可用之前，可以暂时停止逐出Pod，以免驱逐 Pod 而超出预算。

要配置一个PDB，我们需要在 Kubernetes 里创建一个`PodDisruptionBudgets`资源对象（后面简称PDB对象）用来匹配服务中的Pod。举个例子来说，我们想要创建一个PDB对象让我们之前使用Deployment创建的Nginx应用始终保持至少一个Pod可用，那么我们可以应用下面的配置：

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: nginx-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: nginx
```

这会告诉 Kubernetes 我们想要在任意时间点都有至少一个匹配标签`app: niginx`的Pod在集群中可用。使用此方法，我们可以促使Kubernetes 保证在自愿中断（更新/ 维护）进行时服务至少有一个Pod是可用的，避免服务停机。

## PDB的工作原理

为了说明 PDB 是如何工作的，让我们回到我们的一直以来使用的示例。为了简单起见，在示例中，我们将忽略任何 preStop 钩子，就绪性探针和服务请求。我们还将假设我们要对集群节点进行一对一替换。这意味着我们将通过使节点数量加倍，在新节点上运行重建的 Pod。

在图示中我们从两个节点的原始群集开始：

![](https://cdn.learnku.com/uploads/images/202103/27/6964/NiBKgJdBJw.png!large)

我们提供了两个额外的节点来运行新的虚拟机镜像。最终将会在新节点上创建 Pod 替换运行在旧节点上的Pod。

![](https://cdn.learnku.com/uploads/images/202103/27/6964/qqVk8fRIuH.png!large)

要替换服务Pod所在的节点，我们首先需要清空旧节点。在此示例中，让我们看看如果同时向运行Nginx Pod的两个节点发出`kubectl drain`命令时会发生什么。排空Node上Pod的请求将在两个线程中发出（实践时，可以使用两个终端分别运行`kubectl drain`命令），每个线程管理一个节点的排空执行序列。

注意，在这里我们，假设 kubectl drain 命令会立即发出驱逐请求。实际上，drain 操作首先会涉及对节点进行标记（给节点打上 NoSchedule的 标记），以便不会把 Pod 重新调度到旧节点上去。

![标记节点不可调用](https://cdn.learnku.com/uploads/images/202103/27/6964/yfLrJEa9VR.png!large)

节点标记完成后，负责排空节点的线程开始逐出节点上的Pod。这个过程中线程首先会去控制中心查询，看驱逐 Pod 是否会导致服务可用Pod数下降到配置的 PDB 以下。

这里需要注意的是，控制台会将并发请求串行化，一次处理一个PDB查询。这样，在这种情况下，控制平面将成功响应其中一个请求，而使另一个请求失败。这是因为第一个请求基于两个可用Pod的。允许此请求会将可用的 Pod 数量减少到1，PDB 得以维持。当控制中心允许请求继续进行时，便会将其中一个容器逐出，从而变得不可用。之后，当处理第二个请求时，控制平面将拒绝它，因为允许该请求会将可用Pod的数量降至0，低于我们配置的PDB。

鉴于此，在示例中，我们假定节点1是获得成功响应的节点。在这种情况下，节点1负责排空操作的线程将继续逐出 Pod，而节点2的排空线程将会等待并在稍后重试：

![串行化逐出请求，允许线程1的请求，因为不满足PDB拒绝线程2的请求](https://cdn.learnku.com/uploads/images/202103/28/6964/0Z8CdxzrO9.png!large)

![驱逐Node1上的Nginx Pod](https://cdn.learnku.com/uploads/images/202103/28/6964/VmoCkjgJ0r.png!large)

当节点1上的Nginx Pod被驱逐后，Pod 会立即被 Deployment 重建出来并调度到集群的节点上。因为我们集群的旧节点都已经被打上了`NoSchedule`的标记，所以调度器会选择一个新节点进行调度。

![重建Pod被调度到了Node3这个新节点上](https://cdn.learnku.com/uploads/images/202103/28/6964/0hCmmjWIuP.png!large)

至此，成功在新节点上完成了Pod更换，并且排空了原始节点Node1，用于排空Node1的线程就完成任务了。

从现在开始，当 Node2 的排空线程再次去控制中心查询 PDB 时，将会得到成功响应。这是因为有一个正在运行的Pod （刚才在Node3上新建的 Pod）不在考虑驱逐的序列中，因此，让 Node2 的排空线程继续前进不会将可用Pod的数量降到 PDB 以下。所以线程2会继续前进逐出 Node2 上的 Pod，完成驱逐过程：

![线程2再次查询，可以满足PDB后开始驱逐Node2上的Pod](https://cdn.learnku.com/uploads/images/202103/28/6964/n4AO0EZykS.png!large)

![驱逐Node2上的Nginx Pod](https://cdn.learnku.com/uploads/images/202103/28/6964/cG6RKMl4eh.png!large)

![在Node4上新建Pod，完成整个集群Node升级过程](https://cdn.learnku.com/uploads/images/202103/28/6964/rGuh7ePsLQ.png!large)

至此，我们就成功地将两个 Pod 都迁移到了新节点上，而没有遇到无可用 Pod 可以为应用程序提供服务的情况。而且，我们不需要在两个负责排空节点的线程之间有任何协调逻辑，Kubernetes 会根据我们提供的配置为我们处理所有工作！

## 总结

将我们在本博客系列中的内容都联系起来，我们介绍了：

- 如何使用生命周期钩子来实现平滑关闭我们的应用程序的能力，从而不会导致服务硬重启。[ Part II：如何优雅地关闭Kubernetes集群中的Pod](http://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247487192&idx=1&sn=52fc08998b5f4c4cd33be5052d6bbe28&chksm=fa80df4fcdf756594bb4a07e5680e32eb1cce9eb1ce5b2eee11d50b8098840b27fa07a5b07b3&token=2067195606&lang=zh_CN#rd)
- Pod是怎么从Kubernetes系统中被移除的，以及为什么必须在Pod关闭序列中引入延迟。[Part III: 借助 Pod 删除事件的传播实现 Pod 摘流](http://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247487210&idx=1&sn=d73ca74b2e1bc1f969ac094ea44a5b28&chksm=fa80df7dcdf7566bce2f91da94cf33222e8b1bdea147f69403c4139d520641f3c09acc62c927&token=2067195606&lang=zh_CN#rd)
- 如何指定Pod中断预算（PDB），以确保我们始终有一定数量的Pod可用，以便在需要中断的情况下为运行的应用程序提供连续不中断的服务。

当所有这些功能一起使用时，我们可以实现集群维护时服务零停机时间的目标！不过不要只听我在这里说，要继续下去把这里介绍的功能应用在练习和实践中。
