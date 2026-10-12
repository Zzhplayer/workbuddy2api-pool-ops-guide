# 2. 账号池轮询 + 3. 模型降级链

---

# 2. 账号池轮询

## 2.1 先搞清「限流」到底是什么

上游的 429 长这样（关键在 `code:6004` 和那句「将在 …重置」）：

```json
{"code":6004,"msg":"您的使用量已超出频率限制，将在 2026-10-12 11:07:22 UTC+8 重置，您也可以切换其他模型继续使用"}
```

**它是「账号 × 模型」独立计量的**，不是账号总额度用光。由此推出三条重要结论：

1. **换个模型**就能继续用（同一账号）；
2. **换个账号**也能继续用（同一模型）；
3. 「池子里还有余额」和「现在能不能发请求」**是两件事**——别把额度当成可用性。

## 2.2 网关侧的池化能力（配置项）

| 机制 | 键 | 说明 |
|---|---|---|
| 加权轮询 | `pool` 权重 | 按账号表现分配流量 |
| 并发上限 | `pool.max_in_flight` | 每账号同时在飞的请求数 |
| 会话粘性 | `session_sticky` | 同一会话固定账号，利于上游缓存命中 |
| 熔断 | `breaker_threshold` / `breaker_cooldown` | 连续失败 N 次 → 冷却该账号 |
| 降级 | `degrade_threshold` / `degrade_cooldown` | 表现差 → 降权一段时间 |
| 软冷却 | `cooldown.soft_rate` / `soft_rate_max` | 单次限流后的冷却与其上限 |

> `soft_rate_max` 别设太大：设 2h 时，一次限流就会让账号凉两小时；
> 调到 `30m` 后恢复快得多（上游真正的重置时间由 `reset_at` 给出，见下）。

## 2.3 可用性的权威判据：`/status`

```bash
curl -s -H "authorization: Bearer $API_KEY" http://127.0.0.1:<PORT>/status
```

每个账号里有一段**最值钱的数据**：

```json
{
  "nickname": "<ACCOUNT>",
  "credits": 1909,
  "cooling": false,
  "rate_limited_models": [
    {"model": "deepseek-v4.1-flash",
     "until":   "2026-10-11T21:37:06+08:00",   // 网关自己的冷却到期
     "reset_at":"2026-10-12T11:07:22+08:00",   // 上游给的恢复时间
     "reason":  "6004 model rate limit"}
  ]
}
```

**注意两点**：

- 这里的 `model` 名**不带 realm 前缀**（`deepseek-v4.1-flash`），而 DSH 里的模型 id 带（`cn:deepseek-v4.1-flash`）→ 比较前要归一化；
- 网关会**在 `until` 到点后自己重试并更新这条记录**，所以它是自我更新的权威源，**不需要你去探测烧额度**。

## 2.4 加号

```bash
./add-account.sh --realm cn          # 打开一次性登录页；登录后 token 落到 auths/
```

要点：

- **必须是另一个账号**（同一个号只会重新授权已有账号）；
- 用**干净的浏览器上下文**（InPrivate/无痕）避免自动复用旧会话；
- 加完立刻核对 `/status` 的 `total` 是否 +1、`healthy` 是否跟上；
- 多账号的价值 = 把「同一模型同时可服务的账号数」做大，直接降低 429 概率。

---

# 3. 模型降级链

目标：**主力模型不可用时自动降档，一恢复立刻切回**，全程不用人管。

## 3.1 设计（为什么这么设计）

```
chain = [ 主力, 快速备用, 免费/慢速兜底, 夜间免费 ]
```

| 组件 | 做法 | 为什么 |
|---|---|---|
| **判据** | 读 `/status`：某模型「未被限流的账号数 < 阈值」→ 判不可用 | 权威、零成本、自我更新；比自己发探针更准 |
| **阈值** | `minAvailable = 2`（池子只有 1 个号时自动降为 1） | 实测：网关认为还剩 1 个号可用时，真实请求仍会 429 |
| **防抖** | 记住上游给的 `reset_at`（sticky），网关重启后短暂「放行」也不切回 | 网关重启会清空内存里的限流记录 → 不防抖就会来回抖 |
| **解除** | 只在**日志里出现该模型的真实 200** 或 `reset_at` 已过 → 解除 sticky | 「真的成功过」才算恢复 |
| **动作** | 用宿主 API 把**活跃会话**的模型切到目标档 | 会话级切换，客户端无感 |
| **范围** | 只动「正在跑」或「最近 N 分钟活跃」的会话 | 休眠会话的模型是用户自己选的，不该被改 |
| **落盘** | 切换记录写 `data/fallback-state.json` | 插件重载/重启后仍能还原 |
| **配置** | 链写在 `data/fallback-config.json`，每 30 秒重读 | 改一行即生效，不用重启 |

## 3.2 配置文件（可编辑，热生效）

`<GATEWAY_DIR>/data/fallback-config.json`：

```json
{
  "chain": ["cn:<MAIN>", "cn:<FAST_BACKUP>", "cn:<FREE_SLOW>", "cn:<NIGHT_FREE>"],
  "minAvailable": 2,
  "enabled": true,
  "windowMinutes": 120
}
```

## 3.3 实现要点（可直接移植的片段）

### ① 判定可用性（读 `/status`，带 sticky 防抖）

```js
const bare = (m) => String(m || '').replace(/^(cn|global):/, '')   // 归一化 realm 前缀

async function refreshAvail() {
  const st = await getJson('/status', true)
  const healthy = (st.json.accounts || []).filter((a) => !a.disabled)
  const agg = {}
  for (const m of chain) agg[m] = { total: healthy.length, limited: 0, resets: [], untils: [] }
  for (const a of healthy) {
    for (const x of (a.rate_limited_models || [])) {
      const hit = chain.find((m) => bare(m) === x.model)     // ← 前缀归一化在这里
      if (!hit) continue
      agg[hit].limited += 1
      if (x.reset_at) agg[hit].resets.push(x.reset_at)
      if (x.until) agg[hit].untils.push(x.until)
    }
  }
  for (const m of chain) {
    const s = agg[m]
    const free = s.total - s.limited
    const need = Math.min(Math.max(1, minAvailable), s.total)
    if (s.limited >= s.total || free < need) {
      const reset = s.resets.sort()[0] || null
      avail[m] = { ok: false, reason: `可用号不足 ${free}/${s.total}（阈值 ${need}）`, resetAt: reset }
      if (reset) sticky[m] = reset                 // ← 防抖：记住上游恢复时间
    } else if (sticky[m] && Date.parse(sticky[m]) > Date.now()) {
      avail[m] = { ok: false, reason: '网关已放行，但上游重置时间未到（需真实 200 才解除）' }
    } else {
      delete sticky[m]
      avail[m] = { ok: true, reason: `可用号 ${free}/${s.total}` }
    }
  }
}
```

### ② 日志快速信号（只用来触发重判，不直接改判定）

```js
// 日志里出现新的 429/503/200 → 标记 dirty，下一拍去刷 /status
// 关键：不要因为「某一个账号 429」就把整个模型判死
for (const line of tail(<GATEWAY_DIR>/logs/out.log)) {
  const e = parseLine(line)                       // | #N | HH:MM:SS | model | mode | status | acct(uid8) | ...
  if (e.status === 200) { delete sticky[e.model]; dirty = true }   // 真实成功 = 确实恢复
  else if (e.status === 429 || e.status === 503) dirty = true
}
```

### ③ 会话级切换（宿主服务）

```
sessionController.list({})              → { items: [{ sessionId, running, updatedAt, blank, projections }] }
sessionController.selectModel({ sessionId, provider, model })  → { selected: {...} }
```

```js
const svc = sessionController()                       // 见下「服务拿不到的坑」
const items = (await svc.list({})).items || []
for (const it of items) {
  if (!it.sessionId || it.blank || it.origin === 'subagent') continue
  const fresh = it.running === true ||
                (it.updatedAt && Date.now() - Number(it.updatedAt) < windowMinutes * 60000)
  if (!fresh) continue                                // 休眠会话不碰
  const cur = modelOf(it)                             // 从 projections 里取 {provider, model}
  if (!cur || cur.provider !== provider) continue
  const inChain = chain.includes(cur.model)
  const wasSwitched = !!switched[it.sessionId]
  if (!inChain && !wasSwitched) continue              // 用户自己选的别的模型 → 不碰
  if (cur.model === target) continue                  // 已在目标上：保留还原记录，别删
  if (switched[it.sessionId]?.to === target) continue // 已下发过、pending 还没生效 → 不重复调
  await svc.selectModel({ sessionId: it.sessionId, provider, model: target })
  switched[it.sessionId] = { from: cur.model, to: target, at: now() }
}
saveState()                                           // 落盘，重载后仍能还原
```

### ④ 「模型退役」也要迁移

如果某个模型**从链里删掉**了，那些还挂在它上面的会话不会自动被管到
（因为过滤条件是「在链内」）。补一条：

```js
if (!inChain && wasSwitched) { /* 曾经是我们切过去的 → 也要切到新目标 */ }
```

否则会出现「模型已退役，但几个会话永远留在旧模型上」。

### ⑤ 配置热重载（否则「改一行即生效」是假的）

```js
function reloadCfg() {
  const j = JSON.parse(readFileSync(configFile, 'utf8'))
  if (Array.isArray(j.chain) && j.chain.join('>') !== chain.join('>')) { chain = j.chain; changed = true }
  if (typeof j.minAvailable === 'number') minAvailable = j.minAvailable
  if (changed && !chain.includes(target)) target = chain[0]
}
// 每拍（30s）调用一次
```

## 3.4 验证清单

```bash
# ① 当前判定与目标
curl -s http://127.0.0.1:<PLUGIN_PORT>/<ROUTE>/fallback
#   期望: chain=[...] target=<主力> avail={每个档位 ok/reason/resetAt}

# ② 配置热生效（改 chain 后 30 秒内应该看到 cfgSource=file:reloaded + 一条重载事件）
# ③ 端到端：手动把目标切到第 2 档，看活跃会话是否跟着切
curl -s -X POST -H 'content-type: application/json' \
  -d '{"action":"switch","model":"<MODEL_2>"}' http://127.0.0.1:<PLUGIN_PORT>/<ROUTE>/fallback
# ④ 反向验证：主力恢复时是否自动切回（看事件日志里的「目标模型 X → Y」）
```

## 3.5 ⚠️ 成本账陷阱（会误导决策）

网关 `/panel/api/balance_all` 里有 `model_costs[{model, cost_per_1k, samples}]`。
它看起来像「每模型实测成本」，**但归因方式是把两次余额刷新之间的消耗记到最后一个使用的模型上**：

- 并发流量会被算到错误的模型头上；
- 每日领奖的**加分**也会被反向记进去；
- 实测出现过「标称免费的模型显示 0.13 分/千token，而主力模型只显示 0.002」这种完全反过来的结果。

**正确做法**：受控测量 —— 沉降过一个余额刷新周期 → 只打单模型 → 再等一个周期 → 读余额差，
并用请求日志核对窗口内该账号有没有别的流量。详见 [pitfalls.md](pitfalls.md#3-成本账归因陷阱)。
