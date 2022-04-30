## 前言

发现一篇关于Kubernetes上服务滚动更新相关的配置选项的文章，文章列出了最常用的几个配置项，解释了他们是怎么影响调度器对服务进行滚动更新的，同时还带出了`Kubernetes`项目中`Pod`这个逻辑单元的`Ready`状态是怎么确定的，并不是容器运行起来后`Pod`就进入`Ready`状态的。总之觉得是篇非常好的普及`Kubernetes`基础的文章，文章由本人完全手工翻译，尽量做到通顺易懂，英文好的同学可以直接看原文。

>原文标题：Kubernetes Deployments 滚动更新配置/ Kubernetes Deployments Rolling Update Configration.
>
>发布时间：February 26, 2020
>
>原文链接：https://www.bluematador.com/blog/kubernetes-deployments-rolling-update-configuration
>
>文章作者：Keilan Jackson
>
>文章译者：Keivn Yan



`Deployment`是`Kubernetes`中一种常用的`Pod`控制器，它对`Pod`提供了细粒度的全面控制：如何进行`Pod`配置、如何执行`Pod`更新，应运行多少`Pod`以及何时终止`Pod`。有许多这方面的资源会教你如何配置`Deployment`，但是你可能很难理解每个选项是如何影响滚动更新的执行方式的。在此博客文章中，我们将涵盖以下主题，以帮助您成为`Kubernetes` `Deployment`的专家：

- Kubernetes Deployment概貌；
- Kubernetes服务的滚动更新；
- 怎么定义`Pod`的Ready状态；
- Pod Affinity和Anti-Affinity；

## Deployment 概貌

`Deployment`实质上是`ReplicaSet`的上层包装。 `ReplicaSet`管理正在运行的`Pod`数量，`Deployment`在其之上实现功能从而拥有了`Pod`滚动更新，对Pod的运行状况进行健康检查以及轻松回滚更新的能力。 在常规运行期间，`Deployment`仅管理一个`ReplicaSet`，以确保期望数量的`Pod`正在运行：

![Deployment-Replicate-Pod之间的关系](https://cdn.learnku.com/uploads/images/202012/26/6964/J4SwgIE0mw.png!large)

在`Kubernetes`里我们不应直接操作由`Deployment`创建的`ReplicaSet`，对`ReplicaSet`执行的所有操作应改为在`Deployment`上执行，然后由`Deployment`管理更新`ReplicaSet`的过程。以下是一些通常在`Deployment`上执行的操作的示例`kubectl`命令：

```shell
# 列出默认命名空间下的所有Deployment
kubectl get deploy

# 通过定义文件更新Deployment
kubectl apply -f test.yaml

# 监控"test"这个Deployment的状态更新
kubectl rollout status deploy/test

# 暂停"test"这个Deployment的更新流程:
kubectl rollout pause deploy/test

# 恢复"test"这个Deployment的更新流程:
kubectl rollout resume deploy/test

# 查看"test"这个Deployment的更新历史:
kubectl rollout history deploy/test

# 回退test最近的更新
kubectl rollout undo deploy/test

# 把test这个Deployment回滚到指定版本
kubectl rollout undo deploy/test --to-revision=1
```



## Pod的滚动更新

使用`Deployment`来控制`Pod`的主要好处之一是能够执行滚动更新。滚动更新允许你逐步更新`Pod`的配置，并且`Deployment`提供了许多选项来控制滚动更新的过程。

控制滚动更新最重要的选项是更新策略。在`Deployment`的`YAML`定义文件中，由`spec.strategy.type`字段指定`Pod`的滚动更新策略，它有两个可选值：

- **RollingUpdate (默认值)**：逐步创建新的Pod，同时逐步终止旧Pod，用新Pod替换旧Pod。
- **Recreate**：在创建新Pod前，所有旧Pod必须全部终止。

大多数情况下，`RollingUpdate`是`Deployment`的首选更新策略。如果你的`Pod`需要作为单例运行，并且不允许在任何时间存在多副本，这种时候`Recreate`更新策略会很有用。

使用`RollingUpdate`策略时，还有两个选项可以让你微调更新过程：

- **maxSurge**：在更新期间，允许创建超过期望状态定义的`Pod`数的最大值。
- **maxUnavailable**：在更新期间，容忍不可访问的**Pod**数的最大值

`maxSurge`和`maxUnavailable`选项都可以使用整数（比如：2）或者百分比（比如：50%）进行配置，而且这两项不能同时为零。当指定为整数时，表示允许超期创建或者不可访问的`Pod`数。当指定为百分比时，将使用期望状态里定义的`Pod`数作为基数。比如：如果对`maxSurge`和`maxUnavailable`都使用默认值25％，并且将更新应用于具有8个容器的`Deployment`，那么意味着`maxSurge`为2个容器，`maxUnavailable`也将为2个容器。这意味着在更新过程中，将满足以下条件：

- 最多有10个`Pod`（8个期望状态里指定的`Pod`和2个`maxSurge`允许超期创建的`Pod`）在更新过程中处于Ready状态。
- 最少有6个`Pod`（8个期望状态里指定的`Pod`和2个`maxUnavailable`允许不可访问的`Pod`）将始终处于Ready状态。

值得注意的一点是，在考虑`Deployment`应在更新期间运行的`Pod`数量时，使用的是在`Deployment`的更新版本中指定的副本数，而不是现有`Deployment`版本的期望状态中指定的副本数。

可以用另外一种方式理解这两个选项：`maxSurge`是一次将创建的新`Pod`的最大数量，`maxUnavailable`是一次将被删除的旧`Pod`的最大数量。让我们具体看一下使用以下更新策略将具有3个副本的`Deployment`从"v1"更新为" v2"的过程：

```yaml
replicas: 3  
strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```
这个更新策略是，我们想一次新建一个`Pod`，并且应该始终保持`Deployment`中的`Pod`有三个是Ready状态。下面的动图说明了滚动更新的每一步都发生了什么。如果`Deployment`看到`Pod`已经完全部署好了将会把`Pod`标记为Ready，创建中的`Pod`标记为`NotReady`，正在被删除的`Pod`标记为Terminating。
![Deployment滚动更新Pod的过程](https://segmentfault.com/img/bVcMpH5)

## 怎么判读Pod是否Ready

`Kubernetes`自身实现了一个叫做Ready Pod的概念来辅助滚动更新。具体来说就是，`ReadinessProbe` （就绪探针）可以使`Deployment`逐步更新`Pod`，同时也可以使用它控制何时才能进行滚动更新，`Service`也使用它来确定应该将哪些`Pod`包含在服务的Endpoints中。就绪探针与活动性探针相似但不相同，活性探针使`kubelet`可以根据其“重新启动策略”来确定哪些`Pod`需要重新启动，并且它们与就绪性探针分开配置，不会影响`Deployment`的滚动更新的过程。
> **译者注：关于就绪探针和活性探针详细的解释可以看我以前的文章：[浅析Kubernetes Pod重启策略和健康检查](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485500&idx=1&sn=6197d294cc4f409c2a62a7997c431b68&chksm=fa80d9abcdf750bd6735e7b7481c225d4cbfaad2159427b049d8b229877392f4bef11469363e&token=730401594&lang=zh_CN#rd)**

一个Ready状态的`Pod`是指：`Pod`通过了就绪探针的测试，并且自创建以来经过了`spec.minReadySeconds`指定的秒数则被视为已经`Ready`。这些选项的默认值将导致`Pod`内部的容器启动后`Pod`立即进入Ready状态。

事实上，有几个原因通常让你并不想让容器启动后`Pod`立即进入Ready状态：

- 希望在接收流量前，`Pod`能够先通过健康检查。
- 服务需要先预热，然后再提供流量。
- 你要放慢部署速度，以减少对运行中的系统的影响。

对于`Web`应用程序，要求通过健康检查非常常见，这对于以最小的中断执行更新至关重要。下面是为了对一个Web应用进行健康检查而配置的就绪探针：

```yaml
readinessProbe:
          periodSeconds: 15
          timeoutSeconds: 2
          successThreshold: 2
          failureThreshold: 2
          httpGet:
            path: /health
            port: 80
```

这个探针要求对端口80上的`/health`的调用在2秒内成功完成，每15秒执行一次，并且在`Pod`准备就绪之前必须进行2次成功的调用。这意味着在最佳情况下，`Pod`将在约30秒内准备就绪。许多应用程序在启动后2秒钟之内无法立即提供服务，即使是简单的请求，因此应该为前1项或2次检查的失败做好准备，这种情况下实际需要约60秒的准备时间`Pod`才能进入Ready状态。

您还可以配置在容器上执行命令的就绪探针。这让你可以编写可执行的自定义脚本，并确定`Pod`是否已准备好，`Deployment`是否可以继续执行滚动更新：

```yaml
readinessProbe:
          exec:
            command:
              - /startup.sh
          initialDelaySeconds: 5
          periodSeconds: 15
          successThreshold: 1
```

在这个配置中，`Deployment`在`Pod`创建成功后将等待5秒钟，然后每15秒钟执行一次命令。脚本的exit code为0被认为是执行成功。使用命令脚本的灵活性让你可以执行以下类似操作，例如将数据加载到缓存中或预热JVM，或在不修改应用程序代码的情况下对下游服务进行运行状况检查。

我们将在这里讨论的最后一种情况是故意减慢更新过程，以最大程度地减少对系统的影响。乍一看似乎不需要，但是在某些情况下它可能非常有用，包括事件处理系统，监视工具和预热时间较长的`Pod`。通过在`Deployment`定义中指定`spec.minReadySeconds`字段可以轻松实现此目标。当指定`minReadySeconds`时，`Pod`必须运行这么多秒，而且其容器中的任何一个都不能崩溃，才能被`Deployment`视为进入Ready状态。

比如说，假设一个`Deployment`管控着5个`Pod`副本，它们从事件流读取、处理事件，然后把数据保存在数据库中。每个`Pod`需要预热60秒后才能全速处理事件，如果使用默认的选项值，`Pod`创建后立即进入Ready状态，但是它们在第一分钟内处理事件的速度会很慢。更新完成后，由于所有`Pod`同时预热，因此事件将会堆积在事件流里面。相反，你可以将`maxSurge`设置为1，将`maxUnavailable`设置为0，将`minReadySeconds`设置为60。这将确保一次创建一个新的`Pod`，在经过一分钟预热后新建的`Pod`才能进入Ready状态，并且旧`Pod`在新`Pod`就绪之前不会被删除。这样，就可以在大约5分钟的时间内更新所有`Pod`，并且能保持更新期间服务的稳定。

## Pod Affinity和Anti-Affinity

`PodAffinity`和`PodAntiAffinity`这两个配置让你可以控制将`Deployment`的`Pod`调度到哪些节点上。尽管这个功能并非`Deployment`控制器特有的，但对于许多应用程序来说可能非常有用。 在配置`PodAffinity`或`PodAntiAffinity`时，你必须选择在不同环境条件下要添加给新建的`Pod`的调度偏好。这里有两个选项：

- **requiredDuringSchedulingIgnoredDuringExecution**：除非节点与`Affinity`配置相匹配，否则即使没有匹配的节点，直接调度失败，也不会把`Pod`调度到不匹配的节点上去。
- **preferredDuringSchedulingIgnoredDuringExecution**：调度程序将尝试在与配置匹配的节点上调度Pod，但如果无法这样做，则仍将`Pod`调度在另一个节点上。

`podAffinity`性用于将`Pod`调度到某些节点上。如果知道某个`Pod`具有特定的资源要求（例如，只有一组特定的，带有GPU的节点或者某个区域中的节点），则通常需要进行`podAffinity`配置。或者你希望`Pod`与其他`Pod`并置在一个节点上：

```yaml
affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: "kubernetes.io/hostname"
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web
          topologyKey: kubernetes.io/hostname
```

相反地，`podAntiAffinity`在确保同属一个`Deployment`的`Pod`不被调度到同一个节点上时非常有用，上面例子偏向于将标签为`app:web`的`Pod`部署到不同的节点上，降低服务的所有`Pod`因为节点出问题同时出故障的可能性。



使用`affinity`配置时，要注意非常重要的一点是，`affinity`规则是在对`Pod`进行调度时进行评估的。而调度器无法预见`Pod`的调度位置，这意味着`affinity`配置可能无法达到预期的效果。考虑有一个拥有3个节点的集群，一个拥有3个`Pod`副本的使用了上面示例`affinity`规则的`Deployment`，`Deployment`把`maxSurge`配置成1。你期望的调度目标可能是，在每个节点上运行`Deployment`的一个`Pod`，但是由于`maxSurge`设置为1，在滚动更新期间调度器每次只能创建一个新`Pod`。这意味着随着时间的流逝，你可能最终会得到一个更新后没有任何这些`Pod`的节点，然后所有或大多数`Pod`将在下一次更新时移至该节点。调度程序不知道你将要终止旧的Pod，在做`affinity`规则判断时仍然会算上旧`Pod`，这就是导致上面说的可能情况的原因。如果确实需要在每个节点上只能运行一个`Pod`副本，则应使用`DaemonSets`控制器。如果您的应用程序可以接受，另一种选择是将更新策略更改为“重新创建”。这样，当调度程序评估`affinity`规则时，将不会算上旧的`Pod`。



`podAffinity`和`PodAntiAffinity`有很多选项可以影响`Pod`的调度方式，但通常无法保证滚动更新也能满足规则。在某些情况下，这是一个非常有用的功能，但是除非真的需要控制`Pod`的运行位置，否则应让`kubernetes`调度程序来做出这些决定。可以在[此处](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)找到有关`podAffinity`和`podAntiAffinity`的详细文档。

## 总结

我们已经介绍了`Deployment`的基本用法，滚动更新的工作方式以及用于微调更新和`Pod`调度的许多配置选项。此时，你应该能够使用更新策略，就绪探针和`Pod`关联性(`affinity`)来自信地创建和修改`Deployment`的定义文件，以达到应用程序期望的状态。有关`Deployment`支持的所有选项的详细参考，请查看`Kubernetes`文档。
