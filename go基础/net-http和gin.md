@[toc]
## net/http ##
### 初始化 ###
#### 注册路由 ####
- 路由最终注册在ServeMux对象中
```go
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry //精确匹配，pattern -> muxEntry
    es    []muxEntry //模糊匹配
    hosts bool       // whether any patterns contain hostnames
}

//存储路由表达式和handler
type muxEntry struct {
    h       Handler
    pattern string
}
```

- 注册流程
```go
func main() {
	//注册路由
	http.HandleFunc("/", HandleHttp)
}


func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}

func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    if handler == nil {
        panic("http: nil handler")
    }
    mux.Handle(pattern, HandlerFunc(handler))
}

var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux

func (mux *ServeMux) Handle(pattern string, handler Handler) {
   mux.mu.Lock()
   defer mux.mu.Unlock()

   ......
   if mux.m == nil {
      mux.m = make(map[string]muxEntry)
   }
   e := muxEntry{h: handler, pattern: pattern}
   //向ServeMux的map[string]muxEntry增加精确匹配路由
   mux.m[pattern] = e
   //然后如果路由表达式以'/'结尾，则将对应的muxEntry对象加入到[]muxEntry中，按照路由表达式长度排序，用于后续模糊匹配
   if pattern[len(pattern)-1] == '/' {
      mux.es = appendSorted(mux.es, e)
   }

   if pattern[0] != '/' {
      mux.hosts = true
   }
}
```

#### 开启服务 ####
- 调用Server.Serve方法
```go
func main() {
    err := http.ListenAndServe("127.0.0.1:8080", nil)
}

type Server struct {
   Addr string
   Handler Handler
}

func ListenAndServe(addr string, handler Handler) error {
    //创建了一个Server对象，传入了地址和handler参数
    server := &Server{Addr: addr, Handler: handler}
    return server.ListenAndServe()
}

func (srv *Server) ListenAndServe() error {
    if srv.shuttingDown() {
        return ErrServerClosed
    }
    //初始化监听地址Addr
    addr := srv.Addr
    if addr == "" {
        addr = ":http"
    }
    //调用Listen方法设置监听
    ln, err := net.Listen("tcp", addr)
    if err != nil {
        return err
    }
    //将监听的TCP对象传入Serve方法
    return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```

- 一个goroutine处理一个连接
```go
func (srv *Server) Serve(l net.Listener) error {
    ...

    baseCtx := context.Background() // base is always background, per Issue 16220
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        //等待新的连接建立
        rw, e := l.Accept() 

        ...

        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        //开启一个新的goroutine处理连接请求
        go c.serve(ctx)
    }
}

func (srv *Server) newConn(rwc net.Conn) *conn {
   c := &conn{
      server: srv,
      rwc:    rwc,
   }
   if debugServerConnections {
      c.rwc = newLoggingConn("server", c.rwc)
   }
   return c
}
```


- 当一个连接建立之后，该连接中所有的请求都将在这个**协程**中进行处理，直到连接被关闭。
```go
func (c *conn) serve(ctx context.Context) {

    ...

    for {
        w, err := c.readRequest(ctx)
        if c.r.remain != c.server.initialReadLimitSize() {
            // If we read any bytes off the wire, we're active.
            c.setState(c.rwc, StateActive)
        }
        ...
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
        c.setState(c.rwc, StateIdle)
        c.curReq.Store((*response)(nil))

        ...
    }


	type serverHandler struct {
	    srv *Server
	}
}
```

- sh.srv.Handler其实就是我们最初在http.ListenAndServe()中传入的第二个参数，通常为nil
```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}
```

### 处理请求 ###
#### 匹配路由 ####
- ServeMux匹配url和实际的handler，先精确匹配，后模糊匹配
  -  对于类似/path1/path2/path3这样的路由，如果不能找到精确匹配的路由规则，那么则会去匹配和当前路由最接近的已注册的父节点路由，所以如果路由/path1/path2/已注册，那么该路由会被匹配，否则继续匹配下一个父节点路由，直到根路由/。
  -  由于[]muxEntry中的muxEntry按照路由表达式从长到短排序，所以进行近似匹配时匹配到的节点路由一定是已注册父节点路由中最相近的

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
}

func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    mux.mu.RLock()
    defer mux.mu.RUnlock()

    // Host-specific pattern takes precedence over generic ones
    if mux.hosts {
        h, pattern = mux.match(host + path)
    }
    if h == nil {
        h, pattern = mux.match(path)
    }
    if h == nil {
        h, pattern = NotFoundHandler(), ""
    }
    return
}

func (mux *ServeMux) match(path string) (h Handler, pattern string) {
    //首先会利用进行精确匹配，在map[string]muxEntry中查找是否有对应的路由规则存在
    v, ok := mux.m[path]
    if ok {
        return v.h, v.pattern
    }

    //如果没有匹配的路由规则，则会利用es进行近似匹配
    for _, e := range mux.es {
        if strings.HasPrefix(path, e.pattern) {
            return e.h, e.pattern
        }
    }
    return nil, ""
}
```

## gin ##
> 封装net/http，实现路由，支持中间件

### 相对net/http的优化点 ###
1. 路由快速：基于基数树的httprouter
2. 支持中间件：方便、灵活，支持自定义
3. 支持崩溃恢复：可以捕捉panic引发的程序崩溃，使Web服务可以一直运行。自带中间件实现。
4. JSON验证：可以验证请求中JSON数据格式。
5. 路由分组：支持路由分组(RouteGroup)，可以更方便组织路由。
6. 多种数据渲染方式：支持HTML、JSON、YAML、XML等数据格式的响应。
7. 开源框架，开发者活跃

### 路由数据结构 ###
#### 基数树 ####
- 使用基于基数树（Radix Tree）的httprouter
  - 压缩版前缀树，：对于基数树的每个节点，如果该节点是唯一的子树的话，就和父节点合并![210714.radixtree.png](https://img-blog.csdnimg.cn/b1881c99a5b74d9295a1db438c1f1e1b.png)
- 路由树节点
```golang
// tree.go
type node struct {
   // 节点路径，比如上面的s，earch，和upport
    path      string
    // 和children字段对应, 保存的是分裂的分支的第一个字符
    // 例如search和support, 那么s节点的indices对应的"eu"
    // 代表有两个分支, 分支的首字母分别是e和u
    indices   string
    // 儿子节点
    children  []*node
    // 处理函数链条（切片）
    handlers  HandlersChain
    ......
}
```

- 每个HTTP方法对应一个基数树
```golang
// gin.go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
   // yangxiangrui.site...
   
   // 获取请求方法对应的树
	root := engine.trees.get(method)
	if root == nil {
	
	   // 如果没有就创建一个
		root = new(node)
		root.fullPath = "/"
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
	root.addRoute(path, handlers)
}
```
#### 路由组 ####
- 路由组是对路由树的封装
```go
type RouterGroup struct {
   Handlers HandlersChain //中间件
   basePath string
   engine   *Engine
   root     bool
}

type Engine struct {
   RouterGroup
   trees            methodTrees
   ......
}

type methodTree struct {
   method string
   root   *node //各种HTTP方法的路有树
}
```

### 初始化 ###
#### 注册路由 ####
```go
func main(){
    group.GET(relativePath, handler)
}

func (group *RouterGroup) GETEX(relativePath string, handler gin.HandlerFunc, handlerName string) gin.IRoutes {
   internal.SetHandlerName(handler, handlerName)
   return
}

func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
   return group.handle(http.MethodGet, relativePath, handlers)
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
   absolutePath := group.calculateAbsolutePath(relativePath)
// 业务逻辑添加到在RouterGroup的Handlers上
   handlers = group.combineHandlers(handlers)
// RouterGroup的Handlers和path绑定后，加入基数树
   group.engine.addRoute(httpMethod, absolutePath, handlers)
   return group.returnObj()
}

func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
   finalSize := len(group.Handlers) + len(handlers)
   if finalSize >= int(abortIndex) {
      panic("too many handlers")
   }
   mergedHandlers := make(HandlersChain, finalSize)
   copy(mergedHandlers, group.Handlers)
   copy(mergedHandlers[len(group.Handlers):], handlers)
   return mergedHandlers
}

func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
   ......
   root.addRoute(path, handlers)
   ......
}

// addRoute 将具有给定句柄的节点添加到路径中。
// 不是并发安全的
func (n *node) addRoute(path string, handlers HandlersChain) {
        fullPath := path
        n.priority++
        numParams := countParams(path)  // 数一下参数个数

        // 空树就直接插入当前节点
        if len(n.path) == 0 && len(n.children) == 0 {
                n.insertChild(numParams, path, fullPath, handlers)
                n.nType = root
                return
        }

        parentFullPathIndex := 0

walk:
        for {
                ......

                // 找到最长的通用前缀
                i := longestCommonPrefix(path, n.path)

                // 节点分裂
                // 例如一开始path是search，新加入support，s是他们通用的最长前缀部分
                // 分裂成父节点 s 子节点：earch 和 upport
                // 公共前缀后的部分作为子节点
                if i < len(n.path) {
                        child := node{
                                path:      n.path[i:],  
                                wildChild: n.wildChild,
                                indices:   n.indices,
                                children:  n.children,
                                handlers:  n.handlers,
                                priority:  n.priority - 1, //子节点优先级-1
                                fullPath:  n.fullPath,
                        }
                        n.children = []*node{&child}
                        n.indices = bytesconv.BytesToString([]byte{n.path[i]})
                        n.path = path[:i]
                        n.handlers = nil
                        n.wildChild = false
                        n.fullPath = fullPath[:parentFullPathIndex+i]

                      ......                }

                // 将新来的节点插入新的parent节点作为子节点
                if i < len(path) {
                        ......
                        n.insertChild(numParams, path, fullPath, handlers)
                        return
                }
                
                // 路由重复校验
                if n.handlers != nil {
                   panic("handlers are already registered for path '" + fullPath + "'")
                }

                ......
                return
        }
}
````

#### 注册中间件 ####
- 注册中间件其实就是将中间件函数追加到RouterGroup.Handlers中
- 中间件+业务逻辑构成请求处理链
```go
//默认的中间件
func Default() *Engine {
   r := New()
   mwConfig := DefaultMiddlewareConfig()
   r.Use(mwConfig.MiddlewareList()...)
   return r
}

func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
   engine.RouterGroup.Use(middleware...)
   engine.rebuild404Handlers()
   engine.rebuild405Handlers()
   return engine
}

//注册中间件其实就是将中间件函数追加到RouterGroup.Handlers中：
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
   group.Handlers = append(group.Handlers, middleware...)
   return group.returnObj()
}
```

#### 开启服务 ####
- 底层原生net/http
```go
func (engine *Engine) Run(addr ...string) (err error) {
   ......
   if listener, hook, err := createListener(); nil != err {
      logs.Warnf("create listen fail, err is %s", err)
      return err
   } else {

      errCh := make(chan error, 1)
      go func() {
         logs.Info("Run in %s mode", appConfig.Mode)
         server := &http.Server{Handler: engine}
         if err := doHttpServerConfig(server); nil != err {
            errCh <- err
         } else {
            errCh <- netex.ListenAndServe(listener, server)
         }
      }()
      startDebugServer()
      data := generateFicData(engine)
      data["protocol"] = "http"
      reportMetainfo(data)
      // start report go gc stats
      stats.DoReport(PSM())
      if err = waitSignal(errCh, hook); err != nil {
         logs.Warnf("wait signal fail, err is %s", err)
         return err
      }
      return nil
   }
}


func ListenAndServe(l net.Listener, s *http.Server) error {
    // 底层引用原生的net/http包
   if strings.HasPrefix(l.Addr().String(), "/") {
      return s.Serve(l)
   } else {
      return s.Serve(ApplyKeepAlive(l))
   }
}
```

### 处理请求 ###
#### 匹配路由 ####
1. 匹配HTTP method对应的基数树
```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    //context对象池
   c := engine.pool.Get().(*Context)
   c.writermem.reset(w)
   c.Request = req
   c.reset()

   engine.handleHTTPRequest(c)

   engine.pool.Put(c)
}

func (engine *Engine) handleHTTPRequest(c *Context) {
        // 根据请求方法找到对应的路由树
        t := engine.trees
        for i, tl := 0, len(t); i < tl; i++ {
                if t[i].method != httpMethod {
                        continue
                }
                root := t[i].root
                // 在基数树中根据path查找
                value := root.getValue(rPath, c.Params, unescape)
                if value.handlers != nil {
                        c.handlers = value.handlers
                        c.Params = value.params
                        c.fullPath = value.fullPath
                        c.Next()  // 执行中间件职责链
                        c.writermem.WriteHeaderNow()
                        return
                }
 
        c.handlers = engine.allNoRoute
        serveError(c, http.StatusNotFound, default404Body)
}
```

2. 基数树中寻找handlers
``` go
type nodeValue struct {
        handlers HandlersChain
        params   Params  
        tsr      bool
        fullPath string
}

func (n *node) getValue(path string, po Params, unescape bool) (value nodeValue) {
        value.params = po
walk: // Outer loop for walking the tree
        for {
                prefix := n.path
                if path == prefix {
                        // 我们应该已经到达包含处理函数的节点。
                        // 检查该节点是否注册有处理函数
                        if value.handlers = n.handlers; value.handlers != nil {
                                value.fullPath = n.fullPath
                                return
                        }

                        if path == "/" && n.wildChild && n.nType != root {
                                value.tsr = true
                                return
                        }

                        // 没有找到处理函数 检查这个路径末尾+/ 是否存在注册函数
                        indices := n.indices
                        for i, max := 0, len(indices); i < max; i++ {
                                if indices[i] == '/' {
                                        n = n.children[i]
                                        value.tsr = (len(n.path) == 1 && n.handlers != nil) ||
                                                (n.nType == catchAll && n.children[0].handlers != nil)
                                        return
                                }
                        }

                        return
                }

                if len(path) > len(prefix) && path[:len(prefix)] == prefix {
                        path = path[len(prefix):]
                        // 如果该节点没有通配符(param或catchAll)子节点
                        // 我们可以继续查找下一个子节点
                        if !n.wildChild {
                                c := path[0]
                                indices := n.indices
                                for i, max := 0, len(indices); i < max; i++ {
                                        if c == indices[i] {
                                                n = n.children[i] // 遍历树
                                                continue walk
                                        }
                                }

                                // 没找到
                                // 如果存在一个相同的URL但没有末尾/的叶子节点
                                // 我们可以建议重定向到那里
                                value.tsr = path == "/" && n.handlers != nil
                                return
                        }

                        // 根据节点类型处理通配符子节点
                        n = n.children[0]
                        switch n.nType {
                        ......

                        default:
                                panic("invalid node type")
                        }
                }

              ......
        }
}
```
#### 执行 ####
1. 执行中间件+业务逻辑的职责链
```go
func (c *Context) Next() {
        c.index++
        for c.index < int8(len(c.handlers)) {
                c.handlers[c.index](c)
                c.index++
        }
}
```

2. 如果自定义中间件，可以使用next()实现切面<br>![210714.netx.png](https://img-blog.csdnimg.cn/1bc096386c0f40d49aa8b1a42aa06d46.png)
```go
func middleware(ctx *Context){
    //before
    c.Next()
    //after
}
```

## 参考 ##
- [深入理解Golang之http server](https://juejin.cn/post/6844903998869209095)
- [Gin源码解析](https://blog.csdn.net/weixin_45961841/article/details/113722686?ivk_sa=1024609v)