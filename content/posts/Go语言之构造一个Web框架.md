---
title: Go语言之构造一个Web框架
tags:
  - Golang
date: 2024-10-20 16:20:05
draft: false
---
#### 参考文献
[Go语言动手写Web框架 - Gee第二天 上下文Context | 极客兔兔](https://geektutu.com/post/gee-day2.html)

大佬写的非常清晰易懂，跟着走下来对整个web框架的了解变得更加深入了，也整理了一些学习心得

#### 整体架构
简单介绍一下整体框架的设计思路以及对应需要实现的一些功能，框架的具体实现主要是根据极客兔兔的博客以及极客时间的教学视频进行的
Web框架主要是用于解决标准库中需要频繁处理的问题或者将处理起来较为麻烦的部分进行一定程度上的封装
根据HTTP基础部分整体的框架其实很清楚了，用户这边写的是路由和对应的处理方法，框架用HandlerFunc来接受对应的handler，与对应的路由通过map进行绑定，通过不同的请求可以映射到不同的处理方法
首先要有一个整体代表服务器的抽象，Server，提供生命周期控制、路由注册接口，作为http包到Web框架的桥梁，怎么接入http包？通过http包暴露的一个接口handler来作为Web框架与http包的连接点，其次是对于路由的处理，这也是Web框架的核心之一，通过动态路由将请求映射到函数，还有一些功能如上下文，分组控制，中间件等等。

#### 功能简介
**动态路由**：对于路由来说，最重要的当然是注册与匹配了。开发服务时，注册路由规则，映射handler；访问时，匹配路由规则，查找到对应的handler。因此，`Trie树`需要支持节点的插入与查询。插入功能很简单，递归查找每一层的节点，如果没有匹配到当前`part`的节点，则新建一个。
**上下文**：将`Handler`的参数变成`Context`，设计`上下文(Context)`，封装`*http.Request`和`http.ResponseWriter`的方法，简化相关接口的调用，提供对 JSON、HTML 等返回类型的支持。
**分组控制**：`分组控制(Group Control)`是 Web 框架应提供的基础功能之一。所谓分组，是指路由的分组。如果没有路由分组，我们需要针对每一个路由进行控制。但是真实的业务场景中，往往某一组路由需要相似的处理。
1. 以`/post`开头的路由匿名可访问。
2. 以`/admin`开头的路由需要鉴权。
3. 以`/api`开头的路由是 RESTful 接口，可以对接第三方平台，需要三方平台鉴权。

**中间件**：`中间件(middlewares)`，就是非业务的技术类组件。Web框架本身不可能去理解所有的业务，因而不可能实现所有的功能。因此，框架需要有一个插口，允许用户自己定义功能，嵌入到框架中，仿佛这个功能是框架原生支持的一样。
- 插入点在哪？使用框架的人并不关心底层逻辑的具体实现，如果插入点太底层，中间件逻辑就会非常复杂。如果插入点离用户太近，那和用户直接定义一组函数，每次在 Handler 中手工调用没有多大的优势了。
- 中间件的输入是什么？中间件的输入，决定了扩展能力。暴露的参数太少，用户发挥空间有限。

**HTML模板**：`HTML模板（template）`部分主要是实现两个内容：
- 实现静态资源服务(Static Resource)
- 支持HTML模板渲染
将静态文件放在`/web`，`filepath`的值即是该目录下文件的相对地址。映射到真实的文件后，将文件返回，静态服务器就实现了。
返回文件直接交给`http.FileServer`处理就好了
Go语言内置了`html/template`为 HTML 提供了较为完整的支持。包括普通变量渲染、列表渲染、对象渲染等。

**错误处理**：用户不正确的参数，可能会触发某些异常，例如数组越界，空指针等。如果因为这些原因导致系统宕机，必然是不可接受的，所以错误处理机制是非常必要的，将错误处理作为一个中间件添加到框架中，向用户返回 _Internal Server Error_，并且在日志中打印必要的错误信息，方便进行错误定位。

#### Http库下方法的使用
这一部分主要是介绍一下net/http库下的一些基础功能使用，Body的获取，URL的查询，Form表单的输出等等
- `http.ResponseWriter`: 这是一个接口，提供了对 HTTP 响应的返回方法。你可以使用它来写入 HTTP 响应的头信息和主体内容。
- `*http.Request`: 这是一个指向 `http.Request` 的指针，表示 HTTP 请求。它包含了关于 HTTP 请求的所有信息，如请求方法、URL、头信息、表单数据等
##### Body和GetBody
```Go
func home(w http.ResponseWriter, r *http.Request) {  
    //body 只能读取一次  
    body, err := io.ReadAll(r.Body)  
    if err != nil {  
       fmt.Println(w, "读取请求体失败")  
       return  
    }  
    fmt.Fprintln(w, "第一次请求：", string(body), "长度为", len(string(body)))  
    //尝试再次读取，不会报错，但是什么也没有读到  
    body, err = io.ReadAll(r.Body)  
    if err != nil {  
       fmt.Println(w, "读取请求体失败")  
       return  
    }  
    fmt.Fprintln(w, "第二次请求：", string(body), "长度为", len(string(body)))  
    if r.GetBody == nil {  
       fmt.Fprintln(w, "GetBody为空")  
    } else {  
       fmt.Fprintln(w, "GetBody不为空")  
    }  
}
```
##### URL
```Go
func query(w http.ResponseWriter, r *http.Request) {  
    //查询参数是一个map，type Values map[string][]string  
    values := r.URL.Query()  
    //url.Values类型是一个映射字符串到字符串切片的映射，通常用于处理URL查询参数  
    fmt.Fprintln(w, reflect.TypeOf(values))  
    name := values.Get("name")  
    fmt.Fprintln(w, "查询对应的值为", name)  
}

func wholeUrl(w http.ResponseWriter, r *http.Request) {  
    data, _ := json.Marshal(r.URL)  
    fmt.Fprintln(w, string(data))  
}
```

##### Form
```GO
func form(w http.ResponseWriter, r *http.Request) {  
    //直接输出表单,没有对应输出  
    fmt.Fprintln(w, r.Form)  
    //需要先解析表单，才会有对应输出  
    r.ParseForm()  
    fmt.Fprintln(w, r.Form)  
}
```

#### Server实现
`Httpserver` 是HTTP服务器的封装，实现了`Server`接口，提供路由注册，生命周期控制以及作为与http包结合的桥梁
`ServerHTTP`是整个Web框架的核心入口，在其中将实现大部分功能，Context构建，路由匹配以及业务逻辑的执行
**封装与抽象**：将HTTP请求的处理、响应的生成、路由的注册等逻辑都通过封装类和接口进行抽象，`Context`封装了请求和响应对象，`HandlerBaseOnMap`封装了路由逻辑，`sdkHttpserver`封装了服务器的启动和路由配置。
```Go
// Server 定义一个服务器接口  
type Server interface {  
    Routable  
    Start(address string) error  
    GET(pattern string, handlerFunc func(c *Context))  
    POST(pattern string, handlerFunc func(c *Context))  
}  
type Httpserver struct {  
    Name    string  
    handler *Router  
}  
  
func NewHttpServer(name string) Server {  
    return &Httpserver{  
       Name:    name,  
       handler: NewRouter(),  
    }  
}  
func (s *Httpserver) Route(method string, pattern string, handlerFunc HandlerFunc) {  
    s.handler.Route(method, pattern, handlerFunc)  
}  
func (s *Httpserver) GET(pattern string, handlerFunc func(c *Context)) {  
    s.handler.Route("GET", pattern, handlerFunc)  
}  
func (s *Httpserver) POST(pattern string, handlerFunc func(c *Context)) {  
    s.handler.Route("POST", pattern, handlerFunc)  
}  
func (s *Httpserver) Start(address string) error {  
    //http.Handle("/", s)  
    //net.Listen函数创建一个监听器，监听器监听的是TCP网络地址  
    //l, err := net.Listen("tcp", address)  
    //if err != nil {    // return err    //}    
    //http.serve 启动的灵活性更强
    //return http.Serve(l, s)    
    
    //ListenAndServe监听TCP网络地址addr，并为该网络地址的网络连接提供HTTP服务。  
    return http.ListenAndServe(address, s)  
}  
func (s *Httpserver) ServeHTTP(W http.ResponseWriter, R *http.Request) {  
    c := NewContext(W, R)  
    s.handler.handle(c)  
}
```

 #### 实现Handler接口
 Go通过接口实现多态，Go的接口是隐式的，只要结构体上定义的“方法”在形式上（名称、参数、返回值）和 接口定义的“方法”一样，那么这个结构体就自动实现了这个接口，我们就可以使用这个接口变量来指向这个结构体对象，实际上也就是多态和继承。
`Handler`是一个接口，需要实现方法 _ServeHTTP_ ，也就是说，只要传入任何实现了 _ServerHTTP_ 接口的实例，所有的HTTP请求，都交给了该实例进行处理，_ServeHTTP_ 方法的作用就是，解析请求的路径，查找路由映射表，如果查到，就执行注册的处理方法。
如何设计方法`ServeHTTP`，这个方法有2个参数，第二个参数是 _Request_ ，该对象包含了该HTTP请求的所有的信息，比如请求地址、Header和Body等信息；第一个参数是 _ResponseWriter_ ，利用 _ResponseWriter_ 可以构造针对该请求的响应
 ```Go
// HandlerFunc 定义使用的请求处理程序  
type HandlerFunc func(c *Context)  
  
// 确保Httpserver实现了Server接口  
var _ Server = &Httpserver{}  
  
// Server 定义一个服务器接口  
type Server interface {  
    AddRoute(method string, pattern string, handlerFunc HandlerFunc)  
    Start(address string) error  
    GET(pattern string, handlerFunc HandlerFunc)  
    POST(pattern string, handlerFunc HandlerFunc)  
    NewGroup(prefix string) *RouterGroup  
    Use(middlewares ...HandlerFunc)  
}  
  
type Httpserver struct {  
    Name    string  
    handler *Router  
    *RouterGroup  
    Groups []*RouterGroup  
}

func NewHttpServer(name string) Server {  
    server := &Httpserver{  
       Name:    name,  
       handler: NewRouter(),  
    }  
    //初始化 RouterGroup，将其与server关联  
    server.RouterGroup = &RouterGroup{server: server}  
    // 将该 RouterGroup 添加到 server 的 groups 列表中  
    server.Groups = []*RouterGroup{server.RouterGroup}  
    return server  
}

func (s *Httpserver) ServeHTTP(W http.ResponseWriter, R *http.Request) {  
    //当我们接收到一个具体请求时，要判断该请求适用于哪些中间件，在这里我们简单通过URL的前缀来判断。得到中间件列表后，赋值给c.handlers  
    var middlewares []HandlerFunc  
    for _, group := range s.Groups {  
       if strings.HasPrefix(R.URL.Path, group.prefix) {  
          middlewares = append(middlewares, group.middlewares...)  
       }  
    }  
    c := NewContext(W, R)  
    c.handlers = middlewares  
    s.handler.handle(c)  
}

 ```

#### 上下文定义
Context随着每一个请求的出现而产生，请求的结束而销毁，和当前请求强相关的信息都应由Context承载。因此，设计Context结构，扩展性和复杂性留在了内部，而对外简化了接口。路由的处理函数，以及将要实现的中间件，参数都统一使用Context实例，Context就像一次会话的百宝箱，可以找到任何东西
我大致明白对应的意思了，实际上就是不需要自己去写对应的读取请求解析请求，而是转交给对应的框架来实现，而这部分框架就是用Context来实现的，用来简化传输的部分
```Go
type Context struct {  
    R *http.Request  
    W http.ResponseWriter  
}  
  
// ReadJson 读取json数据  
func (c *Context) ReadJson(resp interface{}) error {  
    body, err := io.ReadAll(c.R.Body)  
    if err != nil {  
       return err  
    }  
    err = json.Unmarshal(body, resp)  
    if err != nil {  
       return err  
    }  
    return nil  
}  
  
// WriteJson 写入json数据  
func (c *Context) WriteJson(code int, resp interface{}) error {  
    c.W.WriteHeader(code)  
    data, err := json.Marshal(resp)  
    if err != nil {  
       return err  
    }  
    _, err = c.W.Write(data)  
    if err != nil {  
       return err  
    }  
    return nil  
}  
func (c *Context) OkJson(resp interface{}) error {  
    return c.WriteJson(http.StatusOK, resp)  
}  
func (c *Context) BadRequestJson(resp interface{}) error {  
    return c.WriteJson(http.StatusBadRequest, resp)  
}
```

#### 路由实现
##### 静态路由（Map实现）
 使用map定义一个handler，用来实现**路由功能**，允许根据HTTP方法和URL路径将请求分发到相应的处理函数
使用map来存储对应的函数，key是method与pattern的组合，将所有方法存储在哈希表中，然后通过key去匹配
```Go
type Handler interface {  
    http.Handler  
    Routable
}  
//http.Handler的实现，直接继承过来
type Handler interface {  
    ServeHTTP(ResponseWriter, *Request)  
}
```

创建一个实例用来进行我们自己的处理逻辑
```Go
type HandlerBaseOnMap struct {  
    handlers map[string]func(c *Context)  
}  
```

整体的Handler部分
```Go
type Routable interface {  
    Route(method string, pattern string, handlerFunc func(c *Context))  
}  
  
// 基于map的路由  
type Handler interface {  
    http.Handler  
    Routable
}  
type HandlerBaseOnMap struct {  
    handlers map[string]func(c *Context)  
}  
  
func (h *HandlerBaseOnMap) Route(method string, pattern string, handlerFunc func(c *Context)) {  
    key := h.key(method, pattern)  
    h.handlers[key] = handlerFunc  
}  
  
func (h *HandlerBaseOnMap) ServeHTTP(writer http.ResponseWriter, request *http.Request) {  
    key := h.key(request.Method, request.URL.Path)  
    if handler, ok := h.handlers[key]; ok {  
       handler(NewContext(writer, request))  
    } else {  
       writer.WriteHeader(http.StatusNotFound)  
    }  
}  
  
func (h *HandlerBaseOnMap) key(method string, pattern string) string {  
    return method + "#" + pattern  
}  
func NewHandlerBasedOnMap() Handler {  
    return &HandlerBaseOnMap{  
       handlers: make(map[string]func(c *Context)),  
    }  
}
```

##### 动态路由（Trie树实现）
之前用了一个非常简单的`map`结构存储了路由表，使用`map`存储键值对，索引非常高效，但是有一个弊端，键值对的存储的方式，只能用来索引静态路由。那如果我们想支持类似于`/hello/:name`这样的动态路由怎么办呢？所谓动态路由，即一条路由规则可以匹配某一类型而非某一条固定的路由。例如`/hello/:name`，可以匹配`/hello/geektutu`、`hello/jack`等。
实现的动态路由主要具备两个功能：
- 参数匹配`:`。例如 `/p/:lang/doc`，可以匹配 `/p/c/doc` 和 `/p/go/doc`。
- 通配`*`。例如 `/static/*filepath`，可以匹配`/static/fav.ico`，也可以匹配`/static/js/jQuery.js`，这种模式常用于静态服务器，能够递归地匹配子路径
实现动态路由最常用的数据结构，被称为前缀树(Trie树)：每一个节点的所有的子节点都拥有相同的前缀。
介绍一下前缀树的概念：
**前缀树**本身是一个多叉树结构，树中的每个节点存储一个字符，它与普通树状数据结构最大的差异在于，存储数据的 key 不存放于单个节点中，而是由从根节点 root 出发直到来到目标节点target node之间的**沿途路径组成**. 基于这样的构造方式，导致拥有相同前缀的字符串可以复用公共的父节点，直到在首次出现不同字符的位置才出现节点分叉，最终形成多叉树状结构.
比如以下面search、see、seat为例，从根节点出发，先存储se，然后进行分叉，分出a和e，接着继续分叉分出r和t
当存储拥有公共前缀的内容时，可以在很大程度上节省空间提高节点利用率. 同时由于这种公共前缀的设计方式，也赋以了Trie树能够支持前缀频率统计以及前缀模糊匹配的功能

#### 分组控制
这里实现的分组控制也是以前缀来区分，并且支持分组的嵌套。例如`/post`是一个分组，`/post/a`和`/post/b`可以是该分组下的子分组。作用在`/post`分组上的中间件(middleware)，也都会作用在子分组，子分组还可以应用自己特有的中间件。中间件可以给框架提供无限的扩展能力，应用在分组上，可以使得分组控制的收益更为明显，而不是共享相同的路由前缀这么简单。例如`/admin`的分组，可以应用鉴权中间件；`/`分组应用日志中间件。
首先是一开始结构体的定义
```Go
//server作为顶层的分层，要在server里添加RouterGroup
type Httpserver struct {
    Name    string
    handler *Router
    *RouterGroup
    Groups []*RouterGroup
}

//在Group中，保存一个指针，指向server
type RouterGroup struct {
    prefix      string        //所有该组中的路由都将共享这个前缀
    middlewares []HandlerFunc //支持中间件
    parent      *RouterGroup  //支持嵌套
    server      *Httpserver   //所有组共享一个server实例
}

func NewHttpServer(name string) Server {
    server := &Httpserver{
        Name:    name,
        handler: NewRouter(),
    }
    //初始化 RouterGroup，将其与server关联
    server.RouterGroup = &RouterGroup{server: server}
    // 将该 RouterGroup 添加到 server 的 groups 列表中
    server.Groups = []*RouterGroup{server.RouterGroup}
    return server
}
```

Group的构造函数，用来构造RouterGroup
```Go
// NewGroup 以创建新的 RouterGroup
// 所有组共享同一个 server 实例
func (g *RouterGroup) NewGroup(prefix string) *RouterGroup {
    server := g.server
    //创建一个新的Group
    newGroup := &RouterGroup{
        prefix: g.prefix + prefix, // 新组的前缀是父组前缀加上新前缀
        parent: g,
        server: server, //所有组共享同一个 server 实例
    }
    server.Groups = append(server.Groups, newGroup)
    return newGroup
}
```

后面就是将路由相关的函数都通过RouterGroup来实现
```Go
//将和路由有关的函数，都交给RouterGroup实现
func (g *RouterGroup) AddRoute(method string, pattern string, handlerFunc HandlerFunc) {
    //将路由组的前缀（g.prefix）和当前路由的模式（pattern）拼接在一起，形成完整的路由路径
    pattern = g.prefix + pattern
    //日志输出，非常直观
    log.Printf("Route %4s - %s", method, pattern)
    g.server.handler.AddRoute(method, pattern, handlerFunc)
}
func (g *RouterGroup) GET(pattern string, handlerFunc HandlerFunc) {
    g.AddRoute(http.MethodGet, pattern, handlerFunc)
}

func (g *RouterGroup) POST(pattern string, handlerFunc HandlerFunc) {
    g.AddRoute(http.MethodPost, pattern, handlerFunc)
}
```

然后是在主函数里面进行分组
```Go
    v1 := server.NewGroup("/v1")
    {
        v1.GET("/hello", Hello)
        v1.GET("/signup", Signup)
    }
    v2 := server.NewGroup("/v2")
    {
        v2.GET("/hello/:name", Param)
        v2.POST("/login", Login)
    }
```

#### 中间件
中间件的定义与路由映射的`Handler`一致，处理的输入是`Context`对象。**插入点**是框架接收到请求初始化`Context`对象后，允许用户使用自己定义的中间件做一些额外的处理，例如记录日志等，以及对`Context`进行二次加工。另外通过调用`(*Context).Next()`函数，中间件可等待用户自己定义的`Handler`处理结束后，再做一些额外的操作，例如计算本次处理所用时间等。即所实现的中间件支持用户在请求被处理的前后，做一些额外的操作。
之前的框架设计是这样的，当接收到请求后，匹配路由，该请求的所有信息都保存在`Context`中，中间件也不例外，接收到请求后，应查找所有应作用于该路由的中间件，保存在`Context`中，依次进行调用。
```Go
// Logger 当在中间件中调用Next方法时，控制权交给了下一个中间件，直到调用到最后一个中间件，然后再从后往前，调用每个中间件在Next方法之后定义的部分
func Logger() HandlerFunc {  
	return func(c *Context) {  
		// 开始计时
		t := time.Now()  
		// 处理请求
		c.Next()  
		// 计算处理时间 
		log.Printf("[%d] %s in %v", c.StatusCode, c.Req.RequestURI, time.Since(t))  
	}  
}
```


#### HTML模板渲染
处理静态文件
```Go
// createStaticHandler 静态文件的处理程序  
func (g *RouterGroup) createStaticHandler(relativePath string, fs http.FileSystem) HandlerFunc {  
    absolutePath := path.Join(g.prefix, relativePath)  
    fileServer := http.StripPrefix(absolutePath, http.FileServer(fs))  
    return func(c *Context) {  
       file := c.Param("filepath")  
       // 检查文件是否存在  
       if _, err := fs.Open(file); err != nil {  
          c.Status(http.StatusNotFound)  
          return  
       }  
       fileServer.ServeHTTP(c.W, c.R)  
    }  
}  
  
// Static 提供静态文件  
// 用户可以将磁盘上的某个文件夹root映射到路由relativePath  
func (g *RouterGroup) Static(relativePath string, root string) {  
    handler := g.createStaticHandler(relativePath, http.Dir(root))  
    urlPattern := path.Join(relativePath, "/*filepath")  
    g.GET(urlPattern, handler)  
}
```

HTML模板渲染
```Go
Engine struct {  
	*RouterGroup  
	router        *router  
	groups        []*RouterGroup     // store all groups  
	htmlTemplates *template.Template // for html render  
	funcMap       template.FuncMap   // for html render  
}  
  
func (engine *Engine) SetFuncMap(funcMap template.FuncMap) {  
	engine.funcMap = funcMap  
}  
  
func (s *Httpserver) LoadHTMLGlob(pattern string) {  
    //因为运行配置里写的工作目录是LearnGo，修改一下变成LearnGo\web就可以了  
    s.htmlTemplates = template.Must(template.New("").Funcs(s.funcMap).ParseGlob(pattern))  
}
```

#### 错误处理
使用error与panic用来进行错误处理，**error**一般用于表达可以被处理的错误，**panic**一般用于表达非常严重不能恢复的错误，遇事不决选error
errors包下的方法：**New**创建一个新的error，**Is**判断是不是特定的某个error，**As**类型转换为特定的error，**Unwrap**解除包装，返回被包装的error
```Go
func ErrorsPkg() {  
    err := &MyError{}  
    //返回的是一个包装好的error  
    wrappedErr := fmt.Errorf("this is an wrapped error %w", err)  
    //再解出来  
    if errors.Is(err, errors.Unwrap(wrappedErr)) {  
       fmt.Println("unwrapped")  
    }  
    //判断是否是包装好的error  
    if errors.Is(wrappedErr, err) {  
       fmt.Println("wrapped is err")  
    }  
    copyErr := &MyError{}  
    //尝试将wrappedErr转换成为MyError  
    if errors.As(wrappedErr, &copyErr) {  
       fmt.Println("convert error")  
    }  
}
```

延迟（defer）从下往上执行，先进后出，类似于栈
```Go
defer func() {  
    fmt.Println("aaa")  
}()  
defer func() {  
    fmt.Println("bbb")  
}()  
defer func() {  
    fmt.Println("ccc")  
}()
```

错误处理也可以作为一个中间件，增强整体框架的能力
```Go
func Recovery() HandlerFunc {  
	return func(c *Context) {  
		defer func() {  
			if err := recover(); err != nil {  
				message := fmt.Sprintf("%s", err)  
				log.Printf("%s\n\n", trace(message))  
				c.String(http.StatusInternalServerError, "Internal Server Error")  
			}  
		}()  
  
		c.Next()  
	}  
}
```
