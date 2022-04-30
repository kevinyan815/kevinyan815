现在微服务架构盛行，很多以前的单体应用服务都被拆成了多个分布式的微服务，以解决应用系统发展壮大后的开发周期长、难以扩展、故障隔离等挑战。

不过技术领域有个谚语叫--没有银弹，这句话的意思其实跟现实生活中任何事都有利和弊两面一样，意思是告诉我们不要寄希望于用一个解决方案解决所有问题，引入新方案解决旧问题的同时，势必会引入新的问题。典型的比如，原本在单体应用里可以靠本地数据库的ACID 事务来保证数据一致性。但是微服务拆分后，就没那么简单了。

同理拆分成为服务后，一个业务逻辑的完成一般需要多个服务的协作才能完成，每个服务都会有自己的业务日志，那怎么把各个服务的业务日志串联起来，也会变难，今天我们就聊一下微服务的日志串联的方案。

在早前的文章[分布式链路跟踪中的traceid和spanid代表什么？](https://mp.weixin.qq.com/s/eKbFYwnH4vwgWm6_5sWs3w) 里我给大家介绍过 TraceId 和 SpanId 的概念

![图片](https://mmbiz.qpic.cn/mmbiz_png/z4pQ0O5h0f4uFOdytPQ2y2czD6Ac3BSfBg3cwia4F6QmqllnrlhxRys8bWST4iacGWBd2UMFeTKxH7u6Tiav0PytA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

- **trace 是请求在分布式系统中的整个链路视图**
- **span 则代表整个链路中不同服务内部的视图，span 组合在一起就是整个 trace 的视图**

在微服务的日志串联里，我们同样能使用这两个概念，通过 trace 串联出一个业务逻辑的所有业务日志，span 串联出在单个服务里的业务日志。

而单个微服务的日志串联的时候还有个挑战是怎么把数据库执行过程的一些日志也注入这些 traceid 和 spanid 打到业务日志里。下面我们就分别通过

- HTTP 服务间的日志追踪参数传递
- HTTP 和 RPC 服务间的追踪参数传递
- ORM 的日志中注入追踪参数

来简述一下微服务业务日志串联的思路。提前声明本文中给出的解决方案更多是 Go 技术栈的，其他语言的技术栈有些方案实现跟这里列举的稍有不同，尤其是 Java 一些开源库上比较容易实现的东西在 Go 这里并不简单。

>其实如果使用 APM 的话，是有比较统一的解决方案的，比如接入 Skywalking 就可以，不过还是有额外的学习成本以及需要引入外部系统组件的。

### HTTP 服务间的日志追踪参数传递

HTTP 服务间的追踪参数传递，主要是靠在全局的路由中间件来搞，我们可以在请求头里指定 TraceId 和 SpanId。当然如果是请求到达的第一个服务，则生成 TraceId 和 SpanId，加到Header 里往下传。

```go
type Middleware func(http.HandlerFunc) http.HandlerFunc

func withTrace() Middleware {
    // 创建中间件
    return func(f http.HandlerFunc) http.HandlerFunc {

      	return func(w http.ResponseWriter, r *http.Request) {
    				traceID := r.Header.Get("xx-tranceid")
						parentSpanID := r.Header.Get("xx-spanid")
		        spanID := genSpanID(r.RemoteAddr)
          	if traceID == "" {// traceID为空，证明是初始调用，让root span id == trace id
      					traceId = spanID
    				}
            // 把 追踪参数通过 Context 在服务内部处理中传递
          	ctx := context.WithValue(r.Context(), "trace-id", traceID)
          	ctx := context.WithValue(ctx, "pspan-id", parentSpanID)
						ctx := context.WithValue(ctx, "span-id", parentSpanID)
            r.WithContext(ctx)

            // 调用下一个中间件或者最终的handler处理程序
            f(w, r)
        }
    }
}
```

上面主要通过在中间件程序中，获取 Header 头里存储的追踪参数，把参数保存到请求的 Context 中在服务内部传递。上面的程序有几点需要说明：

- genSpanID 是根据远程客户端IP 生成唯一 spanId 的方法，生成方法只要保证哈希串唯一就行。
- 如果服务是请求的开始，在生成spanId 的时候，我们把它也设置成 traceId，这样就能通过 spanId == traceId 判断出当前的 Span 是请求的开端、即 root span。

接下来往下游服务发起请求的时候，我们需要在把 Context 里存放的追踪参数，放到 Header 里接着往下个 HTTP 服务传。

```go
func HttpGet(ctx context.Context url string, data string, timeout int64) (code int, content string, err error) {
	req, _ := http.NewRequest("GET", url, strings.NewReader(data))
	defer req.Body.Close()

	req.Header.Add("xx-trace-tid", ctx.Value("trace-id").(string))
	req.Header.Add("xx-trace-tid", ctx.Value("span-id").(string))
  
	client := &http.Client{Timeout: time.Duration(timeout) * time.Second}
	resp, error := client.Do(req)
}
```

### HTTP 和 RPC 服务间的追踪参数传递

上面咱们说的上下游服务都是 HTTP 服务的追踪参数传递，那如果是 HTTP 服务的下游是 RPC 服务呢？

其实跟发HTTP请求可以配置HTTP客户端携带的Header和Context一样，RPC客户端也支持类似功能。以 gRPC 服务为例，客户端调用RPC 方法时在可以携带的元数据里设置这些追踪参数。

```go
traceID := ctx.Value("trace-id").(string)
traceID := ctx.Value("trace-id").(string)
md := metadata.Pairs("xx-traceid", traceID, "xx-spanid", spanID)
// 新建一个有 metadata 的 context
ctx := metadata.NewOutgoingContext(context.Background(), md)
// 单向的 Unary RPC
response, err := client.SomeRPCMethod(ctx, someRequest)
```

RPC 的服务端的处理方法里，可以再通过 metadata 把元数据里存储的追踪参数取出来。

```go
func (s server) SomeRPCMethod(ctx context.Context, req *xx.someRequest) (reply *xx.SomeReply, err error) {

  remote, _ := peer.FromContext(ctx)
	remoteAddr := remote.Addr.String()
  // 生成本次请求在当前服务的 spanId
	spanID := utils.GenerateSpanID(remoteAddr)
  
  traceID, pSpanID := "", ""
	md, _ := metadata.FromIncomingContext(ctx)
	if arr := md["xx-tranceid"]; len(arr) > 0 {
		traceID = arr[0]
	}
	if arr := md["xx-spanid"]; len(arr) > 0 {
		pSpanID = arr[0]
	}
	return
}
```

有一个概念我们需要注意一下，代码里是把上游传过来的 spanId 作为本服务的 parentSpanId 的，本服务处理请求时候的 spanId 是需要重新生成的，生成规则在之前我们介绍过。

除了 HTTP 网关调用 RPC 服务外，处理请求时也经常出现 RPC 服务间的调用，那这种情况该怎么弄呢？

### RPC 服务间的追踪参数传递

其实跟 HTTP 服务调用 RPC 服务情况类似，如果上游也是 RPC 服务，那么则应该在接收到的上层元数据的基础上再附加的元数据。

```go
md := metadata.Pairs("xx-traceid", traceID, "xx-spanid", spanID)
mdOld, _ := metadata.FromIncomingContext(ctx)
md = metadata.Join(mdOld, md)
ctx = metadata.NewOutgoingContext(ctx, md)
```

当然如果我们每个客户端调用和RPC 服务方法里都这么搞一遍得类似，GRPC 里也有类似全局路由中间件的概念，叫拦截器，我们可以把追踪参数传递这部分逻辑封装在客户端和服务端的拦截器里。

>gRPC 拦截器的详细介绍请看我之前的文章 -- [gRPC生态里的中间件](https://mp.weixin.qq.com/s/kHW5t0bqLuJZUNNknBJnAA)

#### 客户端拦截器

```go
func UnaryClientInterceptor(ctx context.Context, ... , opts ...grpc.CallOption) error {
	md := metadata.Pairs("xx-traceid", traceID, "xx-spanid", spanID)
	mdOld, _ := metadata.FromIncomingContext(ctx)
	md = metadata.Join(mdOld, md)
	ctx = metadata.NewOutgoingContext(ctx, md)

	err := invoker(ctx, method, req, reply, cc, opts...)

	return err
}

// 连接服务器
conn, err := grpc.Dial(*address, grpc.WithInsecure(),grpc.WithUnaryInterceptor(UnaryClientInterceptor))
```

#### 服务端拦截器

```go
func UnaryServerInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
	remote, _ := peer.FromContext(ctx)
	remoteAddr := remote.Addr.String()
	spanID := utils.GenerateSpanID(remoteAddr)

	// set tracing span id
	traceID, pSpanID := "", ""
	md, _ := metadata.FromIncomingContext(ctx)
	if arr := md["xx-traceid"]; len(arr) > 0 {
		traceID = arr[0]
	}
	if arr := md["xx-spanid"]; len(arr) > 0 {
		pSpanID = arr[0]
	}
  // 把 这些ID再搞到 ctx 里，其他两个就省略了
  ctx := Context.WithValue(ctx, "traceId", traceId)
	resp, err = handler(ctx, req)
  
	return
}

```

### ORM 的日志中注入追踪参数

其实，如果你用的是 GORM 注入这个参数是最难的，如果你是 Java 程序员的话，可能会对比如阿里巴巴 的Druid 和 Hibernate 链接池加入类似 traceId 这种参数习以为常，但是 Go 的 GORM 库确实做不到，也有可能新版本可以，我用的还是老版本，其他 Go 的 ORM 库也不熟，知道的同学可以留言给我们普及一下。

GORM做不到在日志里加入追踪参数的原因就是这个GORM 的 logger 没有实现SetContext方法，所以除非修改源码中调用db.slog的地方，否则无能为力。

不过话也不能说死，之前介绍过一种使用函数调用栈实现 Goroutine Local Storage 的库 [jtolds/gls](https://github.com/jtolds/gls) ，只能通过它在外面封装一层来实现，并且还需要重新实现 GORM Logger 的打印日志的 Print 方法。

下面大家感受一下，GLS 库的使用，确实有点点怪，不过能过。

```go
func SetGls(traceID, pSpanID, spanID string, cb func()) {
	mgr.SetValues(gls.Values{traceIDKey: traceID, pSpanIDKey: pSpanID, spanIDKey: spanID}, cb)
}

gls.SetGls(traceID, pSpanID, spanID, func() {
		data, err =  findXXX(primaryKey)
})
```

重写 Logger 的我就简单贴贴，核心思路还是在记录SQL到日志的时候，从调用栈里把 traceId 和 spanId 取出来放一并加入到日志记录里。

```go
// 对Logger 注册 Print方法
func (l logger) Print(values ...interface{}) {
	if len(values) > 1 {
			// ...
			l.sqlLog(sql, args, duration, path.Base(source))

		} else {
			err := values[2]
			log.Error("source", source, "err", err)
		}
	}
}

func (l logger) sqlLog(sql string, args []interface{}, dur time.Duration, source string) {
	argsArr := make([]string, len(args))
	for k, v := range args {
		argsArr[k] = fmt.Sprintf("%v", v)
	}
	argsStr := strings.Join(argsArr, ",")
  
  spanId := gls.GetSpanId()
  traceId := gls.GetTraceId()
	//对于超时的,统一打warn日志
	if dur > (time.Millisecond * 500) {
		log.Warn("xx-traceid", traceId, "xx-spanid", spanId, "sql", sql, "args_detal", argsStr, "source", source)
	} else {
		log.Debug("xx-traceid", traceId, "xx-spanid", spanId, "sql", sql, "args_detal", argsStr, "source", source)
	}
}
```

通过调用栈获取 spanId 和 traceId 的是类似这样的方法，由 GLS 那个库提供的方法封装的。

```go
//  Get spanID 用于Goroutine的链路追踪
func GetSpanID() (spanID string) {
	span, ok := mgr.GetValue(spanIDKey)
	if ok {
		spanID = span.(string)
	}
	return
}
```

日志打印的话，也是对超过 500 毫秒的SQL执行进行 Warn 级别日志的打印，方便线上环境分析问题，而其他的SQL执行记录，因为使用了 Debug 日志级别只会在测试环境上显示。

### 总结

用分布式链路追踪参数串联起整个服务请求的业务日志，在线上的分布式环境中是非常有必要的，其实上面只是简单阐述了一些思路，只有把日志搞的足够好，上下文信息足够多才会能高效地定位出线上问题。感觉这部分细节太多，想用一篇文章阐述明白非常困难。

而且还有一点就是日志错误级别的选择也非常有讲究，如果本该用Debug的地方，用了 Info 级别，那线上日志就会出现非常多的干扰项。

细节的地方就属于实践的时候才能把控了，希望这篇文章能给你个主旨思路。如果希望了解线上的具体实践方案，本人可以针对企业接受有偿的技术咨询。
