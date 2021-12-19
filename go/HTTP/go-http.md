# HTTP服务

## demo

共有两种启动方式

- ### 方式一
  
  ```go
  package main
  
  import (
  	"io"
  	"log"
  	"net/http"
  )
  
  // 请求处理函数
  func handler(w http.ResponseWriter, r *http.Request) {
  	_, _ = io.WriteString(w, "hello, world!\n")
  }
  
  func main() {
  	// 注册路由
  	http.HandleFunc("/", handler)
  	// 创建监听服务
  	err := http.ListenAndServe(":8080", nil)
  	if err != nil {
  		log.Fatal("ListenAndServe: ", err)
  	}
  }
  ```
  
- ### 方式二
  
  ```go
  package main
  
  import (
  	"fmt"
  	"net/http"
  )
  
  type HelloHandlerStruct struct {
  	str string
  }
  
  func (handler *HelloHandlerStruct) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  	fmt.Fprintf(w, handler.str)
  }
  
  func main() {
  	http.Handle("/", &HelloHandlerStruct{str: "Hello World"})
  	http.ListenAndServe(":8080", nil)
  }
  ```

## 内部实现

第一种方式通过调用`http.HandleFunc`函数来进行路由注册，绑定路由和handler，其中函数的第二个参数为`func(ResponseWriter, *Request)`类型的函数。

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

第二种方式通过调用`http.Handle`进行绑定，函数第二个参数为一个`Handler`类型的`interface`。

```go
func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }
```

`Handler`定义如下，因此只要传入一个实现了`ServeHTTP`方法的`strut`即可。

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

两种方式的共同点在于路由注册函数内部调用的都是`DefaultServeMux`这个处理HTTP请求的多路复用器进行路由注册，这是一个go默认的`HTTP ServeMux`，结构如下

```go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}
```

也正是因为`http`包提供了默认的的`ServeMux`负责处理路由绑定，所以上述两种方式最后调用的`http.ListenAndServe`函数的第二个参数为`nil`，即不指定时用的就是默认的`DefaultServeMux`，当然也可以手动显式声明一个`ServeMux`类型的变量。



进一步分析可以发现第一种方式虽然传递的是一个`func`，但会对这个`func`进行进一步的处理

```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
  // 调用真正的mux进行处理
	mux.Handle(pattern, HandlerFunc(handler))
}
```

可以看到`mux.Handle`的第二个参数`handler`转换成了`HandlerFunc`，而由于`HandlerFunc`实现了`ServeHTTP`接口，故而已经变成了一个可以处理请求的`Handler`，所以两种方式是殊途同归的。

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

# 路由注册

从demo的`HandleFunc/Handle`函数入口，分析路由注册过程，从以上分析可以看出，路由注册主要通过`ServeMux`进行，server端对外暴露的`HandleFunc/Handle`函数主要的处理逻辑都是最终通过`ServeMux`这个`strut`来实现的，接下来深入分析`ServeMux`的工作逻辑

## Handler函数

注册路由时最终调用的就是`ServeMux.Handle`函数，代码如下

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

  // 三层校验，pattern、handle不能为空且不能重复注册pattern
	if pattern == "" {
		panic("http: invalid pattern")
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
  // 将handler和pattern封装成muxEntry
	e := muxEntry{h: handler, pattern: pattern}
  // 将pattern和muxEntry的映射存入mux.m字段，方便后续快速通过pattern查找对应的handler
	mux.m[pattern] = e
  // 通过appendSorted函数将所有pattern按照长度从大到小排序，为的是按序依次从长到端短匹配路由
	if pattern[len(pattern)-1] == '/' {
		mux.es = appendSorted(mux.es, e)
	}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}
```

函数的主要逻辑就是先进行一些必要的前置条件检查，然后将外部传入的handler和pattern封装在自己的私有字段中，方便后续根据pattern查找对应的handler进行快速处理

# 启动服务

路由注册完成调用`http.ListenAndServe`方法启动一个http server，该方法主要启动一个`Server`实例然后调用`server.ListenAndServe()`监听指定端口的TCP连接并进行处理

```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

## Server

`Server`定义了包含HTTP服务running的参数，包括了地址、Handler、tls等基本信息，超时时间，头部限制自己数，keepalive和shutdown处理函数等参数

```go
type Server struct {
  // 监听的TCP连接地址，默认为80端口
	Addr string

  // 默认为http.DefaultServeMux
	Handler Handler 

  // TLS配置
	TLSConfig *tls.Config

  // 一些读写操作的超时时间
	ReadTimeout time.Duration

	ReadHeaderTimeout time.Duration

	WriteTimeout time.Duration

	IdleTimeout time.Duration

	// MaxHeaderBytes controls the maximum number of bytes the
	// server will read parsing the request header's keys and
	// values, including the request line. It does not limit the
	// size of the request body.
	// If zero, DefaultMaxHeaderBytes is used.
	MaxHeaderBytes int

	// TLSNextProto optionally specifies a function to take over
	// ownership of the provided TLS connection when an ALPN
	// protocol upgrade has occurred. The map key is the protocol
	// name negotiated. The Handler argument should be used to
	// handle HTTP requests and will initialize the Request's TLS
	// and RemoteAddr if not already set. The connection is
	// automatically closed when the function returns.
	// If TLSNextProto is not nil, HTTP/2 support is not enabled
	// automatically.
	TLSNextProto map[string]func(*Server, *tls.Conn, Handler)

	// ConnState specifies an optional callback function that is
	// called when a client connection changes state. See the
	// ConnState type and associated constants for details.
	ConnState func(net.Conn, ConnState)

	// ErrorLog specifies an optional logger for errors accepting
	// connections, unexpected behavior from handlers, and
	// underlying FileSystem errors.
	// If nil, logging is done via the log package's standard logger.
	ErrorLog *log.Logger

	// BaseContext optionally specifies a function that returns
	// the base context for incoming requests on this server.
	// The provided Listener is the specific Listener that's
	// about to start accepting requests.
	// If BaseContext is nil, the default is context.Background().
	// If non-nil, it must return a non-nil context.
	BaseContext func(net.Listener) context.Context

	// ConnContext optionally specifies a function that modifies
	// the context used for a new connection c. The provided ctx
	// is derived from the base context and has a ServerContextKey
	// value.
	ConnContext func(ctx context.Context, c net.Conn) context.Context

	inShutdown atomicBool // true when when server is in shutdown

	disableKeepAlives int32     // accessed atomically.
	nextProtoOnce     sync.Once // guards setupHTTP2_* init
	nextProtoErr      error     // result of http2.ConfigureServer if used

	mu         sync.Mutex
	listeners  map[*net.Listener]struct{}
	activeConn map[*conn]struct{}
	doneChan   chan struct{}
	onShutdown []func()
}

```

## ListenAndServe函数

然后分析`Server`的`ListenAndServe`函数，主要逻辑就是调用socket的接口进行监听然后接收连接，socket相关源码可另行查阅，本文只关注go的http层的相关逻辑

```go
func (srv *Server) ListenAndServe() error {
  // 判断服务是否正在关闭
	if srv.shuttingDown() {
		return ErrServerClosed
	}
  // 填充默认值
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
  // 调用socket函数就行监听
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
  // 启动server接收连接
	return srv.Serve(ln)
}
```

`srv.Serve`的主要流程就是接收新的连接然后对每一个连接启动新的goroutine进行处理

```go
// Serve accepts incoming connections on the Listener l, creating a
// new service goroutine for each. The service goroutines read requests and
// then call srv.Handler to reply to them.
//
// HTTP/2 support is only enabled if the Listener returns *tls.Conn
// connections and they were configured with "h2" in the TLS
// Config.NextProtos.
//
// Serve always returns a non-nil error and closes l.
// After Shutdown or Close, the returned error is ErrServerClosed.
func (srv *Server) Serve(l net.Listener) error {
	if fn := testHookServerServe; fn != nil {
		fn(srv, l) // call hook with unwrapped listener
	}

	origListener := l
	l = &onceCloseListener{Listener: l}
	defer l.Close()

  // 设置http2 server
	if err := srv.setupHTTP2_Serve(); err != nil {
		return err
	}

	if !srv.trackListener(&l, true) {
		return ErrServerClosed
	}
	defer srv.trackListener(&l, false)

	baseCtx := context.Background()
	if srv.BaseContext != nil {
		baseCtx = srv.BaseContext(origListener)
		if baseCtx == nil {
			panic("BaseContext returned a nil context")
		}
	}

	var tempDelay time.Duration // how long to sleep on accept failure

  // 根据接收到的listerner设置相关context
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
    // 等待新的连接
		rw, err := l.Accept()
		if err != nil {
			select {
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
			if ne, ok := err.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", err, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return err
		}
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
    // 启动goroutine处理每一个新的连接
		go c.serve(connCtx)
	}
}
```

然后走到`c.serve`函数，函数较长，这里只保留主要逻辑，最主要的步骤就是调用`serverHandler.ServeHTTP`进行处理

```go
func (c *conn) serve(ctx context.Context) {
    ...
    // 保存response
		c.curReq.Store(w)

		if requestBodyRemains(req.Body) {
			registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead)
		} else {
			w.conn.r.startBackgroundRead()
		}
  
    // 传入response w参数，上层handler处理完写到w，然后后续将w返回给请求方
		serverHandler{c.server}.ServeHTTP(w, w.req)
		w.cancelCtx()
		if c.hijacked() {
			return
		}
		w.finishRequest()
		if !w.shouldReuseConnection() {
			if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
				c.closeWriteAndWait()
			}
			return
		}
		c.setState(c.rwc, StateIdle, runHooks)
		c.curReq.Store((*response)(nil))
}
```

`serverHandler`是一个上层handle的委托，同样实现了`ServeHTTP`方法然后通过上层传递的Handler直接调用`ServeHTTP`进行处理。

```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
  // 若传递Handler为空，则使用默认的DefaultServeMux
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```

在这里也再次印证了上文demo里说的`http.ListenAndServe`函数若第二个参数为`nil`，则使用默认的Handle进行处理。因此接着分析`DefaultServeMux`的`ServeHTTP`方法

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		if r.ProtoAtLeast(1, 1) {
			w.Header().Set("Connection", "close")
		}
		w.WriteHeader(StatusBadRequest)
		return
	}
  
  // 查找每个request对应的handler
	h, _ := mux.Handler(r)
  // 调用上层事先定义的每个路由对应的具体hander进行处理
	h.ServeHTTP(w, r)
}
```

到这里逻辑就已经十分清晰了，通过`mux.Handler`函数查找事先定义的hander，然后直接调用hander的`ServeHTTP`方法处理即可，然后在看下具体是怎么通过请求参数查找对应handler的

```go
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {

	// CONNECT requests are not canonicalized.
	if r.Method == "CONNECT" {
		// If r.URL.Path is /tree and its handler is not registered,
		// the /tree -> /tree/ redirect applies to CONNECT requests
		// but the path canonicalization does not.
		if u, ok := mux.redirectToPathSlash(r.URL.Host, r.URL.Path, r.URL); ok {
			return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
		}

		return mux.handler(r.Host, r.URL.Path)
	}

	// host、path数据清理
  // 只提取主机名
	host := stripHostPort(r.Host)
  // 得到真实的path，如/hello/world/../会被转化为/hello/world
	path := cleanPath(r.URL.Path)

	// If the given path is /tree and its handler is not registered,
	// redirect for /tree/.
	if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
		return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
	}

  // 重定向操作
	if path != r.URL.Path {
		_, pattern = mux.handler(host, path)
		url := *r.URL
		url.Path = path
		return RedirectHandler(url.String(), StatusMovedPermanently), pattern
	}

  // 继续查找得到真正的handler
	return mux.handler(host, r.URL.Path)
}
```

继续调用`mux.handler`进行进一步的处理

```go
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	// 先查找是否为特定主机的path
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
	if h == nil {
		h, pattern = mux.match(path)
	}
  // 没有对应的handler，返回404
	if h == nil {
		h, pattern = NotFoundHandler(), ""
	}
	return
}
```

通过match函数查找pattern对应的路由，若没有对应的路由，返回404

match函数的实现就比较简单了，首先直接进行映射查找，若找到直接返回，否则进行最长匹配查找，直到找到根路劲`/`为止

```go
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	// Check for exact match first.
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

	// Check for longest valid match.  mux.es contains all patterns
	// that end in / sorted from longest to shortest.
	for _, e := range mux.es {
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}
```

# 优雅关停

生产环境中，通常会在`http`服务关闭时处理一些收尾工作，防止对进行到一半的连接实施暴力停止时发生一些意想不到的后果，此时就可以借助http的`Shutdown`函数和go的`channel`进行优雅关停

```go
package main

import (
   "context"
   "fmt"
   "log"
   "net/http"
   "os"
   "os/signal"
   "syscall"
)

func main() {
   mux := http.NewServeMux()
   mux.Handle("/", &helloHandler{})

   server := &http.Server{
      Addr:    ":8081",
      Handler: mux,
   }

   // 创建系统信号接收器
   done := make(chan os.Signal)
   signal.Notify(done, os.Interrupt, syscall.SIGINT, syscall.SIGTERM)
   go func() {
      <-done

      if err := server.Shutdown(context.Background()); err != nil {
         log.Fatal("Shutdown server:", err)
      }
   }()

   log.Println("Starting HTTP server...")
   err := server.ListenAndServe()
   if err != nil {
      if err == http.ErrServerClosed {
         log.Print("Server closed under request")
      } else {
         log.Fatal("Server closed unexpected")
      }
   }
}

type helloHandler struct{}

func (*helloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
   fmt.Fprintf(w, "Hello World")
}
```

使用自定义的`Server`对象作为`sever`端进行处理，主协程启动监听处理连接，然后定义系统信号(Ctrl+c或者`SIGTERM`等)，通过`channel`监听信号变化，收到信号后可以做一些收尾工作，然后调用`server.Shutdown`函数通知主协程停止服务。

# 总结

`go`的`http`包暴露了两个函数`http.Handle/HandleFunc`和`ListenAndServe`，通过这两个函数我们可以快速的搭建起一个http服务，简单易用，内部逻辑对使用者透明。本文从这两个函数入手，分析了其内部实现，即外部通`Handler/HandlerFunc`绑定路由和`hander`，而这些路由和hander都会被封到`ServeMux`这个`strut`中，然后通过一个`http.Server`绑定`ServeMux`这个`Handler`，并通过调用`socket`相关接口实现底层通信，进一步对外透明，体现出分层设计和低耦合的设计思想。最终会根据每个请求的路径`pattern`选择合适的`hander`处理，上层处理完，`socket`直接取结果即可，这其中也涉及用户态与内核态的切换工作。最后使用了优雅关停来处理服务器关闭前的收尾工作。本文分析着重于应用层，相关的socket原理以及高性能服务器实现的多路复用等技术并未涉及，留待以后探讨。