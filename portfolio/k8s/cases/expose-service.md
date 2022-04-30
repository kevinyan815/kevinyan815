`Kubernetes`支持多种将外部流量引入集群的方法。`ClusterIP`、`NodePort`和`Ingress`是三种广泛使用的资源，它们都在路由流量中发挥作用。每一个都允许您使用一组独特的功能和折衷方案来公开服务。

### 背景

默认情况下，`Kubernetes`上运行的服务都是在自己的 [Pod](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247485464&idx=1&sn=00ca443bbcd4b2996efdede396b6c667&chksm=fa80d98fcdf7509944d63f618264e36cd8082a77e23aa36428a3d57a2f4189bcce4e52986967&token=2075750696&lang=zh_CN#rd) 里过着与世隔绝的生活，外部无法打扰他们。我们可以通过创建  [Service](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247486082&idx=1&sn=42a9bc8fcfc9da09445e9e2f4cf2fb96&chksm=fa80db15cdf752039494992f71a3bc488cf386841bd1aaaa44115f5e7f155ba55ce468ec89ee&token=2075750696&lang=zh_CN#rd) 使容器供外部世界可见，这个“外部世界” 即可以整个集群、也可以是整个互联网。

`Service`将流量路由到`Pod`内的容器中。`Service`是一种用于在网络上公开`Pod`的抽象机制。每个`Service`有一个类型——`ClusterIP`、`NodePort`或`LoadBalancer`。这些定义了外部流量如何到达服务。

但是光有`Service`也不行 ，有时候我们需要将不同域名和URL路径上的流量路由到集群内部，这就需要`Ingress`帮助才行。

### ClusterIP

ClusterIP 是默认的`Service`类型，不指定`Type`时默认就是`ClusterIP`类型的`Service`。`ClusterIP`在集群内提供网络连接。它通常无法从外部访问。我们将这些`ClusterIP Service`用于服务之间的内部网络。

```yaml
apiVersion: v1
kind: Service
spec:
  metadata:
    name: my-service
  selector:
    app: my-app
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

上面的示例定义了一个`ClusterIP Service`。到`ClusterIP` 上端口 80 的流量将转发到你的`Pod` 上的端口 8080  (targetPort配置项)，携带 `app: my-app`标签的 Pod 将被添加到 `Service`中作为作为服务的可用端点。

可以通过运行 `kubectl get svc my-service` 查看分配的 IP 地址。集群中的其他服务可以使用 10.96.0.1:80 与这个的 Service 管控的服务进行交互。

```shell
➜  kubectl get svc app-service
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
my-service        ClusterIP   10.96.0.1        <none>        8080:80/TCP       63d
```



可以使用 `spec.clusterIp` 字段手动将 ClusterIP 设置为特定 IP 地址：

```yaml
spec:
  type: ClusterIP
  clusterIp: 123.123.123.123
```

### NodePort

`NodePort`在固定端口号上公开向集群外部暴露服务，它允许从集群外部访问该服务，在集群外部需要使用集群的 IP 地址和`NodePort`指定的端口才能访问。 创建`NodePort Service`将在集群中的每个`Node`上开放该端口。 `Kubernetes`会自动将端口流量路由到它所连接的服务。 下面是一个 `NodePort Service` 的示例：

```yaml
apiVersion: v1
kind: Service
spec:
  metadata:
    name: my-service
  selector:
    app: my-app
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

`NodePort`的定义与`ClusterIP Service`具有相同的属性。唯一的区别是把类型设置成了："NodePort"。 `targetPort`字段仍然是必需的，因为`NodePort`由`ClusterIP`提供支持。

创建`NodePort Service`的同时还会自动创建一个`ClusterIP` 类型的`Service`，`NodePort`会将端口上的流量路由给`ClusterIP` 类型的 Service。

这也就是为什么下面我们查看`NodePort Service`时发现他也是有 ClusterIP 的原因：

```shell
➜  kubectl get svc my-service
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
my-service        NodePort    10.96.44.244     <none>        8080:30176/TCP    56d
```

使用上述例子创建`NodePort Service`，`Kubernetes`将会从30000-32767这个范围随机分配一个端口作为`NodePort`端口，不过我们可以通过设置`ports.nodePort`字段来手动指定端口：

```yaml
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
      nodePort: 32000
      protocol: TCP
```

这个会将 32000 端口上的流量通过`Service`最终路由给`Pod`里容器的 8080 端口。

您可以使用`NodePort`快速设置用于开发环境的服务或在其上公开`TCP`或`UDP`服务，但是对于公开`HTTP`服务来说`NodePort`不是一个的理想选择，因为其使用的都是非`HTTP`标准的端口，我们需要使用其他替代方案。

### Ingress

`Ingress` 实际上是与`Service`完全不同的资源，算是`Service`上面的一层代理，通常在 `Service`前使用`Ingress`来提供`HTTP`路由配置。它让我们可以设置外部 URL、基于域名的虚拟主机、SSL 和负载均衡。

给`Service`前面加`Ingress`，你的集群中需要有`Ingress-Controller`才行。有多种控制器可供选择。大多数主要的云提供商都有自己的`Ingress-Controller`，与他们的负载平衡基础设施相集成。如果是自建K8S集群，通常使用`nginx-ingress`作为控制器，它使用`NGINX`服务器作为反向代理来把流量路由给后面的`Service`。

> 关于控制器Nginx-Ingress的安装部署，请参考：https://kubernetes.github.io/ingress-nginx/deploy/ 后面介绍`Ingress`实践的文章也会再细说。

可以使用`Ingress` 资源类型创建`Ingress`。 kubernetes.io/ingress.class 注释可让你指明正在创建的`Ingress`分类。如果集群里安装了多个`Ingress-Controller`这将很有用，也可以将不同的`Service`分别挂在不同分类的`Ingress`下面，增加一些高可用性。

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: my-service
              servicePort: 80
    - host: another-example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: second-service
              servicePort: 80
```

上面定义了两个`Ingress`端点。第一个主机规则将 example.com 流量路由到`my-service`服务上的端口 80。第二条规则将 another-example.com 流量路由到`second-service`。

如果想使用 `HTTPs` 访问服务，可以通过在`Ingress` 规范中设置`tls`字段来配置 `SSL`：

```yaml
spec:
  tls:
    - hosts:
      - example.com
      - secretName: my-secret
```

不过前提是在集群中需要通过`Secret`对象配置这些域名的证书信息。

当需要处理来自多个域名 和 URL 路径的流量时，应该使用`Ingress`。它让我们可以使用声明性语句配置路由和`Service`。`Ingress`控制器将提供你的路由并将它们映射到服务。

### 总结

`ClusterIP`、`NodePort`、`Ingress`将流量路由到集群中的服务。每一个都是为不同的用例设计的。`ClusterIP`更多是为集群内服务的通信而设计，某些向集群外部暴露的`TCP`和`UDP`服务适合使用`NodePort`。而如果向外暴露的是`HTTP`服务，且需要提供域名和`URL`路径路由能力时则需要在`Service`上面再加一层`Ingress`做反向代理才行。



可能对`Ingress`，`Ingress-Controller`还是有一点模糊，后面再写一篇`Ingress`的实践文章，做个扫盲记录。
