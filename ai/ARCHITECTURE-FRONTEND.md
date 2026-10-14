# 前端架构详解（web/）

> 目录：`web/`，包名 `hotgo` v2.18.9，Vue 3.4 + Naive UI 2.44 + Vite 5 + Pinia 2 + TypeScript 5.5。
> 配套阅读：[PROJECT_ANALYSIS.md](PROJECT_ANALYSIS.md) · [ARCHITECTURE-BACKEND.md](ARCHITECTURE-BACKEND.md) · [MODULES.md](MODULES.md) · [AGENTS.md](../AGENTS.md)

---

## 1. 工程化配置

### 1.1 环境变量

| 文件 | 关键项 | 值 |
|------|--------|-----|
| `web/.env` | `VITE_PORT` | `8001`（dev 端口） |
| | `VITE_GLOB_APP_TITLE` / `VITE_GLOB_APP_SHORT_NAME` | `加载中` / `HG` |
| `web/.env.development` | `VITE_PUBLIC_PATH` | `/` |
| | `VITE_PROXY` | `[["/admin","http://localhost:8000/admin"]]` |
| | `VITE_GLOB_API_URL_PREFIX` | `/admin` |
| | `VITE_APP_DEMO_ACCOUNT` | 快速登录账号矩阵（超管/管理员/租户/商户/用户） |
| `web/.env.production` | `VITE_PUBLIC_PATH` | `/admin`（部署到后端 `/admin` 路径下） |
| | `VITE_GLOB_API_URL_PREFIX` | `/admin` |
| | `VITE_BUILD_COMPRESS` | `none`（可改 gzip / brotli） |

### 1.2 `vite.config.ts`

- `base` 取自 `VITE_PUBLIC_PATH`；`wrapperEnv` 负责 `.env` 解析与类型转换。
- **别名**：`@` → `src/`，`/#/` → `types/`；`dedupe: ['vue']`。
- **插件**：统一由 `createVitePlugins(viteEnv, isBuild)`（`build/vite/plugin/index.ts`）聚合。
- **CSS**：less `javascriptEnabled: true`，全局注入 `src/styles/var.less`。
- **devServer**：`host: true`、`port: VITE_PORT`、`proxy: createProxy(VITE_PROXY)`。
- **build**：`target: es2015`、`cssTarget: chrome80`、`outDir: dist`、`chunkSizeWarningLimit: 3000`。
- **define**：注入 `__APP_INFO__`（包信息 + 构建时间）。

### 1.3 `build/` 目录

| 文件 | 职责 |
|------|------|
| `build/constant.ts` | `GLOB_CONFIG_FILE_NAME='app.config.js'`、`OUTPUT_DIR='dist'` |
| `build/getConfigFileName.ts` | 生成运行期配置全局变量名 `__PRODUCTION__<SHORT_NAME>__CONF__` |
| `build/utils.ts` | `wrapperEnv`、`getEnvConfig`（提取 `VITE_GLOB_`）、环境判断、`getRootPath` |
| `build/vite/proxy.ts` | 由 `[[prefix, target], ...]` 生成 vite 代理（`ws: true`，rewrite 去前缀） |
| `build/vite/plugin/index.ts` | 插件聚合：`vue()`、`vueJsx()`、`Components({resolvers:[NaiveUiResolver()]})`、`AutoImport`（自动导入 vue/vue-router/`t`）、`topLevelAwait`、html、compress、`setupExtend()` |
| `build/vite/plugin/html.ts` | `vite-plugin-html`：注入 title，构建时把 `app.config.js` 以 script 注入 `index.html` |
| `build/vite/plugin/compress.ts` | gzip / brotli 压缩 |
| `build/script/buildConf.ts` | 打包后生成 `dist/app.config.js`（`Object.freeze(window.__PRODUCTION__HG__CONF__)`） |
| `build/script/postBuild.ts` | `pnpm build` 第二阶段入口，调用 `runBuildConfig` |

> **设计亮点**：生产环境可通过修改 `dist/app.config.js` 变更 API 地址等配置，**无需重新打包**。

**自动导入**：`unplugin-auto-import` 自动导入 `vue`、`vue-router` 及 `@/locale/index` 的 `t`；`unplugin-vue-components` + `NaiveUiResolver` 按需自动注册 Naive 组件（同时生成 `auto-imports.d.ts` / `components.d.ts`）。

### 1.4 质量工具

| 工具 | 配置要点 |
|------|----------|
| TypeScript | `strict: true`（但 `strictFunctionTypes: false`、`noImplicitAny: false`）；`noUnusedLocals/noUnusedParameters: true`；`paths`: `@/*`、`/#/*` |
| Tailwind | `content: ['./index.html','./src/**/*.{vue,ts,tsx}']`，`important: true` |
| PostCSS | `tailwindcss` + `autoprefixer` |
| Prettier | `printWidth 100`、单引号、`semi`、`vueIndentScriptAndStyle`、`trailingComma: es5` |
| ESLint 9 | flat config；继承 `vue3-recommended`、`@typescript-eslint/recommended`、`prettier` |
| Stylelint | `standard` + `prettier` + `order` |
| commitlint | `config-conventional`，`type-enum` 含 `feat/fix/perf/style/docs/test/refactor/build/ci/chore/revert/wip/workflow/types/release`，`header-max-length: 108` |
| husky + lint-staged | `*.{vue,js,ts,tsx}` → `eslint --fix` |

常用脚本：`pnpm dev`、`pnpm build`、`pnpm lint:eslint`、`pnpm lint:prettier`、`pnpm lint:stylelint`、`pnpm clean:cache`。

---

## 2. 目录结构

```text
web/
├── build/            # 构建插件与脚本（见 1.3）
├── types/            # 全局类型（#/ 别名）
├── public/
└── src/
    ├── api/          # 接口封装（22 个模块目录）
    ├── assets/       # icons / images
    ├── components/   # 通用组件（18 个目录）
    ├── debug/        # 调试账号（account.ts）
    ├── directives/   # 全局指令：permission / copy / debounce / throttle / draggable / clickOutside
    ├── enums/        # 枚举（11 个文件）
    ├── hooks/        # 组合式函数（common/core/event/setting/web + 顶层）
    ├── layout/       # 布局骨架（Header/Logo/Main/Menu/TagsView）
    ├── locale/       # i18n（index.ts + en/zh-Hans/zh-Hant.json）
    ├── plugins/      # 全局注册：naive.ts / directives.ts / index.ts
    ├── router/       # 路由与守卫
    ├── settings/     # 默认配置（projectSetting/designSetting/componentSetting/animateSetting）
    ├── store/        # Pinia（modules/ 下 10 个 store）
    ├── styles/       # index.less / common.less / hotgo.less / var.less / tailwind.css / transition
    ├── utils/        # http / websocket / is / lib / Storage / dateUtil / hotgo / tree ...
    ├── views/        # 页面（20 个一级模块，含 addons/ 插件页）
    ├── App.vue
    └── main.ts
```

---

## 3. 应用启动流程 `src/main.ts`

```ts
app.use(i18n); app.config.globalProperties.t = i18n.global.t;
setupNaive(app);          // 全局注册 Naive 组件
setupDirectives(app);     // v-permission 等
setupStore(app);          // Pinia

appProvider.mount('#appProvider', true);   // 先挂 Provider，供守卫/Axios 使用 $dialog/$message
await setupRouter(app);
await router.isReady();
setupWebsocket();         // 建立 WS 连接与全局消息处理
app.mount('#app', true);
```

`src/components/Application`（`AppProvider`）内部为
`NLoadingBarProvider > NDialogProvider > NNotificationProvider > NMessageProvider`，
并挂载 `DialogContent` / `MessageContent`——这是路由守卫与 Axios 拦截器中能直接使用 `window['$dialog']` / `window['$message']` 的原因。

---

## 4. 路由机制：后端菜单全量驱动

核心文件：`src/router/{index.ts, base.ts, constant.ts, generator-routers.ts, router-guards.ts, router-icons.ts}` 与 `src/store/modules/asyncRoute.ts`。

### 4.1 静态路由极简

```ts
export const asyncRoutes = [...routeModuleList];   // import.meta.glob('./modules/**/*.ts') —— 该目录不存在，恒为空
export const constantRouter = [LoginRoute, RootRoute, RedirectRoute];
const router = createRouter({ history: createWebHashHistory(''), routes: constantRouter, strict: true });
```

- `base.ts` 提供 `ErrorPageRoute`（`/:path(.*)*` → `views/exception/404.vue`）与 `RedirectRoute`。
- `constant.ts` 定义 `Layout`（`layout/index.vue`）、`ParentLayout`、`ErrorPage`。
- **路由使用 hash 模式**。

### 4.2 动态路由生成 `generator-routers.ts`

1. `generatorDynamicRouter()` → `adminMenus()`（`api/system/menu.ts` → `GET /role/dynamic`）；
2. `routerGenerator(list)` 递归转换后端菜单树 → vue-router 记录，`meta = {...item.meta, label, icon, permissions}`；`type===1`（目录）时若无 `redirect` 则默认跳第一个子路由；
3. `asyncImportRoute(routes)` 用 `import.meta.glob('../views/**/*.{vue,tsx}')` 把后端 `component` 字符串映射为组件：
   - `LAYOUT` → `layout`；`IFRAME` → `views/iframe`；
   - 仅 `frameSrc` 无组件 → `IFRAME`；仅 `name` 无组件 → `ParentLayout`；
4. `removeHiddenMenus()` 过滤 `type===3`（按钮权限）与 `hidden` 菜单。

### 4.3 权限模式与过滤 `store/modules/asyncRoute.ts`

`generateRoutes(userInfo)`：
- 读取 `useProjectSetting().getPermissionMode`，默认 `settings/projectSetting.ts` 中的 `'BACK'`；
- `'BACK'` → `generatorDynamicRouter()`；否则走前端静态 `asyncRoutes` + `routeFilter`；
- 再按 `user.permissions` 过滤，结果写入 `setRouters/setMenus`。

### 4.4 路由守卫 `router-guards.ts`

```ts
router.beforeEach(async (to, from, next) => {
  const token = storage.get(ACCESS_TOKEN);
  if (whitePathList.includes(to.path)) { await userStore.LoadLoginConfig(); next(); return; }
  if (!token) { if (to.meta.ignoreAuth) next(); else next({ path: '/login', query: { redirect: to.path } }); return; }
  if (asyncRouteStore.getIsDynamicAddedRoute) { next(); return; }

  const userInfo = await userStore.GetInfo();
  await userStore.LoadLoginConfig();
  if (userStore.allowWxOpenId()) { jump('/wechat/authorize', {...}); return; }
  await userStore.GetConfig();

  const routes = await asyncRouteStore.generateRoutes(userInfo);
  routes.forEach((item) => router.addRoute(item));   // 动态挂载
  router.addRoute(ErrorPageRoute);                  // 兜底 404
  asyncRouteStore.setDynamicAddedRoute(true);
  next(nextData);
});
```

- `afterEach` 设置 `document.title`、结束进度条；`onError` 打印错误。
- 动态路由失败时通过 `window['$dialog']` 提示"是否为角色分配菜单权限？"。
- `setupRouter(app)` 会调用 `createRouterIcon()`：把 `@vicons/antd` 图标注册为菜单图标渲染函数。

> **结论**：权限标识与后端菜单/路由路径一致（如 `/testCategory/edit`），页面路由**完全不可在前端源码硬编码**。

---

## 5. 状态管理（Pinia）`src/store/modules/`

| 文件 | store id | 职责 |
|------|----------|------|
| `user.ts` | `app-member` | token、用户信息、`permissions`、系统配置；`login/mobileLogin/handleLogin/GetInfo/GetConfig/LoadLoginConfig/allowWxOpenId/logout`；`isCompanyDept/isTenantDept/isMerchantDept/isUserDept`（按 `deptType`）；token 持久化 localStorage |
| `dict.ts` | `app-dict` | 字典选项（详见第 8 节），持久化 `CURRENT-DICT` |
| `asyncRoute.ts` | `app-async-route` | `menus/routers/keepAliveComponents/isDynamicAddedRoute`、`generateRoutes` |
| `tabsView.ts` | `app-tabs-view` | 多标签页：`addTabs/closeLeft/closeRight/closeOther/closeAll`，保留 `meta.affix` 固定页 |
| `projectSetting.ts` | `app-project-setting` | navMode / navTheme / menu / multiTabs / crumbs / permissionMode / 动画 / isMobile |
| `designSetting.ts` | `app-design-setting` | `darkTheme`、`appTheme`、`appThemeList` |
| `lockscreen.ts` | `app-lockscreen` | 锁屏 `isLock/lockTime`（默认 1 小时无操作锁屏） |
| `i18n.ts` | `I18nStore` | `currentLocale`、`setLocale/initLocale`（结合后端 `loginConfig`） |
| `notification.ts` | `notificationStore` | 站内信未读数、`pullMessages/triggerNewMessages/getUnreadCount` |

持久化 key（`store/mutation-types.ts`）：`ACCESS-TOKEN`、`CURRENT-USER`、`CURRENT-CONFIG`、`CURRENT-LOGIN-CONFIG`、`IS-LOCKSCREEN`、`TABS-ROUTES`、`CURRENT-DICT`、`Current-Locale`。

> 本项目**没有独立的 `permission` store**——权限集中在 `user.store.permissions` + `hooks/web/usePermission.ts` + `asyncRoute`。

---

## 6. HTTP 封装 `src/utils/http/axios/`

文件：`Axios.ts`、`index.ts`、`axiosTransform.ts`、`axiosCancel.ts`、`checkStatus.ts`、`helper.ts`、`types.ts`。

### 6.1 `VAxios` 类

- `request()`：`cloneDeep(config)` → `beforeRequestHook` → `axiosInstance.request` → `transformRequestData`，出错走 `requestCatch`；
- `uploadFile()`：构建 `FormData`（数组字段用 `key[]`）后 POST；
- `setupInterceptors()`：请求拦截做**重复请求取消**（`AxiosCanceler`）+ `requestInterceptors`；响应拦截做 `responseInterceptors` + 异常处理 `responseInterceptorsCatch`。

### 6.2 关键 transform（`index.ts`）

```ts
// 请求前：拼接 urlPrefix / apiUrl（GET 可加时间戳；POST 把 params 移入 data）
beforeRequestHook: (config, options) => { config.url = `${urlPrefix}${config.url}`; ... }

// 请求拦截：注入 token 与语言
requestInterceptors: (config) => {
  const token = userStore.getToken;
  if (token && config?.requestOptions?.withToken !== false)
    config.headers.Authorization = token;
  config.headers.Locale = i18nStore.getLocale();
  return config;
}
```

- **`transformRequestData`（响应解包）**：后端统一返回结构 `{ code, data, message }`；
  - `code === 0`（`ResultEnum.SUCCESS`）→ 返回 `data`，按需 `$dialog.success`；
  - `code === 61`（`ResultEnum.TIMEOUT`）→ `$dialog.warning` + `storage.clear()` + 跳登录；
  - 其它 → `$message.error` / `$dialog.info` 并抛 `Error`。
- **`responseInterceptorsCatch`**：处理 `ECONNABORTED`（超时）、`Network Error`，否则 `checkStatus(status, msg)` 按 HTTP 状态码提示。
- `RequestOptions` 默认：`joinPrefix: true`、`urlPrefix: '/admin'`、`withToken: true`、`ignoreCancelToken: true`、`formatDate: true`、`timeout: 30000`、`Content-Type: application/json`。

### 6.3 文件导出 `jumpExport`

```ts
export const jump = (url, params) => {
  window.location.href = urlPrefix + url + '?' +
    encodeParams({ ...delNullProperty(params), ...{ authorization: useUserStoreWidthOut().token } });
};
```

用于导出/下载类接口（把 token 拼进 URL），如各模块的 `Export()`。

### 6.4 `src/api` 模块划分

| 分组 | 模块目录 |
|------|----------|
| 认证与权限 | `system/`（user、menu、role）、`base/`（验证码/上传） |
| 字典 | `dict/` |
| 组织 | `org/`（dept、post、user） |
| 应用 | `apply/`（attachment、notice、provinces） |
| 资产 | `asset` 对应 API：`cash/`、`creditsLog/`、`pay/`、`order/`、`serveLicense/` |
| 日志 | `log/`、`loginLog/`、`serveLog/` |
| 运维 | `monitor/`、`sys/`（blacklist、config、cron） |
| 开发 | `develop/`（addons、code） |
| 示例 | `curdDemo/`、`normalTreeDemo/`、`optionTreeDemo/`、`testCategory/` |
| 插件 | `addons/hgexample/`（comp、config、table、treeTable、tenantOrder） |
| 首页 | `dashboard/` |

统一使用 `http.request`，函数命名规范：`List / Edit / Delete / Status / Switch / View / MaxSort / Export`。

---

## 7. 组件体系 `src/components`

18 个组件目录：`Application`、`CitySelector`、`ComplexMemberPicker`、`CountTo`、`DatePicker`、`DialogContent`、`Editor`、`FileChooser`、`Form`、`IconSelector`、`LoadingContent`、`Lockscreen`、`MessageContent`、`Modal`、`SvgIcon`、`Table`、`Upload`、`YamlEditor`。

### 7.1 表格 `components/Table`

- 导出：`BasicTable`（`src/Table.vue`）、`TableAction`（`src/components/TableAction.vue`）。
- `src/props.ts` 的 `basicProps` 继承 `NDataTable.props`，扩展 `title/dataSource/columns/request/beforeRequest/afterRequest/rowKey/pagination/actionColumn/canResize`。
- Hooks：`useColumns.ts`（列渲染；`column.auth` 权限过滤、`column.edit` 可编辑渲染）、`useDataSource.ts`（`fetch` 组装分页参数、`reload`）、`usePagination/useLoading/useTableContext`；分页字段常量在 `components/Table/src/const.ts` 的 `APISETTING`。
- 附加：`components/editable/`（`EditableCell.vue`）、`components/settings/ColumnSetting.vue`（列显隐/拖拽）。

```vue
<BasicTable :columns="columns" :request="loadDataTable" :row-key="(row) => row.id"
  ref="actionRef" :actionColumn="actionColumn" :scroll-x="scrollX"
  @update:sorter="handleUpdateSorter" />
<!-- 刷新：actionRef.value.reload() -->
<!-- 操作列：actionColumn.render 中用 h(TableAction, { actions: [{ label, onClick, auth, ifShow }] }) -->
```

### 7.2 表单 `components/Form`

- 导出：`BasicForm`（`src/BasicForm.vue`）、`useForm`。
- `src/types/form.ts`：`FormSchema{ field, label, component, componentProps, rules, giProps, auth, ifShow, ... }`、`FormProps`、`FormActionType`、`UseFormReturnType`。
- 用法：schema 驱动 + 自定义插槽。

```ts
const [register, methods] = useForm({ gridProps, labelWidth, schemas });
// <BasicForm @register="register" @submit="..." @reset="..." />
```

### 7.3 弹窗 `components/Modal`

- 导出 `basicModal`（注意导出名为小写）与 `useModal`；`useModal` 返回 `[register, { setProps, openModal, closeModal, setSubLoading }]`。
- 实践中，多数 CRUD 页面直接使用 Naive 的 `<n-modal preset="dialog">`。

### 7.4 其它

`Upload`（image/file）、`FileChooser`（文件/图片选择器，支持 `maxNumber/fileType`）、`Editor`（Quill）、`DatePicker`（含范围）、`CitySelector`（省市区）、`ComplexMemberPicker`（人员选择）、`IconSelector`、`CountTo`、`SvgIcon`、`LoadingContent`、`Lockscreen`、`YamlEditor`（CodeMirror）。

---

## 8. 字典机制 `src/store/modules/dict.ts`

- 类型：`DictType = DefaultDictType | EnumsDictType | FuncDictType | string`；`Option{ label, value, key, listClass, extra }`。
- **`loadOptions(types)`**：调 `Dicts({ types })`（`api/dict/dict.ts` → `GET /dictData/options`），合并进 `this.options` 并持久化 `CURRENT-DICT`。
- **`getOption(type)`**：返回 `ComputedRef<SelectMixedOption[]>`；**`getOptionUnRef(type)`**：返回普通数组（v-for 场景）。
- **`getLabel(type, value)`**：按 value 取 label；另有 `getType`（取 `listClass`）、`getExtra`、`hasValue`、`genOption`。
- **`checkOptionValue(type)`**：优先查本地字典 `src/enums/localDictEnum.ts`，再查 store。

页面用法：

```ts
const dict = useDictStore();
onMounted(() => dict.loadOptions(['sys_normal_disable', 'sys_switch']));
// 下拉：<n-select :options="dict.getOptionUnRef('sys_switch')" />
// 单选：<n-radio-button v-for="s in dict.getOptionUnRef('sys_normal_disable')" ... />
// 表格标签：renderOptionTag('sys_normal_disable', row.status)
```

> 规范：**禁止在模板中硬编码字典文本**，必须经 `dict.getLabel()` / `renderOptionTag()` 渲染；枚举值由后端 `consts` + `dict.RegisterEnums` 统一下发。

---

## 9. 权限体系

| 层级 | 实现 |
|------|------|
| 路由级 | 后端菜单树 → `asyncRouteStore.generateRoutes` → `routeFilter`（依据 `meta.permissions`） |
| 元素级 | 指令 `v-permission`（`directives/permission.ts`，`binding.value={action,effect}`，`effect==='disabled'` 时置灰否则移除）；`usePermission().hasPermission/hasEveryPermission/hasSomePermission` 配合 `v-if` |
| 表格级 | `column.auth`（列权限）、`TableAction` 的 `action.auth` / `ifShow`（`utils/hotgo.ts` 的 `isIfShow`） |

> 本项目**没有 `v-auth` 指令**，等价能力为 `v-permission`；权限数据来源是 `userStore.permissions`（`GetInfo` 返回）。

---

## 10. hooks / directives / utils / settings / locale

### hooks `src/hooks`

| 分组 | 内容 |
|------|------|
| `common/` | `useBoolean`、`useContext`、`useCountDown`、`useLoading`、`useLoadingEmpty`、`useSendCode`、`useSorter`（多列排序，配 `BasicTable @update:sorter`） |
| `core/` | `useTimeout` |
| `event/` | `useBreakpoint`、`useEventListener`、`useWindowSizeFn` |
| `setting/` | `useGlobSetting`（读 `VITE_GLOB_*`）、`useProjectSetting`、`useDesignSetting` |
| `web/` | `useECharts`、`usePage`（分页/查询通用封装）、`usePermission` |
| 顶层 | `use-async`、`useBattery`、`useCreateScript`、`useDomWidth`、`useEcharts`、`useOnline`、`useTime`、`useBoolean` |

### directives `src/directives`

`permission`（`v-permission`）、`copy`（`v-copy`）、`debounce`、`throttle`、`draggable`、`clickOutside`；统一在 `plugins/directives.ts` 的 `setupDirectives` 注册。

### utils `src/utils`

| 文件 | 说明 |
|------|------|
| `index.ts` | `renderIcon/renderNew/renderBadge/renderTag/renderOptionTag`（字典标签）、`renderImage/renderImageGroup`、`renderFile/renderFileGroup`、`renderPopoverMemberSumma`、`deepMerge` |
| `hotgo.ts` | `adaModalWidth`、`adaTableScrollX`（按权限计算表格横向宽）、`isIfShow`、`timeFix`、`convertListToTree`、`getTreeKeys`（`getOptionLabel/getOptionTag` 已 `@deprecated`） |
| `http/` | Axios 封装（第 6 节） |
| `websocket/` | WS 客户端（第 11 节） |
| `Storage.ts` | localStorage 封装 |
| `env.ts` | `getAppEnvConfig`（读取运行期 `app.config.js`） |
| `dateUtil.ts` | `defRangeShortcuts` 等日期工具（搜索栏日期范围常用） |
| 其它 | `is/`（类型判断）、`lib/echarts.ts`、`validateUtil.ts`、`downloadFile.ts`、`encrypt.ts`、`urlUtils.ts`、`tree.ts`、`array.ts`、`domUtils.ts`、`Drag.ts`、`icons.ts`、`log.ts` |

### settings `src/settings`

- `projectSetting.ts`：`navMode: 'vertical'`、`navTheme: 'dark'`、`permissionMode: 'BACK'`、`multiTabsSetting`、`menuSetting`、`crumbsSetting`、页面动画。
- `designSetting.ts`：`darkTheme: false`、`appTheme: '#2d8cf0'`、`appThemeList`（18 色）。
- `componentSetting.ts`、`animateSetting.ts`。

### locale `src/locale`

- `createI18n({ legacy: false, locale: 'zh-CN', fallbackLocale: 'zh-CN', messages: { en, 'zh-CN', 'zh-TW' } })`，导出 `t()`；`unplugin-auto-import` 使 `t` 全局可用。
- 语言切换由 `store/modules/i18n.ts` + `layout/components/Header` 完成，`initLocale()` 会结合后端 `loginConfig.i18nSwitch/defaultLanguage`。

---

## 11. WebSocket `src/utils/websocket/`

- `main.ts` 在 `router.isReady()` 后调用 `setupWebsocket()`；连接 `new WebSocket(`${config.wsAddr}?authorization=${token}`)`；未登录或地址为空则 10s 后重连。
- 心跳：`heartCheck`（5s ping，超时关闭）+ `reconnect`（`lockReconnect` 防抖）+ `window.onbeforeunload` 关闭。
- 消息分发：内部 `Map<event, handler>`；导出 `sendMsg(event, data, isRetry)`、`addOnMessage/removeOnMessage/getAllOnMessage`。
- `registerMessage.ts` 的 `registerGlobalMessage` 注册全局事件：`ping`、`kick`（`SocketEnum.EventKick` → 登出 + 清空标签页 + reload）、`notice`（→ `notificationStore.triggerNewMessages`）。
- 事件枚举 `src/enums/socketEnum.ts`：`ping/kick/notice/connected`、`admin/monitor/trends`、`admin/monitor/runInfo`、`admin/order/notify`。
- 业务示例：`views/addons/hgexample/portal/websocketTest.vue`、`views/monitor/serve-monitor/index.vue`、`views/asset/recharge/recharge.vue`。

> 项目**未使用 SSE**（无 `EventSource` 实现），实时能力全部走 WebSocket。

---

## 12. 主题、暗黑模式与布局

- `App.vue`：`<NConfigProvider :theme="getDarkTheme" :theme-overrides="getThemeOverrides">`；`getDarkTheme` 依 `designStore.darkTheme` 取 `darkTheme`；`getThemeOverrides` 依据 `appTheme` 生成 `primaryColor/Hover/Pressed`（`lighten()`）以及滚动条、LoadingBar 覆盖；同时处理锁屏倒计时。
- 设置面板：`layout/components/Header/ProjectSetting.vue`（暗黑开关、主题色列表、导航模式 `togNavMode/togNavTheme`）。
- 布局：`layout/index.vue`（主框架）+ `parentLayout.vue`（二级父级）+ `components/{Header,Logo,Main,Menu,TagsView}`。

---

## 13. 典型 CRUD 页面模式（三件套）

以 `src/views/addons/hgexample/table/` 为例（同一模式见 `curdDemo/`、`normalTreeDemo/`、`optionTreeDemo/`、`testCategory/`、`org/dept/`、`org/user/`、`log/log/`、`apply/notice/` 等）：

| 文件 | 职责 |
|------|------|
| `model.ts` | `State` 接口 + `defaultState` + `newState()`（`cloneDeep`）+ `rules`（表单校验）+ `schemas`（搜索栏 `FormSchema[]`）+ `columns`（表格列）+ `loadOptions()`（字典预加载） |
| `index.vue` | `BasicForm`（搜索）+ `BasicTable`（`:request="loadDataTable"`）+ 操作列 `TableAction`；顶部按钮用 `#tableTitle` 插槽；`onMounted` 调 `loadOptions()`；操作后用 `actionRef.value.reload()` 刷新 |
| `edit.vue` | `<n-modal preset="dialog">` + `<n-form :model="params" :rules="rules">`；props `showModal/formParams`，`emit('reloadTable','updateShowModal')`；`formRef.value.validate()` 后调 `Edit()` |

约定：

- 搜索栏 schema 中字典下拉使用 `options: computed(() => dict.getOptions('XxxOptions'))` 保持响应式；
- 表格列中字典渲染用 `dict.getLabel('XxxOptions', row.status)` 或 `renderOptionTag`；
- 操作按钮权限用 `hasPermission(['/xxx/edit'])`，与后端路由路径一致（禁止硬编码路由）；
- 内联 style 禁止；组件优先用 Naive UI 原生或项目封装组件（`BasicTable`/`BasicForm`/`n-modal` 等）。

---

## 14. 前端待清理项（分析结论）

1. `src/router/modules/` 目录不存在，导致 `asyncRoutes` 恒为空数组（`permissionMode` 为 `BACK` 时无影响，但代码语义误导）。
2. `src/store/modules/index.ts` 中 `import.meta.globEager('./*/index.ts')` 与若干 `import './async-route'` 路径与真实文件名不符（历史遗留，未被 `setupStore` 使用）。
3. `src/utils/hotgo.ts` 的 `getOptionLabel/getOptionTag` 已标注 `@deprecated`，应统一迁移到 `dict` store。
4. `src/enums/localDictEnum.ts` 与后端字典存在双份枚举定义，需注意同步。
5. `build/vite/plugin/styleImport.ts`（ant-design-vue 样式按需）与 `src/plugins/customComponents.ts`、`globalMethods.ts` 为空实现/未启用，可清理。
