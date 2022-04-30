## gRPC的服务发现和负载均衡
gRPC的负载均衡不同于`Nginx`、`Lvs`或者`F5`这些服务端的负载均衡策略，gRPC采用的是客户端实现的负载均衡。什么意思呢，对于使用服务端负载均衡的系统，客户端会首先访问负载均衡的域名/IP，再由负载均衡按照策略分发请求到后端具体某个服务节点上。而对于客户端的负载均衡则是，客户端从可用的后端服务节点列表中根据自己的负载均衡策略选择一个节点直连后端服务器。


`Etcd`软件包的`naming`组件里提供了一个命名解析器（naming resolver）结合`gRPC`本身自带的`RoundRobin` 轮询调度负载均衡器，让使用者能方便地搭建起一套服务注册/发现和负载均衡体系。如果轮询调度满足不了调度需求或者不想使用`Etcd`作为服务的注册中心和命名解析器的话，可以通过写代码实现`gRPC`定义的`Resolver`和`Balancer`接口来满足系统的自定义需求。

>本文引用的源码对应的版本为：gRPC v1.2.x、  Etcd  v3.3
>

## gRPC服务注册发现

先来简单的说明一下用`Etcd`实现服务注册和发现的原理。服务注册和发现这个流程可以用下面这个示意图简单描述出来：

![](https://cdn.learnku.com/uploads/images/202010/23/6964/tVBRqjyHsp.jpg!large)

上图的服务A包含了两个节点，服务在节点上启动后，会以包含服务名加节点IP的唯一标识作为Key（比如/service/a/114.128.45.117），服务节点IP和端口信息作为值存储到`Etcd`上。这些Key都是带租约的Key，需要我们的服务自己去定期续租，一旦服务节点本身宕掉，比如node2上的服务宕掉，无法完成续租后，那么它对应的Key：/service/a/114.128.45.117 就会过期，客户端也就无法再从Etcd上获取到这个服务节点的信息了。


与此同时客户端也会利用`Etcd`的`Watch`功能监听以`/servive/a`为前缀的所有Key的变化，如果有新增或者删除节点Key的事件发生`Etcd`都会通过`WatchChan`发送给客户端，`WatchChan`在编程语言上的实现就是`Go`的`Channel`。

### 服务注册

关于`Etcd`的服务注册，官方提供的软件包里并没有提供统一的注册函数供调用。那么我们在新增服务节点后怎么把节点的信息存储到`Etcd`上并通知给命名解析器呢？在Etcd源码包的naming/grpc.go里可以发现提供了一个`Update`方法，这个`Update`既能执行添加也能执行删除操作：

```go
func (gr *GRPCResolver) Update(ctx context.Context, target string, nm naming.Update, opts ...etcd.OpOption) (err error) {
	switch nm.Op {
	case naming.Add:
		var v []byte
		if v, err = json.Marshal(nm); err != nil {
			return status.Error(codes.InvalidArgument, err.Error())
		}
		_, err = gr.Client.KV.Put(ctx, target+"/"+nm.Addr, string(v), opts...)
	case naming.Delete:
		_, err = gr.Client.Delete(ctx, target+"/"+nm.Addr, opts...)
	default:
		return status.Error(codes.InvalidArgument, "naming: bad naming op")
	}
	return err
}
```

服务在启动完成后可以通过`Update`方法把自己的服务地址和端口`Put`到自定义的target为前缀的key里，针对上面图示里的例子，变量target就应该是我们定义的服务名/service/a。一般在具体实践里都是自己根据系统的需求封装`Update`方法完成服务注册，以及服务节点Key在Etcd上的定期续租，这块每个公司的实践都不一样，我就不放具体的代码了，一般续租都是通过`Etcd`租约里的`KeepAlive`方法实现的（Lease.KeepAlive）。

### 服务发现

在注册完新节点、或者是原来的节点停掉后，客户端是怎么知道的呢？这块就需要命名解析器Resolver来帮助实现了，Resolver的作用可以理解为从一个字符串映射到一组IP端口等信息。

gRPC对Resolver的接口定义如下：
```go
type Resolver interface {
	// Resolve creates a Watcher for target.
	Resolve(target string) (Watcher, error)
}
```

命名解析器的Resolve方法会返回一个Watcher，这个Watcher可以监听命名解析器发来的target（类似上面例子里说的与服务名相对应的Key）对应的后端服务器地址信息变化，通知Balancer对自己维护的地址进行动态地增删。

Watcher接口的定义如下：

```go
//源码地址 https://github.com/grpc/grpc-go/blob/v1.2.x/naming/naming.go
type Watcher interface {
	Next() ([]*Update, error)
	// Close closes the Watcher.
	Close()
}
```



Etcd为这两个接口都提供了实现：

```go
// 源码地址：https://github.com/etcd-io/etcd/blob/release-3.3/clientv3/naming/grpc.go

// GRPCResolver 实现了grpc的naming.Resolver接口
type GRPCResolver struct {
	// Client is an initialized etcd client.
	Client *etcd.Client
}

func (gr *GRPCResolver) Resolve(target string) (naming.Watcher, error) {
	ctx, cancel := context.WithCancel(context.Background())
	w := &gRPCWatcher{c: gr.Client, target: target + "/", ctx: ctx, cancel: cancel}
	return w, nil
}

// 实现了grpc的naming.Watcher接口
type gRPCWatcher struct {
	c      *etcd.Client
	target string
	ctx    context.Context
	cancel context.CancelFunc
	wch    etcd.WatchChan
	err    error
}

func (gw *gRPCWatcher) Next() ([]*naming.Update, error) {
	if gw.wch == nil {
		// first Next() returns all addresses
		return gw.firstNext()
	}

	// process new events on target/*
	wr, ok := <-gw.wch
	if !ok {
  ...
	updates := make([]*naming.Update, 0, len(wr.Events))
	for _, e := range wr.Events {
		var jupdate naming.Update
		var err error
		switch e.Type {
		case etcd.EventTypePut:
			err = json.Unmarshal(e.Kv.Value, &jupdate)
			jupdate.Op = naming.Add
		case etcd.EventTypeDelete:
			err = json.Unmarshal(e.PrevKv.Value, &jupdate)
			jupdate.Op = naming.Delete
		default:
			continue
		}
		if err == nil {
			updates = append(updates, &jupdate)
		}
	}
	return updates, nil
}
  
func (gw *gRPCWatcher) firstNext() ([]*naming.Update, error) {
  // 获取前缀为gw.target的所有Key的值，放到现有数组里
	resp, err := gw.c.Get(gw.ctx, gw.target, etcd.WithPrefix(), etcd.WithSerializable())
	if gw.err = err; err != nil {
		return nil, err
	}

	updates := make([]*naming.Update, 0, len(resp.Kvs))
	for _, kv := range resp.Kvs {
		var jupdate naming.Update
		if err := json.Unmarshal(kv.Value, &jupdate); err != nil {
			continue
		}
		updates = append(updates, &jupdate)
	}

	opts := []etcd.OpOption{etcd.WithRev(resp.Header.Revision + 1), etcd.WithPrefix(), etcd.WithPrevKV()}
  // watch 监听这些Key的变化，包括前缀相同的新Key的加入
	gw.wch = gw.c.Watch(gw.ctx, gw.target, opts...)
	return updates, nil
}

func (gw *gRPCWatcher) Close() { gw.cancel() }
```

这部分`GRPCResolver`和`gRPCWatcher`类型的每个方法的功能和起到的作用都和`RoundRobin`这个gRPC Balancer结合地比较紧密，我准备放到下面和负载均衡的源码实现一起说明。



## 负载均衡

首先我们来看一下gRPC对负载均衡的接口定义：

```
type Balancer interface {

	Start(target string, config BalancerConfig) error

	Up(addr Address) (down func(error))

	Get(ctx context.Context, opts BalancerGetOptions) (addr Address, put func(), err error)

	Notify() <-chan []Address
	// Close shuts down the balancer.
	Close() error
}
```

在gRPC 客户端与服务端之间建立连接时调用的`Dail`方法里可以用`WithBalancer`方法在`DiaplOption`里指定负载均衡组件：

```go
  client, err := etcd.Client()
	...
	resolver := &naming.GRPCResolver{Client: client}
	b := grpc.RoundRobin(resolver)
	opt0 := grpc.WithBalancer(b)

  grpc.Dial(target, opt0 , opt1, ...) // 后面省略了
```
上面的例子使用了gRPC自带的Balancer实现RoundRobin，RoundRobin除了实现了Balancer接口外自己内置了Resolver用来从名字获取其后绑定的IP信息以及服务的更新事件（增加删除服务节点这些事件） 。上面的例子里给RoundRobin指定了`Etcd`提供的`name.GRPCResolver`做为它的命名解析器，这个命名解析器就是上一节说的`Etcd`软件包里提供的gRPC`naming.Resolver`接口实现。

## RoundRobin

下面我们研究一下gRPC包里提供的`RoundRobin`代码实现，主要关注负载均衡和利用Resolver进行服务发现及节点更新这两个功能的代码实现原理

`RoundRobin`结构体定义如下：

```go
// 源码在：https://github.com/grpc/grpc-go/blob/v1.2.x/balancer.go
type roundRobin struct {
	r      naming.Resolver
	w      naming.Watcher
	addrs  []*addrInfo // 客户端可以尝试连接的所有地址
	mu     sync.Mutex
	addrCh chan []Address // 用于通知gRPC内部的，客户端可连接地址的信道
	next   int            // index of the next address to return for Get()
	waitCh chan struct{}  // the channel to block when there is no connected address available
	done   bool           // The Balancer is closed.
}
```
- r是命名解析器，可以定义自己的命名解析器，如Etcd命名解析器。如果r为nil，那么Dial中参数target将直接作为可请求地址添加到addrs中。
- w是命名解析器Resolve方法返回的watcher，该watcher可以监听命名解析器发来的地址信息变化，通知roundRobin对addrs中的地址进行动态的增删。
- addrs是从命名解析器获取地址信息数组，数组中每个地址不仅有地址信息，还有gRPC与该地址是否已经创建了ready状态的连接的标记。
- addrCh是地址数组的Channel，该Channel会在**每次**命名解析器发来地址信息变化后，将**所有**地址更新通知到gRPC内部的lbWatcher，lbWatcher是统一管理地址连接状态的协程，负责新地址的连接与被删除地址的关闭操作。
- next是roundRobin的Index，即轮询调度遍历到addrs数组中的哪个位置了。
- waitCh是当addrs中地址为空时，grpc调用`Get()`方法希望获取到一个到target的连接，如果设置了gRPC的failfast为false，那么`Get()`方法会阻塞在此Channel上，直到有ready的连接。



### 启动RoundRobin

启动RoundRobin就是实现Balancer接口的Start方法，该方法是由一开始通过`grpc.WithBalancer`把负载均衡器指定给的`BalancerWrapperBuilder`在创建`BalancerWrapper`时触发的：

```
func (bwb *balancerWrapperBuilder) Build(cc balancer.ClientConn, opts balancer.BuildOptions) balancer.Balancer {
  // 这里触发Balancer的Start方法
	bwb.b.Start(opts.Target.Endpoint, BalancerConfig{
		DialCreds: opts.DialCreds,
		Dialer:    opts.Dialer,
	})
	_, pickfirst := bwb.b.(*pickFirst)
	bw := &balancerWrapper{
		......
	}
	cc.UpdateBalancerState(connectivity.Idle, bw)
	go bw.lbWatcher() // 监听Balancer 通知过来的地址变化
	return bw
}
```



Start方法其主要功能就是通过RoundRobin的命名解析器的`Resolve`方法拿到监听命名解析器后端变化的`Watcher`。与此同时还会新建一个`addrChan`用于向`gRPC`内部的`lbWatcher`推送`Watcher`监听到的地址变化。

```go
func (rr *roundRobin) Start(target string, config BalancerConfig) error {
    rr.mu.Lock()
    defer rr.mu.Unlock()
    if rr.done {
        return ErrClientConnClosing
    }
    if rr.r == nil {
        // 如果没有解析器，那么直接将target加入addrs地址数组
        rr.addrs = append(rr.addrs, &addrInfo{addr: Address{Addr: target}})
        return nil
    }
    // Resolve接口会返回一个watcher，watcher可以监听解析器的地址变化
    w, err := rr.r.Resolve(target)
    if err != nil {
        return err
    }
    rr.w = w
    // 创建一个channel，当watcher监听到地址变化时，通知grpc内部lbWatcher去连接该地址
    rr.addrCh = make(chan []Address, 1)
    // go 创建新协程监听watcher，监听地址变化。
    go func() {
        for {
            if err := rr.watchAddrUpdates(); err != nil {
                return
            }
        }
    }()
    return nil
}
```

创建完addrCh后在Start方法最后会开启一个goroutine，这个goroutine会不停地循环调用`watchAddrUpdates`查询是否有命名解析器的`Watcher`传递过来的更新。

### 监听服务端地址的更新

在`watchAddrUpdates`方法里就是通过上面Start方法里创建的Resolver Watcher的Next方法来监听`Etcd`上后端服务节点的更新，这个Watcher的实现就是上面服务发现章节里说的Etcd软件包里提供的`gRPCWatcher`类型，它的Next方法里会去通过监听Etcd上由服务名组成的Key的变化，然后在这里把这些信息传递给上面`Start`方法里创建好的`addrChan`通道。

```go
func (rr *roundRobin) watchAddrUpdates() error {
    // watcher的next方法会阻塞，直至有地址变化信息过来，updates即为变化信息
    updates, err := rr.w.Next()
    if err != nil {
        return err
    }
    // 对于addrs地址数组的操作，显然是要加锁的，因为有多个goroutine在同时操作
    rr.mu.Lock()
    defer rr.mu.Unlock()
    for _, update := range updates {
        addr := Address{
            Addr:     update.Addr,
            Metadata: update.Metadata,
        }
        switch update.Op {
        case naming.Add:
        //对于新增类型的地址，注意这里不会重复添加。
            var exist bool
            for _, v := range rr.addrs {
                if addr == v.addr {
                    exist = true
                    break
                }
            }
            if exist {
                continue
            }
            rr.addrs = append(rr.addrs, &addrInfo{addr: addr})
        case naming.Delete:
        //对于删除的地址，直接在addrs中删除就行了
            for i, v := range rr.addrs {
                if addr == v.addr {
                    copy(rr.addrs[i:], rr.addrs[i+1:])
                    rr.addrs = rr.addrs[:len(rr.addrs)-1]
                    break
                }
            }
        default:
            grpclog.Errorln("Unknown update.Op ", update.Op)
        }
    }
    // 这里复制了整个addrs地址数组，然后丢到addrCh channel中通知grpc内部lbWatcher，
    // lbWatcher会关闭删除的地址，连接新增的地址。
    // 连接ready后会有专门的goroutine调用Up方法修改addrs中地址的状态。
    open := make([]Address, len(rr.addrs))
    for i, v := range rr.addrs {
        open[i] = v.addr
    }
    if rr.done {
        return ErrClientConnClosing
    }
    select {
    case <-rr.addrCh:
    default:
    }
    rr.addrCh <- open
    return nil
}
```

### 建立连接

Up方法是gRPC内部负载均衡的`watcher`调用的，该`watcher`会读全局的连接状态队列，改变RoundRobin维护的连接列表的里连接的状态 （会有单独的goroutine向目标服务发起连接尝试，尝试成功后才会把连接对象的连接状态改为connected），如果是已连接状态的连接 ，会调用Up方法来改变addrs地址数组中该地址的状态为**已连接**。

```go
func (rr *roundRobin) Up(addr Address) func(error) {
    rr.mu.Lock()
    defer rr.mu.Unlock()
    var cnt int
    //将地址数组中的addr置为已连接状态，这样这个地址就可以被client使用了。
    for _, a := range rr.addrs {
        if a.addr == addr {
            if a.connected {
                return nil
            }
            a.connected = true
        }
        if a.connected {
            cnt++
        }
    }
    // 当有一个可用地址时，之前可能是0个，可能要很多client阻塞在获取连接地址上，这里通知所有的client有可用连接啦。
    // 为什么只等于1时通知？因为可用地址数量>1时，client是不会阻塞的。
    if cnt == 1 && rr.waitCh != nil {
        close(rr.waitCh)
        rr.waitCh = nil
    }
    //返回禁用该地址的方法
    return func(err error) {
        rr.down(addr, err)
    }
}
```

### 关闭连接

关闭连接使用的是Down方法，这个方法就简单, 直接找到addr置为不可用就行了。

```go
func (rr *roundRobin) down(addr Address, err error) {
    rr.mu.Lock()
    defer rr.mu.Unlock()
    for _, a := range rr.addrs {
        if addr == a.addr {
            a.connected = false
            break
        }
    }
}
```

### 客户端获取连接

客户端在调用`gRPC`具体`Method`的`Invoke`方法里，会去`RoundRobin`的连接池addrs里获取连接，如果addrs为空，或者addrs里的地址都不可用，`Get()`方法会返回错误。但是如果设置了`failfast = false`，`Get()`方法会阻塞在`waitCh`这个通道上，直至`Up`方法给到通知，然后轮询调度可用的地址。

```go
func (rr *roundRobin) Get(ctx context.Context, opts BalancerGetOptions) (addr Address, put func(), err error) {
    var ch chan struct{}
    rr.mu.Lock()
    if rr.done {
        rr.mu.Unlock()
        err = ErrClientConnClosing
        return
    }
 
    if len(rr.addrs) > 0 {
        // addrs的长度可能变化，如果next值超出了，就置为0，从头开始调度。
        if rr.next >= len(rr.addrs) {
            rr.next = 0
        }
        next := rr.next
        //遍历整个addrs数组，直到选出一个可用的地址
        for {
            a := rr.addrs[next]
            // next值加一，当然是循环的，到len(addrs)后，变为0
            next = (next + 1) % len(rr.addrs)
            if a.connected {
                addr = a.addr
                rr.next = next
                rr.mu.Unlock()
                return
            }
            if next == rr.next {
                // 遍历完一圈了，还没找到，走下面逻辑
                break
            }
        }
    }
    if !opts.BlockingWait { //如果是非阻塞模式，如果没有可用地址，那么报错
        if len(rr.addrs) == 0 {
            rr.mu.Unlock()
            err = status.Errorf(codes.Unavailable, "there is no address available")
            return
        }
        // Returns the next addr on rr.addrs for failfast RPCs.
        addr = rr.addrs[rr.next].addr
        rr.next++
        rr.mu.Unlock()
        return
    }
    // Wait on rr.waitCh for non-failfast RPCs.
    // 如果是阻塞模式，那么需要阻塞在waitCh上，直到Up方法给通知
    if rr.waitCh == nil {
        ch = make(chan struct{})
        rr.waitCh = ch
    } else {
        ch = rr.waitCh
    }
    rr.mu.Unlock()
    for {
        select {
        case <-ctx.Done():
            err = ctx.Err()
            return
        case <-ch:
            rr.mu.Lock()
            if rr.done {
                rr.mu.Unlock()
                err = ErrClientConnClosing
                return
            }
 
            if len(rr.addrs) > 0 {
                if rr.next >= len(rr.addrs) {
                    rr.next = 0
                }
                next := rr.next
                for {
                    a := rr.addrs[next]
                    next = (next + 1) % len(rr.addrs)
                    if a.connected {
                        addr = a.addr
                        rr.next = next
                        rr.mu.Unlock()
                        return
                    }
                    if next == rr.next {
                        // 遍历完一圈了，还没找到，可能刚Up的地址被down掉了，重新等待。
                        break
                    }
                }
            }
            // The newly added addr got removed by Down() again.
            if rr.waitCh == nil {
                ch = make(chan struct{})
                rr.waitCh = ch
            } else {
                ch = rr.waitCh
            }
            rr.mu.Unlock()
        }
    }
}
```

## 总结

整个`gRPC`基于`Etcd`实现服务注册/发现以及负载均衡的流程和关键的源码实现就梳理完了，其实源码实现的细节远比我这里列举的要复杂，希望能通过文字记录下一学习和实践gRPC的负载均衡和服务解析时的一些关键路径。另外需要注意的是本文里使用的是gRPC v1.2.x的代码，在1.3版本后官方包重新调整了目录和包名，与本文里列举的源码以及Balancer的使用上都会有些出入，不过原理还是大致一样的，只不过每一版都一直在此基础上演进。


