# 业务模块地图

> 后端模块（`api/` → `logic/` → `service/`）与前端（`views/` → `api/`）的完整对照表。
> 配套阅读：[PROJECT_ANALYSIS.md](PROJECT_ANALYSIS.md) · [ARCHITECTURE-BACKEND.md](ARCHITECTURE-BACKEND.md) · [ARCHITECTURE-FRONTEND.md](ARCHITECTURE-FRONTEND.md)

---

## 1. 模块的组织方式

一个完整业务模块由**五个后端文件 + 三个前端文件**构成（以 `testCategory` 为例）：

```text
server/api/admin/testcategory/testcategory.go           # 接口契约（Req/Res + g.Meta）
server/internal/model/input/sysin/test_category.go      # 入参/出参/字段白名单 + Filter 校验
server/internal/controller/admin/sys/test_category.go   # 薄控制器（透传）
server/internal/logic/sys/test_category.go              # 业务实现（init() 注册到 service）
server/internal/router/genrouter/test_category.go       # 路由挂载（init() append 到 LoginRequiredRouter）
server/internal/service/sys.go                          # 追加 ISysTestCategory 接口声明 + Register 函数

web/src/api/testCategory/index.ts                       # 前端请求封装
web/src/views/testCategory/model.ts                     # State / schemas / columns / loadOptions
web/src/views/testCategory/index.vue                    # 列表页（+ edit.vue 编辑弹窗）
```

标准接口清单：`/xxx/list`(GET)、`/xxx/view`(GET)、`/xxx/edit`(POST，`id>0` 为编辑)、`/xxx/delete`(POST，支持 `ids` 批量)、`/xxx/status`(POST)、`/xxx/switch`(POST)、`/xxx/maxSort`(GET)、`/xxx/export`(GET)——按需裁剪。

---

## 2. 系统类模块（`internal/logic/sys/` ↔ `internal/service/sys.go`）

| 后端模块（`api/admin/`） | Service 接口 | 逻辑实现 | 前端页面 | 前端 API |
|--------------------------|--------------|----------|----------|----------|
| `config` | `ISysConfig` | `sys/config.go` | `views/system/config/`（Basic/Cash/Email/Geo/Login/Pay/Reveal/Sms/Theme/Upload/Wechat/`system.vue`） | `api/sys/config.ts` |
| `dict` | `ISysDictType` / `ISysDictData` | `sys/dict_type.go` / `sys/dict_data.go` | `views/system/dict/`（index/list/model/CreateDrawer/columns） | `api/dict/dict.ts` |
| `attachment` | `ISysAttachment` | `sys/attachment.go` | `views/apply/attachment/` | `api/apply/attachment.ts` |
| `provinces` | `ISysProvinces` | `sys/provinces.go` | `views/apply/provinces/` | `api/apply/provinces.ts` |
| `cron` | `ISysCron` / `ISysCronGroup` | `sys/cron.go` / `sys/cron_group.go` | `views/system/cron/` | `api/sys/cron.ts` |
| `blacklist` | `ISysBlacklist` | `sys/blacklist.go` | `views/system/blacklist/` | `api/sys/blacklist.ts` |
| `log` | `ISysLog` | `sys/log.go` | `views/log/log/` | `api/log/log.ts` |
| `loginlog` | `ISysLoginLog` | `sys/login_log.go` | `views/log/login-log/` | `api/loginLog/` |
| `servelog` | `ISysServeLog` | `sys/serve_log.go` | `views/monitor/serve-log/` | `api/serveLog/` |
| `smslog` | `ISysSmsLog` | `sys/sms_log.go` | `views/log/sms-log/` | `api/log/smslog.ts` |
| `emslog` | `ISysEmsLog` | `sys/ems_log.go` | `views/log/ems-log/` | `api/log/` |
| `gencodes` | `ISysGenCodes` | `sys/gen_codes.go` | `views/develop/code/` | `api/develop/code.ts` |
| `addons` | `ISysAddons` | `sys/addons.go` | `views/develop/addons/` | `api/develop/addons.ts` |
| `servelicense` | `ISysServeLicense` | `sys/serve_license.go` | `views/monitor/`（授权信息） | `api/serveLicense/` |
| （无独立契约） | `ISysAddonsConfig` | `sys/addons_config.go` | 插件配置页（插件内调用） | 插件内 `api/addons/<name>/config` |

> `ISysAddonsConfig` 是插件配置的**统一存储服务**：按 `addon+group+key` 读写 `hg_sys_addons_config`，支持 `ConversionType` 类型转换；不支持从前台新增变量，变量需先入库。

---

## 3. 运营/后台类模块（`internal/logic/admin/` ↔ `internal/service/admin.go`）

| 后端模块 | Service 接口 | 逻辑实现 | 前端页面 | 前端 API |
|----------|--------------|----------|----------|----------|
| `member`（用户/管理员） | `IAdminMember` | `admin/member.go`（最大业务文件） | `views/org/user/`（user/list/model/columns/addBalance/addIntegral） | `api/org/user.ts` |
| `member`（用户-岗位） | `IAdminMemberPost` | `admin/member_post.go` | 同上 | 同上 |
| `role`（角色 + casbin 校验） | `IAdminRole` | `admin/role.go` | `views/permission/role/`（含 editMenuAuth/editDataAuth） | `api/system/role.ts` |
| `menu`（菜单/权限点/动态路由） | `IAdminMenu` | `admin/menu.go` | `views/permission/menu/` | `api/system/menu.ts` |
| `dept`（部门，树形） | `IAdminDept` | `admin/dept.go` | `views/org/dept/` | `api/org/dept.ts` |
| `post`（岗位） | `IAdminPost` | `admin/post.go` | `views/org/post/` | `api/org/post.ts` |
| `notice`（通知公告） | `IAdminNotice` | `admin/notice.go` | `views/apply/notice/` | `api/apply/notice.ts` |
| `order`（充值订单） | `IAdminOrder` | `admin/order.go` | `views/asset/rechargeLog/`、`views/asset/recharge/` | `api/order/` |
| `creditslog`（资金/积分变动） | `IAdminCreditsLog` | `admin/credits_log.go` | `views/asset/creditsLog/` | `api/creditsLog/` |
| `cash`（提现） | `IAdminCash` | `admin/cash.go` | `views/asset/cash/` | `api/cash/` |
| `monitor`（服务监控） | `IAdminMonitor` | `admin/monitor.go` | `views/monitor/serve-monitor/`、`views/monitor/online/`、`views/monitor/netconn/` | `api/monitor/monitor.ts` |
| （后台站点：登录/验证码/配置） | `IAdminSite` | `admin/site.go` | `views/login/` | `api/system/user.ts`、`api/base/` |

---

## 4. 其它支撑模块

| 模块 | Service 接口 | 位置 | 前端 |
|------|--------------|------|------|
| 支付（下单/异步通知/退款） | `IPay` / `IPayRefund` | `logic/pay/{pay,create,notify,refund}.go` | `views/asset/payRefund/`、`api/pay/refund.ts` |
| 通用上传 | `ICommonUpload` | `logic/common/upload.go` | `api/base/`、组件 `Upload` / `FileChooser` |
| 微信授权 | `ICommonWechat` | `logic/common/wechat.go` | 登录页微信扫码/授权跳转 |
| 前台视图渲染 | `IView` | `logic/view/{view,view_buildin}.go` | `views/iframe/`、`home` 入口页 |
| Hook（访问日志/最后活跃） | `IHook` | `logic/hook/{init,access_log,last_active}.go` | — |
| 中间件 | `IMiddleware` | `logic/middleware/`（9 个文件） | — |
| 内置 TCP 服务 | `ITCPServer` | `logic/tcpserver/` | — |
| TCP 客户端（授权/定时任务） | `IAuthClient` / `ICronClient` | `logic/tcpclient/` | — |
| 数据字典引擎 | （库） | `library/dict/` | `store/modules/dict.ts` |
| WebSocket 推送 | （运行时） | `internal/websocket/` | `utils/websocket/` |

---

## 5. 生成代码示例模块（开箱即用的范例）

这四个模块由代码生成器产出，是**学习"标准模块长什么样"的最佳样本**：

| 模块 | 生成类型 | 后端 | 前端 |
|------|----------|------|------|
| `curddemo` | CRUD 列表 | `api/admin/curddemo/`、`logic/sys/curd_demo.go`、`genrouter/curd_demo.go` | `views/curdDemo/`、`api/curdDemo/` |
| `normaltreedemo` | 关系树（普通树） | `api/admin/normaltreedemo/`、`logic/sys/normal_tree_demo.go` | `views/normalTreeDemo/` |
| `optiontreedemo` | 关系树（选项树） | `api/admin/optiontreedemo/`、`logic/sys/option_tree_demo.go` | `views/optionTreeDemo/` |
| `testcategory` | CRUD + 字典 | `api/admin/testcategory/`、`logic/sys/test_category.go` | `views/testCategory/`、`api/testCategory/` |

---

## 6. 插件模块

当前仓库仅有一个插件 `hgexample`（Label「功能案例」），其结构、能力与新增方式见 [ADDONS-AND-CODEGEN.md](ADDONS-AND-CODEGEN.md)。

| 插件 | 后端 | 前端 |
|------|------|------|
| `hgexample` | `addons/hgexample/`（`logic/sys/{config,index,table,tree_table,tenant_order}.go`，`router/genrouter/tenant_order.go`） | `web/src/api/addons/hgexample/`（comp/config/table/treeTable/tenantOrder）、`web/src/views/addons/hgexample/`（table、treeTable、tenantOrder、config、portal、comp 20+ 组件演示） |

---

## 7. 前端非业务页面

| 目录 | 说明 |
|------|------|
| `views/dashboard/` | 控制台（`console/` + company/merchant/tenant/user 变体、`workplace/`） |
| `views/home/` | 个人中心（`account/`、`message/`） |
| `views/login/` | 登录（`index.vue` + `login/` + `register/` + `components/`） |
| `views/exception/` | 403 / 404 / 500 |
| `views/about/`、`views/iframe/`、`views/redirect/` | 关于、内嵌页、重定向中转 |

---

## 8. 数据结构与命名约定

| 约定 | 说明 |
|------|------|
| 表前缀 | 全局 `hg_`（因 `hack/config.yaml` 中 `removePrefix: "hg_"`，代码中 dao 名不带前缀，如 `dao.SysConfig`） |
| 插件表前缀 | `addon_<name>_`（生成器会自动剥离该前缀作为模块名） |
| 表 → 代码 | `hg_sys_config` → `dao.SysConfig` + `entity.SysConfig` + `do.SysConfig` + `sysin.*` |
| 逻辑文件 | `logic/sys/<module>.go`，结构体 `sSysXxxModule` |
| 控制器 | `controller/admin/sys/<module>.go`，变量 `sys.XxxModule` |
| 前端页面 | `views/<module|业务域>/`，三件套 `index.vue` + `edit.vue` + `model.ts` |
| 权限标识 | 与非业务路由路径一致，如 `/testCategory/edit`（由后端菜单下发） |

---

## 9. 字典与常量索引（`internal/consts/`，30 个文件）

| 文件 | 内容 |
|------|------|
| `app.go` | 应用入口常量（admin/api/home/websocket/default） |
| `config.go` | 系统配置分组与字段 |
| `status.go` | 通用状态（启用/禁用等） |
| `role.go` | 角色类型（超管/管理员/租户/商户/用户等），配合数据范围权限 |
| `dict.go` 等 | 各业务字典选项（`XxxStatusOptions` 等），`init()` 中 `dict.RegisterEnums` 注册 |
| `cron.go` | 定时任务策略常量（Same/Single/Once/Times） |
| `queue.go` | 队列主题常量（request_log / login_log / serve_log） |
| `gencodes.go` | 生成类型常量（CURD=10、Tree=11、Queue=20、Cron=30） |
| `pay.go`、`order.go` | 支付通道、订单状态 |
| `addons.go` | 插件安装状态（`AddonsInstallStatusOk` 等） |

前端对应的字典 key 通过 `GET /dictData/options` 拉取，页面中使用 `dict.getLabel(type, value)` / `dict.getOptionUnRef(type)`（详见 [ARCHITECTURE-FRONTEND.md](ARCHITECTURE-FRONTEND.md) 第 8 节）。

---

## 10. 内置功能对照（README 23 项 → 代码位置）

| 内置功能 | 后端模块 | 前端页面 |
|----------|----------|----------|
| 用户管理 | `admin/member` | `views/org/user` |
| 部门管理 | `admin/dept` | `views/org/dept` |
| 岗位管理 | `admin/post`、`admin/member_post` | `views/org/post` |
| 菜单管理 | `admin/menu` | `views/permission/menu` |
| 角色管理 | `admin/role` | `views/permission/role` |
| 字典管理 | `sys/dict_type`、`sys/dict_data` | `views/system/dict` |
| 配置管理 | `sys/config` | `views/system/config` |
| 操作日志 | `sys/log`、`hook/access_log` | `views/log/log` |
| 登录日志 | `sys/login_log` | `views/log/login-log` |
| 服务日志 | `sys/serve_log` | `views/monitor/serve-log` |
| 支付网关 | `pay/*`、`library/payment` | `views/asset/payRefund` |
| 资金管理 | `admin/order`、`admin/credits_log`、`admin/cash` | `views/asset/*` |
| 在线用户 | `admin/monitor` | `views/monitor/online` |
| 定时任务 | `sys/cron`、`library/cron`、`crons/*` | `views/system/cron` |
| 代码生成 | `sys/gen_codes`、`library/hggen` | `views/develop/code` |
| 插件应用 | `sys/addons`、`library/addons` | `views/develop/addons` |
| 服务监控 | `admin/monitor` | `views/monitor/serve-monitor` |
| 附件管理 | `sys/attachment`、`library/storager` | `views/apply/attachment` |
| TCP 服务 | `logic/tcpserver`、`library/network` | — |
| 消息队列 | `library/queue`、`queues/*` | — |
| 通知公告 | `admin/notice` + WebSocket | `views/apply/notice`、`views/home/message` |
| 地区编码 | `sys/provinces`、`library/location` | `views/apply/provinces`、组件 `CitySelector` |
| 常用工具 | `utility/*` | — |
