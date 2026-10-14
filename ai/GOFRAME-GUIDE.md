# GoFrame 实战指南（含 HotGo 落地对照）

> 本文档是对 **GoFrame 官方文档**（goframe.org，2.10.x）的系统学习笔记，并逐条对照本仓库 HotGo 的实现。
> 目标：读完之后能按框架的"原生范式"写代码，而不是凭经验猜。
> 配套阅读：[PROJECT_ANALYSIS.md](PROJECT_ANALYSIS.md) · [ARCHITECTURE-BACKEND.md](ARCHITECTURE-BACKEND.md) · [MODULES.md](MODULES.md) · [ADDONS-AND-CODEGEN.md](ADDONS-AND-CODEGEN.md) · [`AGENTS.md`](../AGENTS.md)

---

## 0. 阅读范围与来源

| 页面 | URL | 获取情况 |
|------|-----|----------|
| 下载与使用 | `https://goframe.org/quick/install` | ✅ 正文（内容较薄：`go mod init` + `go get -u -v github.com/gogf/gf/v2`；页面标注最低 Go ≥ 1.20） |
| Hello World | `/quick/hello-world` | ✅ 正文（`g.Server()` 单例 / `BindHandler` / `SetPort` / `Run`，含路由表字段释义） |
| 请求数据结构 | `/quick/request-struct` | ✅ 正文（`r.Parse(&req)` 结构化接参，手写 `if` 校验的弊端 → 引出规范路由） |
| 中间件初试用 | `/quick/middleware` | ✅ 正文（完整代码） |
| 项目脚手架 | `/quick/scaffold` | ⚠️ 索引页（子页 URL 未探到） |
| 路由注册（分类总览） | `/docs/web/router-registering` | ✅ 总览 |
| 路由注册-分组路由 | `/docs/web/router-registering-group` | ✅ 正文（完整代码） |
| 路由注册-规范路由 | `/2.0.x/.../路由注册-规范路由` | ✅ 正文（**2.0.x 版本**页面；2.10.x 对应 URL 未探到） |
| 接口文档-OpenAPIv3 | `/2.0.x/.../接口文档-OpenAPIv3` | ✅ 含**标签全表**（代码示例抓取缺失） |
| 数据库 ORM | `/docs/core/gdb` | ⚠️ 栏目索引页（特性级） |
| 配置管理 | `/docs/core/gcfg` | ⚠️ 栏目索引页（特性级） |
| 数据校验 | `/docs/core/gvalid` | ⚠️ 栏目索引页（特性级） |
| 错误处理 | `/docs/core/gerror` | ⚠️ 栏目索引页（特性级） |
| 服务配置 | `/docs/web/server-config` | ⚠️ 栏目索引页（特性级） |
| 数据校验-校验规则 | `/2.0.x/核心组件-重点/数据校验/数据校验-校验规则` | ✅ 正文（2.0.x，46 条规则含参数与示例） |
| ORM链式操作-写入保存（等） | `/2.0.x/核心组件-重点/数据库ORM/ORM链式操作-重点/ORM链式操作-写入保存` | ✅ 正文（2.0.x） |
| 服务配置-配置文件模板 | `/2.0.x/WEB服务开发/服务配置/服务配置-配置文件模板` | ✅ 正文（完整 YAML 模板） |
| **v2.10.0 源码** | `/data/go/pkg/mod/github.com/gogf/gf/v2@v2.10.0` | ✅ **权威口径**：第 5 节的规则全表、错误码表、链式方法全表、Loader API 均据源码枚举 |

**URL 规律（实测结论）**

- 可用：`/quick/<英文短横线 slug>`（如 `/quick/hello-world`、`/quick/request-struct`、`/quick/middleware`）
- 可用：`/docs/<板块>/<slug>` 的**栏目索引页**（如 `/docs/web/router-registering`、`/docs/core/gdb`、`/docs/core/gvalid`、`/docs/core/gcfg`、`/docs/core/gerror`）
- **可用（唯一能拿到正文的形式）**：`/2.0.x/<中文路径>` 与 `/1.16.x/<中文路径>`；完整正文 URL 清单可从 `https://goframe.org/sitemap.xml` 一次性取得
- 不可用：`/docs`（403）；`/quick/router`、`/quick/response`、`/quick/whats-next`；`/docs/core/<pkg>/<sub>`（如 `/docs/core/gvalid/rules`、`/docs/core/gcfg/loader`、`/docs/core/gdb/chaining`、`/docs/core/gerror/utility`）；`/docs/web/server-config-template`；`/docs/web/router-registering-standard-router`（均 404）
- **结论**：官网当前站点未公开 2.10.x 的深层 slug，因此**"定义/清单"类内容一律以本地 v2.10.0 源码为准，2.0.x 页面只作语义与示例补充**（第 5 节已逐处标注版本差异）。

> 下文凡标注「**栏目页**」的内容为索引级提炼；标注「★」的小节为本次据源码 + 2.0.x 正文补充的全表级内容。

---

## 1. 五个核心心智模型

1. **`g` 组件 = 便捷耦合层，全部单例**
   `g.Server()`、`g.DB()`、`g.Cfg()`、`g.Log()` 都是单例获取入口；`gdb.New()` / `gcfg.Instance()` 才是不走配置文件、非单例的形态。
   记忆点：**`g.` 是糖，`gxx` 包是本体**——读源码、查 API 要落到 `github.com/gogf/gf/v2/xxx/gyyy`。

2. **一切组件接口化、驱动可替换**
   `gcfg` 默认提供文件系统实现，但接口可替换；`gdb` 自 v2.1 起**驱动与主库解耦**，必须手动匿名导入：
   ```go
   import _ "github.com/gogf/gf/contrib/drivers/mysql/v2"
   ```
   这正是 HotGo `server/main.go` 里那几行 driver import 存在的原因。

3. **`context.Context` 贯穿全链路**
   业务方法第一个参数恒为 `ctx`；需要 `Request` 时用 `g.RequestFromCtx(ctx)` 取回。日志、链路追踪、超时、ORM、缓存都依赖它传递。

4. **规范路由 = 契约即代码即文档**
   `api` 结构体上的标签**同时**定义路由、参数校验、OpenAPI 文档，三者天然同步，从机制上杜绝"文档与代码不一致"。

5. **中间件 + Hook 控制请求生命周期**
   中间件是可中断的洋葱链；Hook 是 Server 级事件挂载点（`BeforeServe` / `AfterOutput` 等）。

---

## 2. 规范路由（核心，必须刻进肌肉记忆）

### 2.1 方法签名（四个参数一个都不能少）

```go
func Handler(ctx context.Context, req *Request) (res *Response, err error)
```

| 参数 | 说明 | 约束 |
|------|------|------|
| `ctx context.Context` | 上下文 | Server 自动从请求构造并传入 |
| `req *Request` | 请求对象 | **即使无参数也必须定义**——它同时承载接口定义 |
| `res *Response` | 返回对象 | **即使无返回也必须定义**——它同时承载返回定义 |
| `err error` | 错误对象 | Server 通过它判断接口成功/失败 |

### 2.2 Req / Res 结构体规范

- 命名：输入 `XxxReq`，输出 `XxxRes`；
- **即使为空也要定义**——一是便于后续扩展，二是便于接口信息管理；
- 从 v2.0 起规范路由提供 8 项能力：结构化 API 设计、统一方法参数风格、简化路由维护、统一返回格式、**代码与文档同步**、自动参数对象化接收与校验、自动生成 OpenAPIv3 文档、自动生成 SwaggerUI。

### 2.3 `g.Meta` 与 OpenAPIv3 标签全表

元数据以 **embedded** 方式嵌入输入结构体：

```go
type HelloReq struct {
    g.Meta `path:"/hello" method:"get" tags:"示例" summary:"打个招呼"`
    Name   string `v:"required" dc:"Your name"`
}
type HelloRes struct {
    Reply string `dc:"Reply content"`
}
```

**接口级标签（写在 `g.Meta` 上）**

| 标签 | 含义 | 备注 |
|------|------|------|
| `path` | 接口路径 | 与注册时的分组前缀共同构成 URI |
| `method` | `GET/PUT/POST/DELETE...` | 不区分大小写 |
| `tags` | 接口所属标签，用于分类 | |
| `deprecated` | 标记接口废弃 | |
| `mime` | 如 `multipart/form-data` | 一般全局设置，默认 `application/json` |

**参数/字段级标签**

| 标签 | 缩写 | 含义 |
|------|------|------|
| `summary` | `sm` | 概要描述 |
| `description` | `dc` | 详细描述 |
| `default` | `d` | 默认值 |
| `in` | — | 参数提交方式：`header` / `path` / `query` / `cookie` |
| `type` | — | 参数类型，特殊参数才需手动设置，例如 `file` |
| `v` | — | 校验规则（`gvalid` 规则，见第 5 节） |

> 大部分标签由 Server 自动生成，**开发者需要手写的并不多**；更完整的 OpenAPIv3 标签见 <https://swagger.io/specification/>。
> 需要进一步增强文档（自定义排序、补充描述）时，用 `s.GetOpenApi()` 拿到 OpenAPIv3 结构体对象手动填充，甚至据此生成自定义文档。

### 2.4 参数绑定规则

- 请求参数自动转换到请求结构体；
- **字段映射不区分大小写，并自动忽略特殊字符**（`user_name` / `userName` / `UserName` 均可绑定到 `UserName`）；
- 名称映射细节属于「类型转换组件」，可进一步查阅 `gconv`。

### 2.5 自动校验

- 请求结构体在进入 API 方法**之前**自动执行校验；
- **一条规则失败即终止后续规则校验**；
- 校验使用框架统一的 `gvalid` 组件（见第 5 节）。

### 2.6 路由注册（分组 + Bind）

```go
s.Use(ghttp.MiddlewareHandlerResponse)
s.Group("/", func(group *ghttp.RouterGroup) {
    group.Bind(new(Hello))
})
```

要点：

- **路由地址与请求方式由 `g.Meta` 决定**，注册动作只负责"挂载对象"与"设置前缀"；
- 推荐用**对象化方式**管理路由方法，再通过分组 `Bind` 统一注册。

### 2.7 统一返回结构

- 接口的数据返回处理**需要设置统一的后置中间件**；框架提供默认实现 `ghttp.MiddlewareHandlerResponse`；
- 自定义中间件里用 `r.GetHandlerResponse()` 获取业务执行结果（其返回结构为 `{ Object interface{}; Error error }`）；
- 默认返回体为 `{code, message, data}` 形态；若希望 `data` 直接是数组（而非带键名的对象），官方给的方案是**使用类型别名**。

### 2.8 Context 中取 Request

```go
func RequestFromCtx(ctx context.Context) *Request
// 用法
g.RequestFromCtx(ctx).Response.Writeln("Hello World!")
```

### 2.9 常见问题（官方明确结论）

| 问题 | 官方结论 |
|------|----------|
| 同一接口支持多种 HTTP Method？ | **不支持**。一个接口只做一件事，HTTP Method 是有意义的；OpenAPI 协议也不支持 `ALL` 语义。确需 `ALL` 则放弃规范路由，改用普通路由 `func(r *ghttp.Request)`（两种方式可混用，但不推荐） |
| 如何让默认返回结构的 `data` 直接是数组？ | 使用**类型别名** |

---

## 3. 路由注册的四种模式与分组路由

四种模式：**函数注册**、**对象注册**、**分组路由**、**规范路由**（v2 起业务项目推荐后两种）。

### 3.1 分组路由 API

```go
// 创建分组路由
func (s *Server) Group(prefix string, groups ...func(g *RouterGroup)) *RouterGroup

// Method 路由
func (g *RouterGroup) ALL(pattern string, object interface{}, params ...interface{})
func (g *RouterGroup) GET(pattern string, object interface{}, params ...interface{})
func (g *RouterGroup) PUT(pattern string, object interface{}, params ...interface{})
func (g *RouterGroup) POST(pattern string, object interface{}, params ...interface{})
func (g *RouterGroup) DELETE(pattern string, object interface{}, params ...interface{})

// 中间件
func (g *RouterGroup) Middleware(handlers ...HandlerFunc) *RouterGroup

// 批量注册
func (g *RouterGroup) Map(m map[string]interface{})      // 需按 "GET: /user" 规范写
func (g *RouterGroup) ALLMap(m map[string]interface{})   // 应用到所有 HTTP Method
```

- `prefix` 传 `/` 表示注册在根路由下；
- `REST` 方法用于注册 RESTful 风格路由（需给对象/控制器）；
- 分组路由是官方推荐的业务项目路由方式。

### 3.2 层级注册（灵感来自 Laravel，官方推荐）

```go
s.Use(MiddlewareLog)                        // 全局中间件
s.Group("/api.v2", func(group *ghttp.RouterGroup) {
    group.Middleware(MiddlewareAuth, MiddlewareCORS)   // 分组中间件
    group.GET("/test", func(r *ghttp.Request) { r.Response.Write("test") })
    group.Group("/order", func(group *ghttp.RouterGroup) {
        group.GET("/list", ...)
        group.PUT("/update", ...)
    })
    group.Group("/hook", func(group *ghttp.RouterGroup) {
        group.Hook("/*", ghttp.HookBeforeServe, func(r *ghttp.Request) { ... })
        group.Hook("/:name", ghttp.HookBeforeServe, func(r *ghttp.Request) { ... })
    })
})
```

**自查手段**：分组中间件是否生效，直接看启动打印的路由表 `MIDDLEWARE` 列（`DumpRouterMap: true`）。

### 3.3 批量注册

```go
group.Map(g.Map{
    "GET:    /user": UserGet,
    "DELETE: /user": UserDelete,
})
group.ALLMap(g.Map{ "/": api.Index, "/login": api.Login })
```

---

## 4. 中间件模型

### 4.1 定义与执行模型

```go
func Middleware(r *ghttp.Request) {
    // 前置中间件处理逻辑
    r.Middleware.Next()
    // 后置中间件处理逻辑
}
```

- 中间件定义与普通路由函数一样，但可通过 `r.Middleware` 属性控制请求流程；
- 分**前置**（路由函数调用前）与**后置**（其后）两类；
- 执行顺序：**前置 → `Next()` → 路由函数 → 后置**（即常说的洋葱模型）。

### 4.2 `r.Middleware.Next()` 的作用

1. **推进流程**：进入下一层中间件或路由函数；
2. **不调用即中断**：直接退出可以终止后续流程——**这正是鉴权中间件的标准写法**。

### 4.3 错误处理中间件示例（官方代码）

```go
func ErrorHandler(r *ghttp.Request) {
    r.Middleware.Next()                       // 先执行路由函数
    if err := r.GetError(); err != nil {      // 再判断是否产生错误
        r.Response.Write("error occurs: ", err.Error())
        return
    }
}
```

### 4.4 全局 vs 分组

| 方式 | 写法 | 作用范围 |
|------|------|----------|
| 全局中间件 | `s.Use(MiddlewareLog)` | 所有路由 |
| 分组中间件 | `group.Middleware(MiddlewareAuth, MiddlewareCORS)` | 该分组下所有路由 |

---

## 5. 核心组件要点

> **本节来源与版本口径**：以 **v2.10.0 源码**（`/data/go/pkg/mod/github.com/gogf/gf/v2@v2.10.0`）为权威依据，官方文档页面用于补充语义与示例。
> 官网 2.10.x 的深层子页 slug 未公开（实测 `/docs/core/gvalid/rules`、`/docs/core/gcfg/loader`、`/docs/core/gdb/chaining`、`/docs/web/server-config-template` 等均 404），因此正文页取自官网 **2.0.x** 分支（`/2.0.x/核心组件-重点/...`、`/2.0.x/WEB服务开发/...`），**凡与源码冲突处以源码为准**，差异已逐条标注。

---

### 5.1 配置管理 `gcfg`

#### 5.1.1 特性与文件约定

- 接口化设计，**所有方法并发安全**；默认提供文件系统实现（`AdapterFile`），可注入自定义实现（etcd/Apollo 等）。
- 7 项特性：多格式（`yaml`/`toml`/`json`/`xml`/`ini`/`properties`）、配置缺失时回退**环境变量或命令行参数**、可检索读取资源管理组件（gres）中的配置、**配置文件自动检测热更新**、层级访问、单例模式。
- **格式按后缀自动识别**：业务模板默认给 `config.yaml`，可直接改名为 `config.toml` / `config.ini`，组件能识别。
- 核心接口（源码 `os/gcfg/gcfg_adaper.go`）：
  ```go
  type Adapter interface {
      Available(ctx context.Context, resource ...string) (ok bool)
      Get(ctx context.Context, pattern string) (value any, err error)  // "x.y.z" / "x.0.y"
      Data(ctx context.Context) (data map[string]any, err error)
  }
  type WatcherAdapter interface {
      AddWatcher(name string, fn WatcherFunc)
      RemoveWatcher(name string)
      GetWatcherNames() []string
      IsWatching(name string) bool
  }
  ```

#### 5.1.2 配置对象与自动检索

- `g.Cfg()` / `gcfg.Instance()`：单例获取；常量 `DefaultInstanceName = "config"`、`DefaultConfigFileName = "config"`。
- **单例名即文件名**：`gcfg.Instance("myapp")` 会去找 `myapp.<ext>`（如 `myapp.yaml`），因此天然支持多配置文件。
- 检索目录顺序（源码 `AdapterFile.doGetFilePath`）：
  1. 命令行/环境 `gf.gcfg.path` 指定的目录；否则自动追加 `gfile.Pwd()`、`gfile.MainPkgPath()`、`gfile.SelfDir()`；
  2. 无后缀时依次尝试 `name.toml`、`name.yaml`、`name.yml`、`name.json`、`name.ini`、`name.xml`、`name.properties`；
  3. 检索路径：`""`、`"/"`、`"config/"`、`"manifest/config/"`…；
  4. **优先查 gres 资源管理器**，再查本地文件系统。
- `SetViolenceCheck(true)`：当 key 中含层级分隔符 `.` 时开启"暴力"层级冲突检查（性能开销大，默认关闭）。

#### 5.1.3 常用方法

| 方法 | 语义 |
|------|------|
| `Get(ctx, pattern, def...)` | 层级访问（分隔符 `.`，实际委托 `gjson.Get`）；不存在且有 `def` 返回 `gvar.New(def[0])`，否则 `nil` |
| `GetWithEnv(ctx, key, def...)` | 先 `Get`，**仅当错误为 `gcode.CodeNotFound`** 才回退到环境变量（键名 `GF_PACKAGE_VARIABLE` 大写） |
| `GetWithCmd(ctx, key, def...)` | 同上，回退到命令行选项（键名小写 `gf.package.variable`） |
| `MustGet` / `MustGetWithEnv` / `MustGetWithCmd` | 失败即 panic |
| `Data(ctx)` | 返回全部配置的 `map[string]any` |
| `Available(ctx, resource...)` | 判断配置资源是否存在 |
| `SetAdapter` / `GetAdapter` | 替换/获取适配器（`SetAdapter` **非并发安全**） |
| `Clear()` | 清空配置缓存并通知 watcher（`OperationClear`） |

**加载缓存与热更新**（源码 `AdapterFile.getJson`）：`jsonMap *gmap.KVMap[string,*gjson.Json]` 按文件名缓存；文件发生写/删/建/改名/chmod 时 `jsonMap.Remove(...)` 清缓存 + `notifyWatchers`，下次读取自动重载。watcher 通知**每个 watcher 独立 goroutine 执行并带 panic recover**，互不影响。

#### 5.1.4 配置监听回调

实现 `WatcherAdapter` 后，通过 `AddWatcher(name, fn)` 注册；`WatcherFunc` 会拿到 `Operation`（`Set`/`Delete`/`Rename`/`Chmod`/`Clear`…）与文件路径。

#### 5.1.5 Loader 加载器（v2.10 新特性）★

源码 `os/gcfg/gcfg_loader.go`，注释明确对标 **Spring Boot 的 `@ConfigurationProperties`**。完整公开 API：

| 方法 | 说明 |
|------|------|
| `NewLoader[T](config *Config, propertyKey string, targetStruct ...*T) *Loader[T]` | 创建；`propertyKey` 传 `""` 或 `"."` 表示绑定全量配置 |
| `NewLoaderWithAdapter[T](adapter Adapter, propertyKey string, targetStruct ...*T)` | 用自定义 adapter 创建 |
| `Load(ctx) error` / `MustLoad(ctx)` | 一次性加载绑定 |
| `Watch(ctx, name) error` / `MustWatch(ctx, name)` | 注册监听（**要求 adapter 实现 `WatcherAdapter`**） |
| `MustLoadAndWatch(ctx, name)` | 加载 + 监听一体（最常用） |
| `Get() T` / `GetPointer() *T` | 线程安全读取（`GetPointer` 返回指针） |
| `OnChange(fn func(updated T) error) *Loader[T]` | 变更回调 |
| `SetConverter(converter func(data any, target *T) error) *Loader[T]` | 自定义转换器（默认走 `gvar.Var.Scan(&T)`） |
| `SetWatchErrorHandler(errorFunc func(ctx, err)) *Loader[T]` | 监听出错处理（默认走 `intlog`） |
| `SetReuseTargetStruct(reuse bool) *Loader[T]` | 是否复用同一 struct（**默认 `false`：每次变更新建 struct 再原子替换，避免数据竞争**） |
| `StopWatch(ctx) (bool, error)` / `IsWatching() bool` | 停止/查询监听状态 |

```go
type AppConfig struct {
    Name string `yaml:"name"`
    Port int    `yaml:"port"`
}
loader := gcfg.NewLoader[AppConfig](gcfg.Instance(), "app")
loader.MustLoadAndWatch(ctx, "app-config-watcher")
cfg := loader.Get()      // 线程安全副本；配置变更后自动刷新
```

---

### 5.2 数据校验 `gvalid`

#### 5.2.1 规则体系总览

- 8 项特性：内置规则、Server 层与命令行组件**自动化校验**、基础类型与复杂对象校验、顺序校验与灵活结果处理、自定义错误提示、**结构体属性递归校验**、自定义校验规则、i18n 国际化。
- 规则分两类：**修饰规则**（modifier）与**功能规则**（rule）。
- 结构体 tag：**只识别 `v` 与 `valid`**（别名 `param` / `p`）；**v2 不识别 `gvalid` tag**（v1 遗留）。
- tag 值语法：`[别名@]rule1|rule2:arg#消息1|消息2`，多条规则用 `|` **按书写顺序**执行；单规则解析正则为 `^([\w-]+):{0,1}(.*)`。
- **联合校验场景下，关联字段名自动按"不区分大小写 + 忽略特殊字符"智能匹配**（官方导语明确）。

#### 5.2.2 内置规则全表（v2.10.0 源码口径，共 67 条）★

> 名称取自源码注册表（`util/gvalid/internal/builtin/builtin_*.go`，共 67 个文件，每个规则一个文件）；tag 中多词规则用**连字符**形式。
> ⚠️ 官网 2.0.x 文档仅列出 **46 条**，其中**没有 ** `foreach`、`array`、`eq`/`not-eq`、`gt`/`gte`/`lt`/`lte`、`enums`、`alpha`/`alpha-num`/`alpha-dash`、`numeric`、`lowercase`/`uppercase`、`before`/`after`/`before-equal`/`after-equal`、`not-regex` 等——这些是后续版本新增，**以源码列表为准**。

**修饰规则（3 条）**

| 规则 | 说明 |
|------|------|
| `bail` | 后续规则中任一校验失败即**停止并立即返回** |
| `ci` | Case Insensitive，用于 `same` / `different` / `in` / `not-in` / `eq` 等比较类规则时忽略大小写 |
| `foreach` | 对数组/切片元素**逐个应用**规则（2.0.x 文档未覆盖） |

**必需类（8 条）**

| 规则 | 格式 | 语义 |
|------|------|------|
| `required` | `required` | 必需（支持字符串、Slice、Map） |
| `required-if` | `required-if:field,value,...` | 当给定字段值**等于**所给值时必需 |
| `required-if-all` | `required-if-all:field,value,...` | 所有条件都满足时必需 |
| `required-unless` | `required-unless:field,value,...` | 当字段值**不等于**所给值时必需（可多组 `field,value`） |
| `required-with` | `required-with:field1,field2,...` | 当任意给定字段**非空**时必需 |
| `required-with-all` | `required-with-all:field1,...` | 当所有给定字段**全部非空**时必需 |
| `required-without` | `required-without:field1,...` | 当任意给定字段**为空**时必需 |
| `required-without-all` | `required-without-all:field1,...` | 当所有给定字段**全部为空**时必需 |

**长度 / 数值（14 条）**

| 规则 | 格式 | 语义 |
|------|------|------|
| `size` | `size:size` | 长度恰好为 `size`（Unicode 计数，一个汉字=1） |
| `length` | `length:min,max` | 长度在 `min`~`max` |
| `min-length` / `max-length` | `min-length:min` / `max-length:max` | 长度下限/上限 |
| `min` / `max` | `min:min` / `max:max` | 数值下限/上限（支持整型与浮点） |
| `between` | `between:min,max` | 数值区间 |
| `eq` / `not-eq` | `eq:value` | 等于 / 不等于（**2.0.x 文档未覆盖**） |
| `gt` / `gte` / `lt` / `lte` | `gt:value` | 大于 / 大于等于 / 小于 / 小于等于（**文档未覆盖**） |

**类型（9 条）**

| 规则 | 语义 |
|------|------|
| `integer` | 整数（正整数或负整数，`10.0` 不通过） |
| `float` | 浮点数 |
| `boolean` | 布尔（`1/true/on/yes` 为真；`0/false/off/no/""` 为假） |
| `numeric` | 数字（文档未覆盖） |
| `array` | 数组类型（文档未覆盖） |
| `json` | 合法 JSON 字符串 |
| `alpha` / `alpha-num` / `alpha-dash` | 纯字母 / 字母数字 / 字母数字下划线连字符（文档未覆盖） |
| `lowercase` / `uppercase` | 全小写 / 全大写（文档未覆盖） |

**格式 / 业务（23 条）**

| 规则 | 格式 / 语义 |
|------|-------------|
| `date` | 常用日期：`2006-01-02`、`2006/01/02`、`2006.01.02`、`20060102` |
| `datetime` | 日期时间，分隔符**只支持 `-`**：`2006-01-02 12:00:00` |
| `date-format` | `date-format:Y-m-d H:i:s`（`gtime` 格式串） |
| `email` | 邮箱 |
| `phone` / `phone-loose` | 大中国区手机号（严格/宽松：宽松只要 13–19 开头 11 位） |
| `telephone` | 座机（`XXXX-XXXXXXX` 等 6 种形态） |
| `postcode` | 邮政编码（6 位） |
| `passport` | 通用账号（字母开头 + 字母/数字/下划线，6~18 位） |
| `password` / `password2` / `password3` | 通用（任意可见字符 6~18）/ 中强（含大小写+数字）/ 强（再加特殊字符） |
| `resident-id` | 公民身份证号 |
| `bank-card` | 银行卡号 |
| `qq` | QQ 号（≥10000 且全数字） |
| `ip` / `ipv4` / `ipv6` | IP 地址 |
| `mac` | MAC 地址 |
| `url` | URL（`http`/`https`/`ftp`/`file` 开头） |
| `domain` | 域名（`xxx.yyy`，首字符须为字母） |
| `before` / `after` / `before-equal` / `after-equal` | 日期先后比较（文档未覆盖） |

**集合 / 正则 / 比较（9 条）**

| 规则 | 格式 | 语义 |
|------|------|------|
| `in` | `in:value1,value2,...` | 值在允许集合内（字符串匹配，可配 `ci`） |
| `not-in` | `not-in:value1,...` | 值不在集合内 |
| `enums` | `enums` | 枚举校验（配合 `gtag` 全局枚举，文档未覆盖） |
| `regex` | `regex:pattern` | 正则匹配 |
| `not-regex` | `not-regex:pattern` | 正则不匹配（文档未覆盖） |
| `same` | `same:field` | 与指定字段值相同（如确认密码） |
| `different` | `different:field` | 与指定字段值不同 |

#### 5.2.3 校验对象 `Validator`

源码 `util/gvalid/gvalid_validator.go`；**每个链式方法都会 `Clone()` 后改字段**（同 `glog` 的克隆语义）。

| 方法 | 说明 |
|------|------|
| `New()` | 创建（默认使用全局 i18n） |
| `Data(data)` | 设置被校验数据（`Map` / `Struct` / 基础类型） |
| `Rules(rules)` | 显式规则（优先级高于结构体 tag） |
| `Assoc(data)` | 设置**关联数据**，并置 `useAssocInsteadOfObjectAttributes = true`（解决 struct 默认值干扰 `required` 的官方方案之一） |
| `Messages(msg)` | 自定义错误消息 |
| `RuleFunc(rule, f)` / `RuleFuncMap(map)` | **实例级**自定义规则 |
| `Bail()` | 首错即停 |
| `Ci()` | 忽略大小写 |
| `Foreach()` | 对数组元素逐个校验 |
| `I18n(manager)` | 指定 i18n 管理器 |
| `Run(ctx)` | 执行校验，**返回 `gvalid.Error`**（该接口含 `Error() string`，因此可直接当 `error` 用） |

> 完整签名以源码为准（`gvalid_validator.go`）：`Run` / `Clone` / `I18n` / `Bail` / `Foreach` / `Ci` / `Data` / `Assoc` / `Rules` / `Messages` / `RuleFunc` / `RuleFuncMap`；数据形态与分派见 §5.2.5。`ghttp` 参数自动校验即调用 `gvalid.New().Bail().Data(pointer).Assoc(data).Run(ctx)`。

#### 5.2.4 校验结果 `gvalid.Error`

```go
type Error interface {
    Code() gcode.Code
    Current() error
    Error() string
    FirstItem() (key string, item map[string]error)
    FirstRule() (rule string, err error)
    FirstError() error
    Items() []map[string]map[string]error
    Map() map[string]error
    Maps() map[string]map[string]error
    String() string
    Strings() []string
}
```

- 实现体 `validationError{code, rules, errors, firstKey, firstItem}`；**保序**（`Strings()` 按规则书写顺序返回），并按 `|`/`:` 拆分规则名。
- `Code()` 固定返回 `gcode.CodeValidationFailed`；内部错误为 `CodeInternalError`。
- 构造每条错误时使用 `gerror.NewWithOption(Option{Stack:false, ...})`——**刻意不抓栈**（高频路径）。
- 取首条错误常用 `gerror.Current(err)` 或 `err.FirstError()`。

#### 5.2.5 三种校验数据形态（`Run` 的分派）

`Validator.Run(ctx)` 的返回类型是 **`gvalid.Error`**（它自带 `Error() string`，因此**可以直接当 `error` 用**，无需类型断言），内部按 `reflect.Kind` 三分支（`gvalid_validator.go:44-80`）：

| 数据形态 | `Data()` 传入 | 分派 | 规则来源与要点 |
|---------|---------------|------|----------------|
| 单值 | `10`、`"john"`、`[]int{1,2}` | `doCheckValue` | **必须显式传 `Rules(...)`**，否则无规则可校验 |
| Map | `map[string]any` | `doCheckMap` | 键即字段名；`Assoc` 数据同时参与联合校验 |
| Struct | `*struct` / `struct` | `doCheckStruct` | 从 `v`/`valid` tag 解析规则；**带默认值的字段会误触发 `required`**，用指针或 `Assoc` 规避 |

```go
// 单值：必须给规则
gvalid.New().Rules("between:1,100").Data(age).Run(ctx)

// Map
gvalid.New().Data(g.Map{"name": "john", "age": 18}).Run(ctx)

// Struct + 关联数据（ghttp 内部就是这一形态）
gvalid.New().Bail().Data(req).Assoc(data).Run(ctx)
```

#### 5.2.6 可选校验 / 递归校验 / 跳过校验

| 主题 | 结论（源码位置） |
|------|-----------------|
| **可选校验** | 规则中**不含 `required*` 类**时，`nil` / 空字符串**跳过**校验（`gvalid_validator_check_struct.go`、`_check_map.go`）；**但 `0` 与 `false` 依然会被校验** |
| **递归校验** | 支持 struct / slice / map 自动递归（`doCheckValueRecursively`）；**空指针对象不触发递归**，带默认值的空结构体被视为已传递并校验 |
| **跳过整个字段** | tag `nv`（`gtag.NoValidation`）或该类型实现 `NoValidation()` 接口 |
| **required 语义扩展** | 源码 `Validator.checkRuleRequired(rule)`：**`required*` 前缀 或 自定义规则，一律视为 required**——即用了自定义规则，空值也会进入校验，不会被"空值跳过"绕过 |

> **写自定义规则的直接后果**：只要字段上挂了自定义规则，它就成了"必需校验"语义。若希望空值仍跳过，需在规则函数内自行判断 `in.Value == nil || in.Value.String() == ""` 后返回 `nil`。

#### 5.2.7 自定义规则（完整示例）★

**注册 API**（源码 `util/gvalid/gvalid_register.go`）

| API | 说明 |
|-----|------|
| `type RuleFunc func(ctx context.Context, in RuleFuncInput) error` | 规则函数签名（`ctx` 首参） |
| `RegisterRule(rule string, f RuleFunc)` | 全局注册；**重复注册同一 rule 只会在 intlog 里提示 `overwritten`，不报错** |
| `RegisterRuleByMap(m map[string]RuleFunc)` | 批量全局注册 |
| `GetRegisteredRuleMap() map[string]RuleFunc` | 取出全部已注册自定义规则（**返回副本**；无则 `nil`） |
| `DeleteRule(rules ...string)` | 删除全局自定义规则 |
| `Validator.RuleFunc(rule string, f RuleFunc)` | **实例级**注册（只影响当前 Validator 及其 `Clone`） |
| `Validator.RuleFuncMap(m map[string]RuleFunc)` | 实例级批量注册 |

**`RuleFuncInput` 字段全表**（`gvalid_register.go:23-42`）

| 字段 | 类型 | 说明 |
|------|------|------|
| `Rule` | `string` | 完整规则串，如 `between:1,100`（参数要自己解析） |
| `Message` | `string` | 用户显式配置的消息（tag 的 `#msg` 或 `Messages`），**可能为空** |
| `Field` | `string` | 当前校验的字段名（含别名） |
| `ValueType` | `reflect.Type` | 值类型，**可能为 nil**（未传值时） |
| `Value` | `*gvar.Var` | 当前字段值，**可能为 nil**，取值前先判空 |
| `Data` | `*gvar.Var` | 传给 `Validator` 的整个数据（map/struct），**不需要时可忽略** |

```go
// 1) 全局注册：只应在 init() / 启动期做（见下方"无锁"警告）
func init() {
    gvalid.RegisterRule("my-mobile", func(ctx context.Context, in gvalid.RuleFuncInput) error {
        // 值可能为 nil，先保护
        if in.Value == nil || gstr.IsNumeric(in.Value.String()) {
            return nil
        }
        // 用户配置了消息则优先用用户的
        if in.Message != "" {
            return gerror.New(in.Message)
        }
        return gerror.Newf("参数 %s 的值 %s 不是合法手机号", in.Field, in.Value)
    })
}

type BizReq struct {
    Mobile string `v:"required|my-mobile#手机号格式不正确"`
}

// 2) 实例级注册（仅本次校验生效）
err := gvalid.New().
    RuleFunc("my-rule", func(ctx context.Context, in gvalid.RuleFuncInput) error {
        return nil
    }).
    Data(req).
    Run(ctx)

// 3) 实例级批量
gvalid.New().RuleFuncMap(map[string]gvalid.RuleFunc{
    "rule-1": func(ctx context.Context, in gvalid.RuleFuncInput) error { return nil },
}).Data(req).Run(ctx)
```

**四条必须知道的语义**

1. `Value` / `Data` / `ValueType` 都**可能是 `nil`**，规则函数内必须做空值保护，否则 panic。
2. **自定义规则默认按 `required` 处理**（见 §5.2.6），空值也会被调用。
3. `Message` 为空时**必须由规则函数自己给出默认文案**，否则错误消息会退化成 `__default__` 的泛化提示。
4. 全局注册表 `customRuleFuncMap` 是**包级普通 map，没有任何锁**——**严禁在运行期并发注册**，只能在 `init()`/启动期一次性完成。

#### 5.2.8 自定义错误消息与模板占位符 ★

**三种配置方式**

```go
// ① tag 内消息（多规则时用 | 按顺序对应，缺省段留空）
type Req struct {
    Name     string `v:"name@required|length:2,20|password3|same:password1#||密码强度不足|两次密码不一致"`
    Password string `v:"required#密码不能为空"`
}

// ② Messages()：map[field]string 或 map[field]map[rule]string
gvalid.New().Messages(g.Map{
    "name": g.Map{"required": "名字不能为空", "length": "名字长度需在 2~20"},
}).Data(req).Run(ctx)

// ③ 规则函数内返回 gerror（见 §5.2.7）
```

**消息解析优先级**（源码 `gvalid_validator_message.go` 的 `getErrorMessageByRule`）

1. **自定义消息**（`Messages` 或 tag 的 `#msg`）——注意自定义消息会**先经 i18n 翻译**，命中则用翻译结果；
2. i18n 中 `gf.gvalid.rule.<rule>` 的配置；
3. 内置规则的 `Message()` 默认文案（如 `The {field} field is required`）；
4. 兜底 `__default__`：`The {field} value \`{value}\` is invalid`。

**模板占位符**：`{field}`（字段名/别名）、`{value}`（当前值），部分规则另有专属占位符，例如 `size` 的 `{size}`、`length`/`between` 的 `{min}`/`{max}`。所以自定义消息写成 `"名字长度需在 {min}~{max}"` 也能被填充。

**错误码与内部错误**

| 场景 | Code | 说明 |
|------|------|------|
| 业务校验失败 | `gcode.CodeValidationFailed`（51） | `validationError.code` 固定值 |
| 规则串非法 / 参数非法 / 对象非法 | `gcode.CodeInternalError`（50） | 由 `newValidationErrorByStr` 构造，key 固定为 **`__InternalError__`**（`internalErrorMapKey`），规则名分别是 `InvalidRules` / `InvalidParams` / `InvalidObject` |

> 另有 `CustomMsg = map[string]any` 的类型别名（`map[field] => string | map[rule]string`），`Messages()` 的参数直接用别名即可。

#### 5.2.9 完整示例

```go
type BizReq struct {
    ID       uint   `v:"required" dc:"Your ID"`
    Name     string `v:"required" dc:"Your name"`
    Gender   uint   `v:"in:0,1,2" dc:"0:Secret;1:Male;2:Female"`
    WifeName string `v:"required-if:gender,1"`
    Account  string `v:"bail|required|length:6,16|same:QQ"`
    Password string `v:"required|ci|same:Password2"`
    Password2 string `v:"required"`
}

// Run 直接返回 gvalid.Error（已实现 error），无需类型断言
if err := g.Validator().Data(req).Run(ctx); err != nil {
    fmt.Print(gstr.Join(err.Strings(), "\n"))
}

// 若只想给用户看第一条错误
if err := g.Validator().Bail().Data(req).Run(ctx); err != nil {
    return nil, gerror.New(err.FirstError().Error())
}
```

典型输出：`The Account value \`gf\` length must be between 6 and 16`（`bail` 生效后立即返回，不再校验 `same:QQ`）。

> **易错点**：`Run(ctx)` 的返回类型是 `gvalid.Error` 而非 `error`，所以 `err.Strings()` / `err.FirstError()` / `err.Maps()` **可直接调用**——网上常见的 `err.(gvalid.Error).Strings()` 写法是多余的（除非 err 被赋给了 `error` 变量）。

#### 5.2.10 版本差异提示（重要）

| 项 | 2.0.x 文档 | v2.10.0 源码 |
|----|-----------|--------------|
| 内置规则数量 | 46 条 | **67 条** |
| `foreach` 修饰规则 | 未提及 | **已有** |
| `same`/`different`/`in`/`not-in` 的 `ci` | 有 | 有 |
| 规则命名 | 连字符（`max-length`） | 连字符（文件名为下划线） |

---

### 5.3 错误处理 `gerror` + 错误码 `gcode`

#### 5.3.1 常用方法

| 方法族 | 语义 | 是否抓栈 | 错误码 | 带 cause |
|--------|------|----------|--------|----------|
| `New` / `Newf` | 创建新错误 | ✅ | `CodeNil` | ❌ |
| `NewSkip` / `NewSkipf` | 创建并跳过若干栈帧 | ✅ | `CodeNil` | ❌ |
| `Wrap` / `Wrapf` | 包装下层错误成链 | ✅ | **继承 `Code(err)`** | ✅ |
| `WrapSkip` / `WrapSkipf` | 包装并跳过栈帧 | ✅ | 继承 | ✅ |
| `NewCode` / `NewCodef` / `NewCodeSkip` | 创建并指定错误码 | ✅ | 显式 | ❌ |
| `WrapCode` / `WrapCodef` / `WrapCodeSkip` | 包装并指定错误码 | ✅ | 显式 | ✅ |
| `NewWithOption(Option)` | **高级构造**（框架内部常用） | 仅 `Option.Stack==true` | `Option.Code` | `Option.Error` |

```go
type Option struct {
    Error error      // 被包装错误（cause）
    Stack bool       // 是否抓栈
    Text  string     // 消息模板
    Args  []any      // 模板参数
    Code  gcode.Code // 错误码
}
```

其他：`Cause` / `Current` / `Unwrap`（= `Next()`）/ `Stack` / `HasStack` / `Equal` / `Is` / `As` / `HasError` / `Code` / `HasCode`。

#### 5.3.2 内置错误码全表（源码 `errors/gcode/gcode.go`）★

> 约定：**`code < 1000` 为框架保留**。

| 码 | 常量 | 语义 |
|----|------|------|
| -1 | `CodeNil` | 无错误码 |
| 0 | `CodeOK` | 成功 |
| 50 | `CodeInternalError` | 内部错误 |
| 51 | `CodeValidationFailed` | 校验失败 |
| 52 | `CodeDbOperationError` | 数据库操作错误 |
| 53 | `CodeInvalidParameter` | 参数非法 |
| 54 | `CodeMissingParameter` | 参数缺失 |
| 55 | `CodeInvalidOperation` | 操作非法 |
| 56 | `CodeInvalidConfiguration` | 配置非法 |
| 57 | `CodeMissingConfiguration` | 配置缺失 |
| 58 | `CodeNotImplemented` | 未实现 |
| 59 | `CodeNotSupported` | 不支持 |
| 60 | `CodeOperationFailed` | 操作失败 |
| 61 | `CodeNotAuthorized` | 未授权 |
| 62 | `CodeSecurityReason` | 安全原因 |
| 63 | `CodeServerBusy` | 服务繁忙 |
| 64 | `CodeUnknown` | 未知错误 |
| 65 | `CodeNotFound` | 资源不存在 |
| 66 | `CodeInvalidRequest` | 请求非法 |
| 67 | `CodeNecessaryPackageNotImport` | 必要包未引入 |
| 68 | `CodeInternalPanic` | 内部 panic |
| 300 | `CodeBusinessValidationFailed` | 业务校验失败（≥1000 可自定义） |

`Code` 是接口：`Code() int` / `Message() string` / `Detail() any`；内置实现 `localCode`；工厂 `gcode.New(code, msg, detail)`、`gcode.WithCode(code, detail)`。

> 🔗 **与 HotGo 的呼应**：`61` 正是 `CodeNotAuthorized`，而 HotGo 前端把 `code === 61` 当作"登录失效 → 清 token 跳登录"处理，语义天然对齐。

#### 5.3.3 错误码特性

- **继承与回溯**：`Error.Code()` 在自身为 `CodeNil` 时**沿错误链向上回溯**取第一个非空码：
  ```go
  func (err *Error) Code() gcode.Code {
      if err.code == gcode.CodeNil { return Code(err.Unwrap()) }
      return err.code
  }
  ```
- `gerror.Code(err)`：实现 `ICode` 则取其 `Code()`，否则沿 `Unwrap` 递归；`HasCode(err, code)` 沿链查找。
- **扩展自定义码**：实现 `gcode.Code` 接口（或直接用 `gcode.New`），建议从 1000 起编号（<1000 为框架保留）。
- **错误比较 `Equal`**：先指针相等，再比 **code**，再比 **text**（`TextWithArgs()` 与 `fmt.Sprintf("%-s", target)`）——**比较的是码与文本，不是 cause**。
- `Is` / `As` 直接转发标准库 `errors.Is` / `errors.As`（为对齐 stdlib 性能）。

#### 5.3.4 格式化符号

| 符号 | 输出 |
|------|------|
| `%v` / `%s` | 完整错误文本（含链） |
| `%-v` / `%-s` | **仅当前层**文本（`gvalid` 内部大量使用） |
| `%+s` | 完整调用栈 |
| `%+v` | 错误文本 + `\n` + 完整调用栈 ← **日志打印栈必须用它** |

- 默认 **brief 栈模式**（`internal/errors`），会过滤掉 `github.com/gogf/gf/` 自身的栈帧；环境变量 `GF_GERROR_STACK_MODE` 可切换简要/详细。

#### 5.3.5 最佳实践（含正误对比）

1. **打印栈必须用 `%+v`**：
   - ✅ `g.Log().Errorf(ctx, "%+v", err)`
   - ❌ `g.Log().Errorf(ctx, "%v", err)`（无栈）
   - ❌ `g.Log().Error(ctx, err.Error(), err)`（人工拼接易重复）

2. **`Wrap` 时不要把原错误文本再拼进新消息**：
   - ✅ `return gerror.Wrap(err, "create order failed")`
   - ❌ `return gerror.Wrap(err, "create order failed: "+err.Error())`（输出重复两层）
   - 官方推荐写法等价于 `gerror.Wrapf(err, "create order failed")`。

3. **高频路径避免抓栈**：校验、参数转换等高频错误用 `gerror.NewWithOption(gerror.Option{Stack: false, Text: ..., Code: ...})`。

4. **错码优先**：对外返回错误时显式给码（`gerror.NewCode(gcode.CodeInvalidParameter, "...")`），让上层与前端能稳定判码而不是匹配字符串。

5. **国际化（v2.10.0+）**：实现/使用 `ITextArgs`（`Text() string` / `Args() []any`）把模板与参数分离，交给 `gi18n` 翻译：
   ```go
   err := gerror.NewWithOption(gerror.Option{Text: "user {name} not found", Args: []any{g.Map{"name": "john"}}})
   ```

---

### 5.4 数据库 ORM `gdb`

#### 5.4.1 获取数据库对象的三种方式

- **驱动引入**：自 v2.1 起驱动与主库解耦，需手动匿名导入（`contrib/drivers/mysql/v2` 等），否则运行期报 `CodeInvalidConfiguration`。
- 底层**连接池**自动管理，链接不用时自动关闭，**无需显式 Close**；不建议拼接 SQL，统一用 `?` 占位符预处理（占位符展开与 `gdb.Raw` 见 5.4.4）。

| 方式 | 说明 |
|------|------|
| `g.DB()` ✅ **推荐** | 读 `gcfg` 的 `database` 配置 + gins 单例；**配置缺失会 panic** |
| `gdb.Instance(group...)` | 仅用 gdb 全局配置 + `instances` 单例缓存（按 group） |
| `gdb.NewByGroup(group...)` | 同上但**不缓存** |
| `gdb.New(node)` | 完全显式节点配置，**不使用配置文件、无缓存** |

> 👉 HotGo 的 `handler.Model(dao.Xxx.Ctx(ctx), option...)` 就是 `g.DB()`（经 gins 单例）+ 分组单例 + 链式 Model 的标准组合。

#### 5.4.2 官方文档「链式操作 16 项」↔ 源码方法对照 ★

| 文档章节（ORM链式操作-重点） | 源码文件 | 关键方法 |
|------------------------------|----------|----------|
| 模型创建 | `gdb.go` / `gdb_model.go` | `DB.Model/Table`、`Raw`、`As`、`Schema`、`Clone`、`Safe`、`Ctx`、`Master/Slave` |
| 字段获取 | `gdb_core_utility.go` / `gdb_model_fields.go` | `TableFields()`、`GetFieldsStr`、`GetFieldsExStr`、`HasField` |
| 字段过滤 | `gdb_model_fields.go` / `gdb_model_option.go` | `Fields`、`FieldsEx`、`FieldsPrefix`、`OmitEmpty*`、`OmitNil*` |
| 对象输入 | `gdb_model_insert.go` / `gdb_model_update.go` | `Data`（支持 map / struct / slice） |
| 数据库切换 | `gdb_model.go` + `gdb_core_config.go` | `DB`、`TX`、`Schema`、`Master/Slave`；多 group 配置 |
| 时间维护 | `gdb_model_soft_time.go` | `SoftTime`、`Unscoped`；`created_at`/`updated_at`/`deleted_at` 自动填充 |
| 更新删除 | `gdb_model_update.go` / `gdb_model_delete.go` | `Update`、`UpdateAndGetAffected`、`Increment`、`Decrement`、`Delete` |
| 查询缓存 | `gdb_model_cache.go` | `Cache`、`PageCache`；`Core.ClearCache`/`ClearCacheAll` |
| Handler 特性 | `gdb_model_utility.go` | `Handler(...ModelHandler)`（DB 级 + Model 级函数式改写） |
| Hook 特性 | `gdb_model_hook.go` | `Hook(HookHandler{Select,Insert,Update,Delete})` |
| 主从切换 | `gdb_core_config.go` | 节点 `Role=master/slave` + `Weight`；`Master()`/`Slave()`；读默认走 slave |
| 事务处理 | `gdb_core_transaction.go` / `gdb_model_transaction.go` | `Transaction`、`TransactionWithOptions`、`Begin/SavePoint/RollbackTo` |
| 写入保存 | `gdb_model_insert.go` | `Insert`、`InsertIgnore`、`InsertAndGetId`、`Replace`、`Save`、`Batch`、`OnDuplicate(Ex)`、`OnConflict` |
| 悲观锁 & 乐观锁 | `gdb_model_lock.go` | `Lock`、`LockUpdate`、`LockUpdateSkipLocked`、`LockShared` |
| 数据查询 | `gdb_model_select.go`/`_where*.go`/`_join.go`/`_order_group.go` | `All/One/Array/Value/Count/Scan/ScanList`、`Where*` 全家桶、`LeftJoin/RightJoin/InnerJoin`、`Group/Order/Having`、`Union/UnionAll` |
| 模型关联 | `gdb_model_select.go`、`gdb_model_with.go` | `ScanList(ptr, bindToAttr, relations...)`（动态关联）、`With(...)`（静态关联） |

#### 5.4.3 链式方法全表（源码 `database/gdb/gdb_model*.go`，共 203 个方法）

| 分组（文件） | 公开链式方法 |
|--------------|--------------|
| `gdb_model.go` | `Raw` `Partition` `Ctx` `GetCtx` `As` `DB` `TX` `Schema` `Clone` `Master` `Slave` `Safe` `Args` `Handler` |
| `gdb_model_where.go` | `Where` `Wheref` `WherePri` `WhereLT` `WhereLTE` `WhereGT` `WhereGTE` `WhereBetween` `WhereLike` `WhereIn` `WhereNull` `WhereNotBetween` `WhereNotLike` `WhereNot` `WhereNotIn` `WhereNotNull` `WhereExists` `WhereNotExists` |
| （对称文件） | `WhereOr*` 全家桶、`WherePrefix*`、`WhereOrPrefix*` |
| `gdb_model_fields.go` | `Fields` `FieldsPrefix` `FieldsEx` `FieldsExPrefix` `FieldCount` `FieldSum` `FieldMin` `FieldMax` `FieldAvg` `GetFieldsStr` `GetFieldsExStr` `HasField` |
| `gdb_model_select.go` | `All` `AllAndCount` `Chunk` `One` `Array` `Scan` `ScanAndCount` `ScanList` `Value` `Count` `Exist` `CountColumn` `Min` `Max` `Avg` `Sum` `Union` `UnionAll` `Limit` `Offset` `Distinct` `Page` `Having` |
| `gdb_model_insert.go` | `Data` `Batch` `OnConflict` `OnDuplicate` `OnDuplicateEx` `Insert` `InsertAndGetId` `InsertIgnore` `Replace` `Save` |
| `gdb_model_update.go` | `Update` `UpdateAndGetAffected` `Increment` `Decrement` |
| `gdb_model_delete.go` | `Delete` |
| `gdb_model_join.go` | `LeftJoin` `RightJoin` `InnerJoin` `LeftJoinOnField(s)` `RightJoinOnField(s)` `InnerJoinOnField(s)` |
| `gdb_model_order_group.go` | `Order` `OrderAsc` `OrderDesc` `OrderRandom` `Group` |
| `gdb_model_option.go` | `OmitEmpty` `OmitEmptyWhere` `OmitEmptyData` `OmitNil` `OmitNilWhere` `OmitNilData` |
| `gdb_model_cache.go` | `Cache` `PageCache` |
| `gdb_model_lock.go` | `Lock` `LockUpdate` `LockUpdateSkipLocked` `LockShared` |
| `gdb_model_transaction.go` | `Transaction` `TransactionWithOptions` |
| `gdb_model_soft_time.go` | `SoftTime` `Unscoped` |
| `gdb_model_hook.go` | `Hook` |
| `gdb_model_sharding.go` | `Sharding` `ShardingValue` |

#### 5.4.4 写入保存：5 个方法的差异 ★

| 方法 | 生成 SQL | 主键/唯一索引冲突时 |
|------|----------|---------------------|
| `Insert` | `INSERT INTO` | **返回失败** |
| `InsertIgnore` | `INSERT IGNORE INTO` | 忽略错误，继续执行 |
| `Replace` | `REPLACE INTO` | 删除原有记录，必定写入新记录 |
| `Save` | `INSERT INTO ... ON DUPLICATE KEY UPDATE` | **更新原有数据**（MySQL 自动 `col=VALUES(col)`，会跳过 created 字段） |
| `InsertAndGetId` | `INSERT INTO` | 返回自增 ID（`LastInsertId()`） |

```go
g.Model("user").Data(g.Map{"name": "john"}).Insert()                       // INSERT
g.Model("user").Data(g.Map{"uid": 10000, "name": "john"}).InsertIgnore()   // INSERT IGNORE
g.Model("user").Data(g.Map{"uid": 10000, "name": "john"}).Replace()        // REPLACE
g.Model("user").Data(g.Map{"uid": 10001, "name": "john"}).Save()           // upsert
g.Model("user").Data(g.List{ {"name":"john_1"}, {"name":"john_2"} }).Batch(2).Insert() // 分批批量
g.Model("user").Data(g.Map{"nickname": gdb.Raw("CONCAT('name_', VALUES(`nickname`))")}).Save()
```

补充要点：

- `Data` 支持 **map / struct（推荐带 `orm:"col"` tag）/ slice（批量）**；`OnDuplicate(字段名...或 map)` / `OnDuplicateEx(排除字段)` 需配合 `Save`；**PostgreSQL 需显式 `OnConflict` 指定冲突列**。
- `gdb.Raw("now()")` 是**唯一"直接拼 SQL 片段"通道**（不作为预处理参数），需调用方保证安全。
- `Update`/`Delete` **无 WHERE 直接报错**（`CodeMissingParameter`），不会全表操作。
- 批量插入时 `Core.DoInsert` 会按字段集合**分组**（字段不同的记录拆成多条 SQL），并按表字段顺序排序。

#### 5.4.5 结果处理与「为空判断」★

结果类型（`gdb.go`）：`Value = *gvar.Var`、`Array = gvar.Vars`、`Record = map[string]Value`、`Result = []Record`、`Map = map[string]any`、`List = []Map`、`Raw string`；`SqlResult{Result, Affected}` 用于批量写累加影响行数。

| API | 返回 | 无数据时 |
|-----|------|----------|
| `All` `FindAll` | `Result` | `nil, nil` |
| `One` | `Record` | `nil, nil` |
| `Array` | `Array` | `nil, nil`（结果多列会报 `CodeInvalidParameter`） |
| `Value` | `Value` | `nil, nil` |
| `Count` | `int` | `0, nil` |
| `Exist` | `bool` | `false, nil` |
| `AllAndCount` / `ScanAndCount` | `Result/int` | **count=0 时提前返回，不再查数据** |
| `Scan(&struct)` | `error` | **目标非空 → `sql.ErrNoRows`**；目标为空 slice → `nil` |
| `Scan(&sliceOfLen>0)` | `error` | **`sql.ErrNoRows`** |

**"为空判断"的官方说明**：`gdb` 定义了 `Record.IsEmpty()` 与 `Result.IsEmpty()`；正是这两个方法让 `All/One/Array/Value/Count` 在无数据时返回 `nil/0 + nil error`，而驱动层**永远不会**把 `sql.ErrNoRows` 抛给上层（`DoCommit` 中 `err != sql.ErrNoRows` 才包装）。只有显式 `Scan` 到"非空的具名结构体"才会得到 `sql.ErrNoRows`。

> **实践建议**：列表查询用 `Scan(&list)`（空 slice 得到 `nil`）；详情查询用 `Scan(&one)` 并显式处理 `sql.ErrNoRows`，或改用 `One()` + `Record.IsEmpty()`。

**结果类型常用方法**：`Record.Json/Xml/Map/GMap/Struct/IsEmpty`；`Result.IsEmpty/Len/Size/Chunk/Json/Xml/List/Array/MapKey*/RecordKey*/Structs`。

**字段 → 结构体映射**：默认优先 **`orm` tag**（`OrmTagForStruct = "orm"`），兼容 `gconv` tag；列名与字段名的大小写/下划线由 `gconv` 的"精确优先 + 去符号模糊兜底"完成。**未显式设置 `Fields` 时，`Scan(&struct)` 会自动 `Fields(pointer)`，把 SELECT 列收敛为目标结构体字段**——列名不匹配不会报错，只是"查不到该列"。

#### 5.4.6 事务处理 ★

- **传播类型**（`gdb_core_transaction.go`，`TransactionWithOptions` 可指定）：`NESTED`（**默认**）、`REQUIRED`、`SUPPORTS`、`REQUIRES_NEW`、`NOT_SUPPORTED`、`MANDATORY`、`NEVER`。
- **嵌套用 SavePoint 实现**：`TXCore.transactionCount` 计数，内层 `Commit/Rollback` 变成 `RELEASE SAVEPOINT` / `ROLLBACK TO SAVEPOINT`，**只有最外层才真正 COMMIT/ROLLBACK**；savepoint 名形如 `` `transaction1` ``。
- **事务靠 ctx 传播**（最关键的理解点）：`DoQuery/DoExec/DoPrepare` 在 link 为空或非事务链接时会主动 `TXFromCtx(ctx, group)`，因此：

```go
db.Transaction(ctx, func(ctx context.Context, tx gdb.TX) error {
    // 这里即使写 db.Model("user")... 而不显式用 tx，也会自动入本事务
    _, err := g.Model("user").Ctx(ctx).Data(g.Map{"name": "john"}).Insert()
    return err
})
```

- 禁止在事务闭包内手动 `Commit/Rollback`（`callTxFunc` 统一 recover + 回滚/提交）；**跨 group 不共享事务**，`REQUIRES_NEW` 用 `WithoutTX` 隔离。
- `Model.Transaction` 自身：`m.tx != nil` 时复用当前事务，否则 `m.db.Transaction`。

> 🔗 HotGo 里 logic 层互相调用 service 仍能保持同一事务，正是这条 ctx 传播机制带来的。

#### 5.4.7 高级特性全景 ★

| 特性 | 使用要点 |
|------|----------|
| **时间维护** | 字段候选：`created_at`/`create_at`、`updated_at`/`update_at`、`deleted_at`/`delete_at`；Insert 填充 create+update（+delete 初值），Update 仅在数据中未提供时才补 update；字段值类型由 **DB 字段类型**决定（datetime → `gtime.Now()`，int → 秒级时间戳，bool → 1）；`ConfigNode.TimeMaintainDisabled` 或 `FieldsEx` 命中则跳过；`SoftTimeOption` 可强制 `Time/Timestamp/Milli/Micro/Nano` |
| **软删除** | SELECT 时无 `Unscoped()` 自动拼条件（datetime → `IS NULL`，int/bool → `=0`）；`Delete` 改为 `UPDATE ... SET deleted_at=...` |
| **字段映射** | `orm` tag 优先；参数字段自动识别与过滤（`mappingAndFilterToMap/ToTableFields`） |
| **类型识别** | `CheckLocalTypeForField` 判定 `LocalType`（MySQL 风格类型名 + 模糊兜底）；回程转换 `ConvertValueForLocal`（bool → 0/1，date/datetime → `gtime.Time`） |
| **空跑（DryRun）** | `Core.SetDryRun(true)` 或环境/命令行 `gf.gdb.dryrun`；**只拦 `Exec` 类（DML），SELECT 仍会执行** |
| **日志输出** | `ConfigNode.Debug` / `Core.SetDebug`：`[ 12 ms] [group] [schema] [rows:3] [txid:1] SELECT ...`；`gdb.ToSQL(ctx, f)` / `gdb.CatchSQL(ctx, f)` 是排查 SQL 的首选工具 |
| **主从切换** | group 内节点按 `Role`（master/slave）+ `Weight*100` 区间随机；**至少一个 master**；读默认走 `SlaveLink`，写走 `MasterLink`；`Master()/Slave()` 可强制；`PingMaster/PingSlave/Stats` 检查连接池 |
| **查询缓存** | `Cache(gdb.CacheOption{Duration, Name, Force})` / `PageCache(countOpt, dataOpt)`；key 含 `table@group@schema@name@sql@args`；**`m.tx != nil` 时不走缓存**；`Duration < 0` 是"注册一个可在写操作时被清除的 key"语义；`Force` 防缓存穿透；写入成功后自动 `checkAndRemoveSelectCache` |
| **悲观锁 & 乐观锁** | `LockUpdate()` / `LockShared()` / `LockUpdateSkipLocked()`（SQLite 等无行级锁语法） |
| **Handler 特性** | `Handler(func(ctx, m *Model) *Model)` 对 Model 做函数式改写（等价于自研 FilterAuth / Sorter 等中间件能力） |
| **Hook 特性** | `Hook(HookHandler{Select/Insert/Update/Delete})`，签名 `func(ctx, in *HookXxxInput) (result, err)`；可改 `Table/Schema`（真实改写 SQL 与链接），调用时机在 SQL 组装完成、落库之前 |
| **模型关联** | 动态关联 `ScanList(pointer, bindToAttrName, relationAttrNameAndFields...)`；静态关联 `With(...)` |
| **分库分表** | `Sharding(...)` / `ShardingValue(...)`（`gdb_model_sharding.go`），`getActualSchema/getActualTable` 做实际路由 |
| **分页查询** | `Page(page, limit)` 或 `Limit/Offset`；`Chunk(size, handler)` 内部按页循环，`handler` 返回 false 或 `len < size` 停止 |
| **可观测性** | 内建 OpenTelemetry span（`gdb_core_trace.go`），SQL 与链路可关联 |

#### 5.4.8 常用操作示例（语义速查）

```go
// 条件：map 语义
g.Model("user").Where(g.Map{"id": nil}).All()          // WHERE `id` IS NULL
g.Model("user").Where(g.Map{"age >": 16}).All()         // WHERE `age` > 16
g.Model("user").Where(g.Map{"id": []int{}}).All()       // WHERE 0=1
g.Model("user").Where("id", []int{1,2}).All()           // WHERE `id` IN(1,2)
g.Model("user").WhereOr("id>?", 1).WhereOr("name", "john").All()

// 查询
g.Model("user").Where("uid>?", 1).All()                 // Result
g.Model("user").Fields("id,name").WherePri(1).Scan(&user)
g.Model("user").Where("uid", 1).Count()
g.Model("user").Where("uid", 1).Value("name")

// 缓存
g.Model("user").WherePri(1).Cache(gdb.CacheOption{Duration: time.Minute, Name: "user:1", Force: true}).One()

// 更新（零值必须显式 Data，且不要 OmitEmpty）
g.Model("user").WherePri(1).Data(g.Map{"status": 0}).Update()
g.Model("user").WherePri(1).Increment("score", 10)
```

---

### 5.5 服务配置 `server-config`

#### 5.5.1 三种配置方式

| 方式 | 说明 |
|------|------|
| 配置文件 | `manifest/config/config.yaml` 的 `server` 段（**最常用**） |
| `SetConfig(c ServerConfig)` | 代码方式设置整个配置结构 |
| `SetConfigWithMap(m)` | Map 方式设置：内部 `MapCopy` 浅拷贝 → 对 `maxHeaderBytes`/`clientMaxBodySize`/`formParsingMemory` 用 `gfile.StrToSize` 支持 `"1m"`/`"500kb"` → `gconv.Struct` |

**并发约束（必记）**：配置项在 Server 执行 `Start` **之后不可再修改**。源码口径的准确表述是——这不是强制校验，而是**事实约定**：

1. 唯一硬校验是"**禁止重复 Start**"（`ServerStatusRunning` 时返回 `server is already running`）；
2. 大量配置在 `Start()` 中被**一次性消费**（`sessionManager` 重建、`Handler` 赋值、`EnablePProf`、插件 `Install`、`handlePreBindItems` 真正写路由树、`initOpenApi`、`doServiceRegister`、`doRouterMapDump`）；
3. 路由匹配缓存 `serveCache` TTL = **1 小时**，启动后新增路由在缓存过期前可能不生效；
4. `s.config` **无互斥锁**，运行时修改存在数据竞争；`SetLogger` 的注释明确写了 "cannot be set in runtime"。

#### 5.5.2 全量配置模板（分组）★

```yaml
server:
    # 基本配置
    address:             ":80"                        # 本地监听地址。默认":80"
    httpsAddr:           ":443"                       # TLS/HTTPS 配置地址，需同时配置证书与密钥
    httpsCertPath:       ""                           # TLS/HTTPS 证书文件本地路径（建议绝对路径）
    httpsKeyPath:        ""                           # TLS/HTTPS 密钥文件本地路径（建议绝对路径）
    readTimeout:         "60s"                        # 请求读取超时，默认 60s
    writeTimeout:        "0"                          # 数据返回写入超时，默认不超时
    idleTimeout:         "60s"                        # Keep-Alive 闲置时间，默认 60s
    maxHeaderBytes:      "10240"                      # 请求 Header 大小限制，默认 10KB
    keepAlive:           true                         # 是否开启 Keep-Alive，默认 true
    serverAgent:         "GoFrame HTTP Server"        # 服务端 Agent 信息

    # 接口文档
    openapiPath:         "/api.json"                  # OpenAPI 接口文档地址
    swaggerPath:         "/swagger"                   # 内置 SwaggerUI 地址

    # 静态服务
    indexFiles:          ["index.html","index.htm"]   # 自动首页检索
    indexFolder:         false                        # 访问目录时是否列出文件（默认关闭 → 返回 403）
    serverRoot:          "/var/www"                   # 静态文件服务根路径（配置即开启静态服务）
    searchPaths:         ["/home/www"]                # 额外搜索路径（按顺序查找）
    fileServerEnabled:   false                        # 静态文件服务总开关

    # Cookie
    cookieMaxAge:        "365d"                       # 默认 365 天
    cookiePath:          "/"                          # 默认全站
    cookieDomain:        ""                           # 默认当前域名

    # Session
    sessionMaxAge:       "24h"                        # 默认 24 小时
    sessionIdName:       "gfsessionid"                # SessionId 键名
    sessionCookieOutput: true                         # 是否把 SessionId 写入 Cookie
    sessionPath:         "/tmp/gsessions"             # 文件存储目录

    # 日志（基础）
    logPath:             ""                           # 为空表示关闭文件输出
    logStdout:           true
    errorStack:          true
    errorLogEnabled:     true
    errorLogPattern:     "error-{Ymd}.log"
    accessLogEnabled:    false
    accessLogPattern:    "access-{Ymd}.log"

    # 日志（扩展，传给 glog）
    logger:
      path:                  "/var/log/"
      file:                  "{Y-m-d}.log"
      prefix:                ""
      level:                 "all"
      stdout:                true
      rotateSize:            0                        # 按大小滚动切分，0 关闭
      rotateExpire:          0                        # 按时间间隔滚动切分，0 关闭
      rotateBackupLimit:     0                        # 0 表示不备份，切分即删除
      rotateBackupExpire:    0
      rotateBackupCompress:  0                        # 压缩比 0-9，0 不压缩
      rotateCheckInterval:   "1h"                     # 巡检间隔，默认 1h

    # PProf
    pprofEnabled:        false                        # 默认关闭
    pprofPattern:        ""                           # PProf 页面访问路径

    # 其他
    clientMaxBodySize:   8388608                      # 上传 Body 上限（Byte），默认 8MB
    formParsingMemory:   1048576                      # 表单解析缓冲，默认 1MB
    nameToUriType:       0                            # 对象注册时的路由生成规则
    routeOverWrite:      false                        # 重复路由是否强制覆盖（默认 false → 启动报错退出）
    dumpRouterMap:       true                         # 启动时打印路由表，默认 true
    graceful:            false                        # 平滑重启（启用会额外占用本地 10000 端口）
    gracefulTimeout:     2                            # 父进程平滑重启后退出等待秒数
```

> ⚠️ 官方 2.0.x 模板中 `clientMaxBodySize` 写成了 `810241024`（应为注释所述 `8*1024*1024`），属**模板笔误**；HotGo 显式配为 `"200MB"`，并支持 `"100KB"`/`"1m"`/`"500kb"` 这类字符串（由 `gfile.StrToSize` 解析）。
> 路由相关的另外两个关键项（`routeOverWrite`、`dumpRouterMap`）在排障时价值最高：前者决定重复路由是"杀进程"还是"首个胜出"，后者能直接看到路由表与中间件绑定。

---

## 6. 官方范式 ↔ HotGo 落地对照

| 官方范式 | HotGo 的实现 | 结论 |
|----------|--------------|------|
| 规范路由 Req/Res + `g.Meta` | `server/api/**` 全量按此书写，`path/method/tags/summary` 齐备 | ✅ 完全符合，`/api.json`、`/swagger` 由此自动生成 |
| 分组路由 + `group.Bind` | `internal/router/{admin,api,home,websocket}.go` 四应用分组 + `genrouter.Register` 二次分组 | ✅ 教科书级用法 |
| 分组中间件 | `group.Middleware(AdminAuth / ApiAuth / HomeAuth / WebSocketAuth)` | ✅ |
| 全局中间件 | `cmd/http.go`：`Ctx → CORS → Blacklist → DemoLimit → PreFilter → ResponseHandler` | ✅ 顺序有讲究，`Ctx` 必须最先 |
| Hook | `s.BindHook(service.Hook())` → `BeforeServe` / `AfterOutput` 做访问日志与操作日志 | ✅ 对应官方 Hook 机制 |
| `v:"required"` 自动校验 | **HotGo 不用 `v` 标签**，改为 `sysin.XxxInp.Filter(ctx)` + `utility/validate.PreFilter` | ⚠️ 有意偏离：可控性更强（业务级校验 + 统一错误文案），代价是失去 OpenAPI 参数约束描述。**遵循项目约定，不要擅自加 `v` 标签** |
| 默认 `{code,message,data}` 响应 | HotGo 自定义 `library/response`：`{code,message,data,error,timestamp,traceId}`，前端 axios 按 `code===0` 解包 | ⚠️ 偏离但合理（业务需要 traceId 与 error 明细） |
| `openapiPath` / `swaggerPath` | `manifest/config/config.yaml` 中 `/api.json`、`/swagger` | ✅ |
| 配置目录 `manifest/config` | 一致；`config.yaml` 被 `.gitignore` 忽略，靠 `config.example.yaml` 复制生成 | ✅ 符合"默认配置文件 + 配置目录"约定 |
| `g.DB()` 单例 + 驱动匿名导入 | `server/main.go` 匿名导入 mysql / pgsql / redis 驱动 | ✅ |
| `gf gen dao` / `gf gen service` | `Makefile` 的 `dao` / `service` 目标 + `hack/config.yaml`；另有在线生成器 `internal/library/hggen` | ✅ 且有增强（内嵌 CLI + 模板渲染） |
| `gf build` 资源打包 | `gfcli.build.packSrc: resource` → `internal/packed/packed.go`，运行时经 gres 读取 | ✅ |
| 分组路由 `ALLMap` 批量注册 | 未使用（HotGo 用显式 `group.Bind(...)` 列表） | ➖ 可选优化点 |

**结论**：HotGo 在路由、中间件、配置、ORM、错误处理五方面均为 GoFrame 的规范用法；仅有的两处偏离（输入校验走 `Filter`、响应结构自定义）是**项目约定**而非框架误用，且已固化在 [`AGENTS.md`](../AGENTS.md)——改代码必须服从项目约定。

---

## 7. 进阶学习地图

### Stage 1｜快速开始（本文档已覆盖）
安装与 `go get`、Hello World、分组路由、规范路由、中间件、OpenAPI 标签

### Stage 2｜核心组件（按此顺序精读子页面）
```
gctx（上下文）→ glog（日志 + 链路）→ gerror + gcode（错误码体系）
→ gvalid（校验规则全表）→ gconv / gjson / gvar（万能转换，框架灵魂）
→ gcache → gcfg（含 Loader 加载器）→ gdb（链式操作 16 项 + 事务 6 项）
```

### Stage 3｜WEB 服务开发
```
服务配置（ServerConfig 全表；Start 后配置冻结）
→ 请求输入（Context / JSON·XML / 复杂参数 / 对象处理 / 文件上传 / 自定义变量 / 请求校验 / 默认值绑定）
→ 数据返回（Exit 控制 / 缓冲控制 / 模板解析 / 文件下载 / JSON·XML / Redirect）
→ 接口文档（OpenAPIv3 / 自定义 UI）
→ 会话 Cookie / 静态服务 / WebSocket
```

### Stage 4｜工程化与微服务
`gf init` 脚手架 → `gf gen dao | service | ctrl` 原理 → `gf build` 资源注入 → `gf docker`
→ `contrib/registry`（etcd / nacos / polaris）→ `grpcx` → `contrib/trace`（jaeger / otlp）

### Stage 5｜源码级
读 `gtype` / `gvar` 与 `gdb` 核心实现，理解「**接口化 + 单例 + context 注入**」三件套如何在框架内闭环。

**第 5 节补充内容的来源对照（本次已完成）**

| 主题 | 来源 | 落实位置 |
|------|------|----------|
| 校验规则全表（67 条） | v2.10.0 源码 `util/gvalid/internal/builtin/` 枚举 + 2.0.x 文档语义与示例 | §5.2.2、§5.2.3、§5.2.6 |
| 校验对象 / 校验结果 / 可选与递归校验 / 自定义规则 | v2.10.0 源码 `util/gvalid/*` + 2.0.x 文档 | §5.2.3 ~ §5.2.5 |
| 配置对象 / 文件配置 / 常用方法 / 监听回调 | v2.10.0 源码 `os/gcfg/*` + 2.0.x 文档 | §5.1.1 ~ §5.1.4 |
| **gcfg Loader 加载器** | v2.10.0 源码 `os/gcfg/gcfg_loader.go`（2.0.x 无此章节） | §5.1.5 ★ |
| 错误处理常用方法 / 错误码特性 / 最佳实践 | v2.10.0 源码 `errors/gerror/*`、`errors/gcode/gcode.go` | §5.3.1 ~ §5.3.5 ★ |
| ORM 链式操作（16 项）/ 事务 / 结果处理 / 高级特性 | v2.10.0 源码 `database/gdb/gdb_model*.go`（203 个方法枚举）+ 2.0.x 文档 | §5.4.2 ~ §5.4.8 ★ |
| 服务配置模板与配置方法 | 2.0.x 文档「服务配置-配置文件模板」 + v2.10.0 源码 `ghttp_server_config*.go` | §5.5.1 ~ §5.5.2 ★ |

**第二轮补充：`gvalid` 章节已补齐**（全部以 v2.10.0 源码为准，不依赖官网深层页）

| 主题 | 来源 | 落实位置 |
|------|------|----------|
| 校验数据形态（单值 / Map / Struct）与 `Run` 分派 | `util/gvalid/gvalid_validator.go`（`Run` 三分支 + `Run(ctx) Error` 签名） | §5.2.5 ★ |
| `Validator` 12 个公开方法签名核对 | 源码 `gvalid_validator.go` | §5.2.3 ★ |
| 校验结果 `validationError` 结构、保序机制、内部错误 key | 源码 `gvalid_error.go` | §5.2.4 ★ |
| 自定义规则：注册 API 表 + `RuleFuncInput` 全字段表 + 3 种注册示例 + 4 条语义 | 源码 `gvalid_register.go` | §5.2.7 ★ |
| 自定义错误消息、消息解析 4 级优先级、`{field}/{value}` 占位符、错误码 | 源码 `gvalid_validator_message.go`、`internal/builtin/*.go` | §5.2.8 ★ |
| `checkRuleRequired` 与"自定义规则默认按 required 处理" | 源码 `gvalid_validator.go` | §5.2.6 ★ |

**仍可继续补充**（官网 2.10.x 深层 slug 未公开，建议直接从源码或 2.0.x 对应页获取）：

| 主题 | 建议来源 |
|------|----------|
| 错误码扩展（自定义码实现） | 源码 `errors/gcode/gcode.go`（`Code` 接口 3 方法）+ `/2.0.x/核心组件-重点/错误处理/错误处理-错误码特性/*`（5 篇） |
| ORM 各链式章节细节（时间维护/字段过滤/对象输入/Handler/Hook/悲观锁等） | `/2.0.x/核心组件-重点/数据库ORM/ORM链式操作-重点/*`（16 项） |
| ORM 高级特性 11 项与最佳实践 | `/2.0.x/核心组件-重点/数据库ORM/ORM高级特性/*`、`ORM最佳实践` |
| ORM 数据库类型识别与字段映射（`CheckLocalTypeForField` 全表） | 源码 `database/gdb/gdb_func.go`、`gdb_type_*.go` |
| 日志组件高级特性（Rotate / Handler / Writer / 异步 / Context） | `/2.0.x/核心组件-重点/日志组件/*` |
| 缓存适配器与接口化设计 | `/2.0.x/核心组件-重点/缓存管理/*` |
| 请求输入 / 数据返回 全量章节 | `/2.0.x/WEB服务开发/请求输入/*`、`/2.0.x/WEB服务开发/数据返回/*` |
| 中间件/Hook 触发顺序的源码级证据（`ghttp` 六个 Hook 常量与调用点） | 源码 `net/ghttp/ghttp_server_hook.go`、`ghttp_server_handler.go` |

---

## 8. 必须避开的坑（速查）

| # | 内容 |
|---|------|
| 1 | `gdb` 驱动**必须手动匿名导入**，否则运行期才报"driver not found" |
| 2 | `Server` 配置在 `Start` 之后**不可再修改**（并发安全要求） |
| 3 | 规范路由下 `req` / `res` **必须显式定义**，哪怕为空 |
| 4 | **同一接口不能多 Method**；需要 `ALL` 语义就退回普通路由 `func(r *ghttp.Request)` |
| 5 | 校验中 **`0` 与 `false` 会被校验**，而 `nil` / 空串默认跳过——"状态值传 0 被拦"的根因 |
| 6 | struct 字段默认值会误触发 `required`：改用指针 / 组合规则 / `Assoc` |
| 7 | `Wrap` 时别再拼原错误文本；调试统一用 `%+v` |
| 8 | 分组中间件不生效时，先看启动路由表的 `MIDDLEWARE` 列（`DumpRouterMap: true`） |
| 9 | 中间件里**不调用 `r.Middleware.Next()` 就是中断**——鉴权要拦就拦，别忘放行 |
| 10 | 配置缺项会回退到环境变量/命令行参数，排查"配置没生效"时先确认这两处 |

---

## 9. 与 HotGo 结合的使用建议

1. 新增接口时：先按 [MODULES.md](MODULES.md) 的"五后端 + 三前端"清单落地，`api` 层**严格按规范路由写 `g.Meta`**，这样 `/swagger` 自动同步。
2. 需要免登录/免权限：改 `manifest/config/config.yaml` 的 `router.<app>.exceptLogin` / `exceptAuth`，**不要**在中间件里写路径特判。
3. 新增中间件：先在 `internal/service/middleware.go` 声明接口，再在 `internal/logic/middleware/` 实现，最后在 `cmd/http.go`（全局）或 `internal/router/*.go`（分组）挂载。
4. 涉及数据库：一律 `dao.XxxTable.Columns().FieldName`，禁止硬编码字段名；关联/树形/租户走 `library/hgorm` 的 handler 与 hook。
5. 错误返回：使用 `gerror` + 项目统一的 `library/response`，不要在业务代码里手写 JSON。
6. 生成代码：优先走后台在线生成器（`views/develop/code`），生成结果即为本仓库的标准范式，比自己手写更"像项目"。
