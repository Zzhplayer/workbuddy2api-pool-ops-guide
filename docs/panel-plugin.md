# 5. 插件看板：把号池、额度、领取状态做成侧边栏面板

一个**免构建**（纯 JS）的 DSH 插件：宿主侧读网关数据 + 浏览器侧渲染面板，另带一个会话内工具。

---

## 5.1 插件形态与目录

```
<PLUGIN_DIR>/
├── package.json          # main / exports['./client'] / dsh.bundle.patch / dsh.client.inject
├── cordis.patch.yml      # 装配声明（bundle patch）
├── lib/
│   ├── index.js          # 宿主侧：数据聚合 + HTTP 路由 + 工具注册
│   └── client.js         # 浏览器侧：面板 UI
└── README.md
```

`package.json` 关键段：

```json
{
  "name": "dsh-<name>-pool",
  "type": "module",
  "main": "./lib/index.js",
  "exports": { ".": { "default": "./lib/index.js" }, "./client": { "default": "./lib/client.js" } },
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": { "platform": "web", "inject": ["@deepseek-ai/dsh-client-ui-slots"] }
  }
}
```

## 5.2 宿主侧骨架

```js
export const name = 'dsh-<name>-pool'
export const inject = ['tools', 'webServer']          // 只用声明过的服务

const DEFAULTS = {
  baseUrl: 'http://127.0.0.1:<PORT>',
  apiKeyFile: '<GATEWAY_DIR>/config.json',            // 从网关配置读 key，别硬编码
  logFile: '<GATEWAY_DIR>/logs/out.log',
  errFile: '<GATEWAY_DIR>/logs/err.log',              // 任务行（签到/领取）在这里
  route: '/<ROUTE>',
  cacheMs: 4000,
}

export function apply(ctx, config) {
  const cfg = Object.assign({}, DEFAULTS, config || {})
  // ① 注册 HTTP 路由（面板数据、诊断、控制）
  ctx.effect(() => register({ kind: 'exact', path: cfg.route + '/data', handler: dataHandler }),
             '<name>: data route')
  // ② 注册会话内工具（让 agent 也能查）
  ctx.effect(() => registerTool({ name: '<name>_pool', description: '...', parameters: {...},
                                  execute: async (args) => textReport(args.view) }),
             '<name>: tool')
}
```

**数据聚合**（并发拉、带短缓存，别让 5 秒轮询打爆网关）：

```js
const [health, overview, models, usage, packages, v1models, school, vouchers, queue] =
  await Promise.all([ getJson('/healthz', false), getJson('/panel/api/overview', true), /* ... */ ])
const logData = parseLog()          // 解析 out.log：账号×模型交叉表 + 最近请求
const daily   = parseDaily()        // 解析 err.log：签到/活动/领取记录
```

## 5.3 浏览器侧：slot 契约

```js
window.__ModuleLoader__.load({
  id: 'dsh-<name>-pool',
  factory: () => ({
    inject: ['slots'],
    apply(ctx) {
      // ① 侧边栏面板入口
      ctx.slots.register({ name: 'sidebar.panellist', id: '<PANEL_ID>', order: 30, label: '<面板名>' }, Panel)
      // ② 主区域：**key 必须与上面的 id 相同**，否则选中后面板区是空的
      ctx.slots.register({ name: 'main', key: '<PANEL_ID>' }, Dashboard)
    },
  }),
})
```

> 实测：`main` 的 key 写错时，点面板会「选中但什么都不显示」——
> 容器切了、内容没挂上。key 必须与 `sidebar.panellist` 的 `id` 完全一致。

面板 UI 用 `h('div', {...})` 建树，数据每 5 秒轮询 `/data` 重渲染。

## 5.4 五个真实踩坑（每个都花了时间）

### 坑 1：`style` 传字符串 → React #62 → **整个面板白屏**

```js
// ❌ 错误：dimmer 是颜色字符串
h('span', { style: dimmer }, '—')
// ✅ 正确
h('span', { style: { color: dimmer } }, '—')
```

React 错误 #62 = `The style prop expects a mapping ... not a string`。
**一个格子写错，整棵主树被卸载** → 面板主区域全空。修完加一道自检：

```bash
# 扫描可疑的字符串 style（我每次改完都跑）
grep -nE "style:\s*[A-Za-z_$][A-Za-z0-9_$]*\s*[,}]" lib/client.js
```

### 坑 2：热重载时 `duplicate exact route`

每次热重载都会重新注册同一路径 → 第二次起注册失败，
**如果注册失败抛错中断了 `apply`，后面的路由和工具就全都没注册**。修法：

```js
function purgeRoutes() {                       // 注册前清掉本插件前缀的残留
  for (const map of [ctx.webServer.exact, ctx.webServer.prefixes, ctx.webServer.upgrades]) {
    if (!map) continue
    for (const key of Array.from(map.keys())) if (String(key).startsWith(cfg.route)) map.delete(key)
  }
}
const register = (spec) => { try { ctx.webServer.register(spec) } catch (e) { say('route 注册失败: ' + e.message) } }
```

**要点**：`apply` 里每一步都要 try/catch —— 一个失败不能带走全部。

### 坑 3：访问未声明的服务会**抛错**，不是返回 undefined

```js
// ❌ 一条表达式里串联多个候选 → 第一个抛错，整条被 catch 掉，永远拿不到后面的
const svc = ctx.sessionController || (ctx.root && ctx.root.sessionController)

// ✅ 每条路径各自 try
function sessionController() {
  let svc = null
  try { svc = ctx.sessionController } catch { /* 未声明 → 抛错 */ }
  if (!svc) { try { svc = ctx.root && ctx.root.sessionController } catch { } }
  if (!svc) { try { svc = ctx.type === 'object' && ctx.get('sessionController') } catch { } }
  return svc && typeof svc.list === 'function' ? svc : null
}
```

> 症状是「服务明明存在，但插件一直说不可用」。定位方法：加一个诊断路由，
> 把候选路径逐个试一遍并报告 `has / own / proto / hasList` —— 一眼就能看出是哪条路径抛了。

### 坑 4：宿主看不到浏览器里的报错

面板白屏时宿主日志**一片干净**。做法：让 client 把生命周期阶段 POST 回宿主的一个诊断路由，
宿主缓存最近 N 条，然后 `GET /<ROUTE>/diag` 就能读到「面板是否挂载、容器尺寸、父链、React 报错」。

```js
// client
report('render-ok', { accounts: n, models: m, render: k })
// host
const clientReports = [];  // 上报缓存，/diag 暴露
```

### 坑 5：容器尺寸为 0 也会「什么都没有」

主区域容器是 flex 列布局（`display:flex; flex-direction:column; overflow:hidden`）。
面板根节点要有可用的高度上下文，否则内容高度算成 0。
用 DOM 探针把 `getBoundingClientRect()` 和父链一起上报，能立刻区分「没挂上」和「挂上了但高 0」。

## 5.5 面板应该展示什么（数据模型）

| 区块 | 数据源 | 说明 |
|---|---|---|
| 额度总账 | `/panel/api/overview` + `/panel/api/packages` | 每账号 `credits/credits_total/used` + 积分包明细（含到期时间） |
| 按模型 × 账号 | `out.log` 交叉表 + 模型倍率 | 请求数/token 数/估算分摊（**明确标注是估算**：上游不提供按模型额度） |
| 账号池 | `/status` | 冷却、限流模型、在飞请求、连续失败 |
| 轮询分布 | `out.log` 按账号聚合 | 验证「权重是否真的生效」 |
| 模型额度 | `/panel/api/models` + `/v1/models` 徽章 | 倍率、促销、上下文/输出、能力位 |
| **每日领取** | `err.log` 任务行 + `/school/status` + `/school/vouchers` | 今日签到/活动/出差/学堂、成长任务今日与累计、**近 7 天漏领**、代金券 |
| 降级链 | 插件自身状态 | chain / target / 每档可用性 / 已切会话数 / 事件流 |
| 最近请求 | `out.log` 尾部 | 时间/模型/状态/账号/token/速率/耗时 |

**关键诚实原则**：凡是估算值（按模型分摊、可跑次数）都在表头写清算法与误差来源，
不要把估算渲染成权威数字。

## 5.6 验证清单

```bash
# ① 插件已装配且 active
# ② 路由可达（HTTP 200 + 合理体积）
curl -s -o /dev/null -w '%{http_code} %{size_download}B\n' http://127.0.0.1:<HOST_PORT>/<ROUTE>/data
# ③ 诊断无异常
curl -s http://127.0.0.1:<HOST_PORT>/<ROUTE>/diag
# ④ 面板真的渲染（浏览器里点开侧边栏项，截图确认；宿主日志干净不代表没事）
# ⑤ 会话内工具可用：view=summary / daily / fallback 各来一次
```
