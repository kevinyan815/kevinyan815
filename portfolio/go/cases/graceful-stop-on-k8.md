**平滑关闭**和**服务摘流**是保证部署了多节点的应用能够持续稳定对外提供服务的两个重要手段，平滑关闭保证了应用节点在关闭之前处理完已接收到的请求，用`net/http`库提供的 `http.ShutDown`可以平滑关停HTTP 服务，下面的内容会说明一下`gRPC`分布式服务的平滑关停方法。

应用在进入平滑关闭阶段后拒绝为新进来的流量提供服务，如果此时继续有新流量访问而来，势必会让发送请求的客户端感知到服务的断开，所以在平滑关闭应用前我们还要对应用节点做摘流操作，保证网关不会再把新流量分发到要关闭的应用节点上才行。

如果服务部署在云主机上，摘流只需要运维人员从负载均衡上把机器节点的IP拿掉，待应用重启或者更新完毕后再将机器节点的IP挂回负载均衡上即可。但是人工摘流这对 Kubernetes 这样的进行多节点集群资源调度的系统显然是不可能的，在这里还会介绍如何让 Kubernetes 系统自动为我们即将关停的应用节点完成摘流操作。

## 平滑关闭

在这个章节里除了介绍 gRPC框架平滑关闭应用的方法外还会介绍一下Kubernetes集群里完成Pod删除的整个生命周期，因为如果我们的gRPC服务部署在Kubernetes集群里的话，服务的平滑关闭和摘流都会依赖这个Pod 删除的生命周期，或者叫Pod关闭序列来实现。如果在下面内容里看到我一会儿说 「Pod 的关闭序列」，一会儿说「 Pod 删除的生命周期」请一定要记住他们是一个东西的两种表述方式而已。

### gRPC的gracefulStop

gRPC 框架使用的通信协议是`HTTP2`，`HTTP2`对于连接关闭使用 `goaway` 帧信号（类型是0x7，用于启动连接关闭或发出严重错误状态信号）。`goaway` 允许服务端点正常停止接受新的流量，同时仍然完成对先前已建立的流的处理。

Go 语言版本的 gRPC Server 提供了两个退出方法`Stop`和`GracefulStop`，光看名字就知道后面这个是实现平滑关闭用的。`GracefulStop` 方法里首先会关闭服务监听，这样就无法再建立新的请求，然后会遍历所有的当前连接发送`goaway`帧信号。`serveWG.Wait()`会等待所有`handleRawConn`协程的退出（在gRPC Server里每个新连接都会创建一个`handleRawConn`协程，并且增加`WaitGroup`的计数器的计数）。

```go
func (s *Server) GracefulStop() {
    s.mu.Lock()
    ...

    // 关闭监听，不再接收新的连接请求
    for lis := range s.lis {
        lis.Close()
    }

    s.lis = nil
    if !s.drain {
        for st := range s.conns {
            // 给所有的连接发布goaway信号
            st.Drain()  
        }
        s.drain = true
    }


    // 等待所有handleRawConn协程退出，每个请求都是一个goroutine，通过WaitGroup控制.
    s.serveWG.Wait()

    // 当还有空闲连接时，需要等待。在退出serveStreams逻辑时，会进行Broadcast唤醒。只要有一个客户端退出就会触发removeConn继而进行唤醒。
    for len(s.conns) != 0 {
        s.cv.Wait()
    }
...
```

Stop 方法相对于 GracefulStop 来说少了给连接发送 goaway 帧信号和等待连接退出的逻辑，这里就不再做过多介绍了。

### 应用监听OS信号，启动平滑关闭

知道 gRPC框架提供的服务平滑关闭的方法后，与HTTP服务的平滑关闭一样，我们的应用要能接收到`OS`发来的`TERM` 、`Interrupt`之类的信号，然后主动去触发调用`GracefulStop`进行服务的平滑关闭，当然调用平滑关闭前我们还可以做一些其他应用内的首尾工作，比如应用使用Etcd实现的服务注册，那么这里我建议要先去主动的把节点的IP对应的Key从Etcd上注销掉，如果Key不能及时过期，那么客户端做负载均衡时没有收到这个节点IP删除的通知就仍有可能会往要关闭的端点上发请求。

下面是gRPC服务启动后监听 `OS` 发来的断开信号时开始平滑关闭的方法，演示的代码只是一些伪代码，不过真实度已经很高了，实际应用时可以直接往这个代码模板里套用自己的方法。

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
   // TODO 做一些项目自己的清理工作
   DoSomeCleanJob()
   // 平滑关闭服务
   grpcServer.GracefulStop()
}
```

## Kubernetes Pod关闭时要经历的生命周期

Kubernetes 在应用需要更新、节点需要升级维护、节点资源耗尽的时候都会删除Pod再重新创建和调度Pod，Pod 在Kubernetes集群中被删除前会经历以下生命周期：

1. Pod 状态被标记为`Terminating`， 此时 Pod 开始停止接收新流量。
2. Pod 的 preStop 钩子会被执行，在钩子里我们可以设置要执行的命令或者要发送的HTTP请求，大部分应用可以处理OS发来的TERM中断信号，但是如果应用依赖了不受自主控制的外部系统，可以通过钩子里发送请求完成注销之类的动作，后面介绍的服务摘流也会用到 preStop钩子。
3. Kubernetes向Pod 发送 SIGTERM 信号。
4. Kubernetes 会默认等待30秒让Pod完成关闭，如果需要等待超过30秒应用才能正常退出，可以使用`terminationGracePeriodSeconds` 在Deployment里自己配置平滑关闭Pod Kubernetes要等待的时间。需要注意的是上面说的 preStop 钩子的执行和 SIGTERM 信号的发送都包含在这个时间里，**如果应用早于这个时间关闭会立即进入生命周期的下个阶段**。
5. Kubernetes向应用发送 SIGKILL 信号，然后删除Pod。

上面那个 gRPC 服务，部署在Kubernetes集群里后，假如遇到节点升级或者其他要关闭某个节点上Pod的情况，应用就可以收到Kubernetes 向Pod发送的TERM信号，主动完成平滑关闭服务的操作。


## Kubernetes服务摘流

说起Kubernetes的服务摘流，我们就不得不再把Kubernetes里的Pod和Service这种资源的概念再简单捋一遍。我们的应用服务运行在容器里，容器被 Kubernetes 封装在Pod里，Pod里可以有多个容器，但只能有一个运行主进程的主容器，其他容器都是辅助用的，即Pod 支持的（sidecar）边车模式。

Pod 自身每次重建IP都会变，且Pod自身的IP只能在节点内访问，所以 Kubernetes 就用了一种叫做的 Service 的控制器来管控一组Pod，为它们向外部提供统一的访问方式，Service 它通过selector 指定 Pod的标签来把符合条件的Pod 都加到它的服务端点列表里。

所以我们应用启动后向注册中心注册的IP，不是应用所在的Pod的IP，而是上层 NodePort Service 的 IP(其实就是 Pod 所在节点的IP)，访问它的时候会自动做负载均衡，随机把流量路由给Service后面挂的Pod。


Service 本身其实是会为Pod做探活和摘流的，但是如果你的应用的访问量足够大，Service的摘流有时候并不及时，在Pod 关闭的时候还是会有新流量进来。这就导致了在重启服务，或者是Kubernetes集群内部有一个节点升级、重启之类的动作，节点上的Pod被调度到其他节点上时，客户端还是能感知到闪断。这其实是一个很大的问题，因为 Kubernetes 集群内部做资源重新调度，切换新节点之类的动作还是挺常见的。经过翻看Kubernetes相关的资料和一些社区里的讨论，我们终于找到了Service摘流慢的原因（其实没费多大劲，Github和Kubernetes In Action 那本书里都有说过这个问题）。

原因是 Kubernetes 删除 Pod 前会向 Kubernetes 集群内广播 Pod 的删除事件，会同时有几个子系统接收广播处理事件，其中就包括：

- 要删除的 Pod 所在节点上的`Kubelet`接收到事件后会开启上面介绍的Pod 关闭的生命周期，Pod拒绝伺服新流量等待生命周期内的动作执行完成后被删除。
- Pod 的 Service 控制器收到事件后会把要关闭的 Pod 从服务端点列表里移除出去。

上面动作会同时并行发生，这就导致了有可能Pod已经进入关闭序列了，但是Service那里还没有做完摘流，Service还是有可能会把新来的流量路由给要关闭的Pod上。

社区里和Kubernetes In Action 这本书里针对这个问题，都给出了一个相同的解决方案。利用 Pod 关闭生命周期里的preStop 钩子，让其执行 sleep 命令休眠5~10秒，通过延迟关闭Pod来让Service先完成摘流，**preStop的执行并不影响Pod内应用继续处理现存请求**。

在 preStop 钩子里引入延迟的方法，可以参考下面的配置文件片段。

```yaml
containers:
  - args:
  - /bin/bash
  - -c
  - /go-big-app
  ... 
  # 下面的preStop钩子里引入10秒延迟
  lifecycle:
    preStop:
      exec:
        command:
          - sh
          - -c
          - sleep 10
```

这样就让并行执行的摘流和平滑关闭动作在时间线上尽量错开了，也就不会出现Service摘流可能会有延迟的问题了。

> 关于这个问题详细的描述和解决方案可以参考我前面翻译的文章「[借助 Pod 删除事件的传播实现 Pod 摘流](https://mp.weixin.qq.com/s?__biz=MzUzNTY5MzU2MA==&mid=2247487210&idx=1&sn=d73ca74b2e1bc1f969ac094ea44a5b28&chksm=fa80df7dcdf7566bce2f91da94cf33222e8b1bdea147f69403c4139d520641f3c09acc62c927&token=2033333242&lang=zh_CN&scene=21#wechat_redirect)」，里面有详细的图文解释来说明这个问题的由来和解决办法。

## 总结

这里讲的这些内容算是我在做服务高可用保证时总结出来的一些经验吧，里面介绍的知识点，单看每个在它自己的领域其实都不是什么难掌握的东西，用Go开发的基本上都会用`signal.Notify`接收OS信号完成应用的平滑关闭，做 Kubernetes 运维的对后面那些概念和解决问题的方法应该也都是轻车熟路，但是作为研发如果我们能"跨界"多学一些像Kubernetes这样的在生态里和程序开发紧密结合的知识，靠研发主动推动运维配合我们解决这些问题，所获得的经验和成就感还是挺不一样。
