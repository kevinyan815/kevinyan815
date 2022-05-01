问题描述：

1. 阿里云ecs服务器硬件故障，执行重启修复，导致k8s一台node节点重启。 应用在仍有可用节点的情况下

未对服务进行问题节点摘流。

2. 平时部署的时候因为流量大，应用没有平滑重启，导致部分请求错误。



分析问题的原因：

1. 节点宿主机重启或者其他原因对Pod进行驱逐时，会广播删除时间， 并行地开始执行Pod关闭序列， Service摘流等等这些操作。如果在未摘流前

​      关闭了Pod， 通过其他存活 Pod 注册到 Etcd 的 NodeIP, 通过Service还是有概率把流量路由到正在关闭的节点。

2. 现在应用程序本身不支持接收OS信号进行平滑重启 





Kubernetes 相关概念：



- Pod 应用运行在容器里， 容器装在Pod里，Pod里可以有多个容器，但只能有一个运行主进程的主容器
  其他容器用来辅助，即边车模式（sidecar）。目前公司用的单容器Pod。
- Deployment ，一般不会单独创建Pod， 而是通过给Deployment提供Pod模板让Deployment维护Pod。

​      Deployment是最常用的控制器，用来对应用Pod的副本数、版本（版本在replicaSet上，但是也是由Deployment维护，不推荐单独创建replicaSet)

​      进行维护，让集群当前的状态始终等于它定义里指定的期望状态，副本多了就杀掉，少了就新建。

- Service， Service 是另外一种控制器，用来管控一组应用Pod的流量访问，通过selector指定Pod的标签来把符合条件的Pod加到它的Endpoints列表里。

​     为应用提供统一的访问方式（Pod重建IP会变，且Pod IP 节点之间不能相互访问，所以需要Service），还包括负载均衡策略和探活。

- 控制器Deployment和 Service 并没有实体，不会说部署到哪个节点上，是他们管理调度的Pod部署在具体节点上。





解决方案：

1.  利用Pod的生命周期钩子preStop 为Pod的关闭序列引入延迟，保证Pod能先从Service的Endpoints列表里去除掉后再进入Pod的关闭序列。

 技术背景：Kubernetes删除Pod前会向Kubernetes集群内广播Pod的删除事件，会同时有几个子系统接收广播处理事件：

 ① 节点上的kubelet 会启动Pod的关闭序列，执行生命周期钩子，发送SIGTERM，等待Pod内的应用程序处理完成后删除Pod，如果等待 terminationGracePeriodSeconds 配置的时间（默认30s）应用仍未终止则向Pod发送SIGKILL后强制删除Pod

 ② Service控制器监听到Pod删除的事件后会从自己维护的端点列表中删除Pod，流量不再路由给要删除的Pod。

 上面①和②是并行执行的，有可能Pod为从Service注销时，Pod已经进入了关闭序列，这时再把请求流量路由给正在关闭的Pod，应用是无法处理的会给客户端返回错误。

 所以需要在preStop钩子引入延迟，保证Pod先从Service摘流再关闭。

```yaml

      containers:
      - args:
        - /bin/bash
        - -c
        - /vip-passport
        env:
        - name: RESTART_
          value: "1616988349"
        envFrom:
        - configMapRef:
            name: vip-passport
        image: xxxxx
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - sh
              - -c
              - sleep 15

```

2. 应用增加对OS信号的监听，Pod关闭时Kubernetes会先向Pod发送SIGTERM，应用接收到信号后主动去Etcd上注销，并调用gRPC的平滑关闭流程。

```go

errChan := make(chan error)

stopChan := make(chan os.Signal)

signal.Notify(stopChan, syscall.SIGTERM, syscall.SIGINT, syscall.SIGKILL)

go func() {
     if err := grpcServer.Serve(lis); err != nil {
        errChan <- err
     }
}()

select {
case err := <- errChan:
   panic(err)
case <-stopChan:
   log.Info("server terminating", topic)
   grpcServer.GracefulStop()
   log.Info("server terminated", topic)
}
```





参考资料：



https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace

https://blog.gruntwork.io/delaying-shutdown-to-wait-for-pod-deletion-propagation-445f779a8304

https://stackoverflow.com/questions/55797865/behavior-of-server-gracefulstop-in-golang
