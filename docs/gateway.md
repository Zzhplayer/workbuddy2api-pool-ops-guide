# 1. 反代网关：部署、常驻、配置、管理 API

把只有网页/客户端的上游服务，包成本地 **OpenAI 兼容** HTTP 接口。

---

## 1.1 目标与前置

**目标**：`http://127.0.0.1:<PORT>/v1/chat/completions` 可用，且**进程掉了能自己爬起来**。

**前置**：

- 一台常开的机器（Windows / Linux 都行；下面两条路径都写）
- 上游服务的**至少一个账号**（能登录）
- 网关可执行文件（从项目 Release 取；下文记作 `<GATEWAY_EXE>`）

目录约定（下文统一用占位符）：

```
<GATEWAY_DIR>/
├── <GATEWAY_EXE>          # 网关可执行文件
├── config.json            # 主配置（首次启动会自动生成推荐值）
├── auths/                 # 每个账号一份 token（登录后写入，自动滚动续期）
├── data/                  # 运行态：state.json / 我们后续加的降级链配置
└── logs/
    ├── out.log            # 请求流水（每发一行）
    ├── err.log            # 任务行 + 上游错误（签到/活动/限流都在这里）
    └── watchdog.log       # 看门狗日志
```

---

## 1.2 部署（Windows）

```powershell
# 1) 首次启动：不存在 config.json 时会自动生成推荐配置
cd <GATEWAY_DIR>
.\<GATEWAY_EXE> -config config.json
# 看到监听日志后 Ctrl+C，进入下一步改配置

# 2) 健康检查（期望 {"healthy":N,"total":N,"service":"..."}）
curl.exe -s http://127.0.0.1:<PORT>/healthz
```

**脱离 shell 常驻**（关键，否则关掉终端就没了）：

```powershell
# 用 cmd 的 start /b 真正脱离调用者进程树
cmd /c "start /b /min cmd /c `"cd /d <GATEWAY_DIR> && <GATEWAY_EXE> -config config.json > logs\out.log 2> logs\err.log`""
```

> ⚠️ 实测坑：如果用「PowerShell 起 → 外层再被 job 对象回收」的方式启动，
> 网关会**跟着一起被杀**。所以要：
> ① 用 `cmd /c start /b` 脱离进程树；② 再加一层**看门狗**兜底（见 1.5）。

## 1.3 部署（Linux，systemd）

```ini
# /etc/systemd/system/<svc>.service
[Unit]
Description=Reverse-proxy gateway
After=network-online.target

[Service]
Type=simple
WorkingDirectory=<GATEWAY_DIR>
ExecStart=<GATEWAY_DIR>/<GATEWAY_EXE> -config config.json
Restart=always
RestartSec=5
StandardOutput=append:<GATEWAY_DIR>/logs/out.log
StandardError=append:<GATEWAY_DIR>/logs/err.log

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now <svc>
systemctl status <svc> --no-pager
```

---

## 1.4 配置文件要点（`config.json`）

只列**会影响行为**的键（值按自己需要调）：

| 键 | 作用 | 建议 |
|---|---|---|
| `listen` | 监听地址端口 | 只本机用就绑 `127.0.0.1:<PORT>` |
| `api_key` | 本地网关的访问密钥 | **长随机串**；不要提交到版本库 |
| `auth_dir` / `state_file` | token 目录 / 运行态文件 | 保持默认即可 |
| `cooldown.soft_rate` | 单次限流后的冷却时长 | `600s` 起步 |
| `cooldown.soft_rate_max` | 冷却上限（防止一次限流就凉很久） | **`30m`**（默认 2h 偏长） |
| `pool.max_in_flight` | 每账号并发上限 | `3` 左右；太大容易触发上游风控 |
| `breaker_*` / `degrade_*` | 熔断与降级阈值 | 默认即可，观察后再调 |
| `session_sticky` | 同一会话粘在同一账号的时长 | `30m`（长上下文时命中缓存更快） |
| `schedule.checkin_hours` | 签到时间（数组） | `[9, 21]` |
| `schedule.activity_hours` | 活动时间 | `[10]` |
| `schedule.travel_hours` | 出差/日常任务 | `[9, 21]` |
| `schedule.keepalive_hours` | **保活 + token 续期** | `[22]`（别关，见 1.7） |
| `schedule.balance_refresh_minutes` | 余额刷新间隔 | `5` |

> 判据优先级：**先看 `/status` 的 `rate_limited_models`，再看日志**。
> 网关自己就知道「哪个账号对哪个模型被限流」，不需要你去猜。

## 1.5 常驻守护：看门狗（强烈建议）

一个 30 秒自检的常驻循环，发现端口没在跑就调用幂等的启动脚本补位：

```powershell
# watchdog.ps1（要点）
$mutex = New-Object System.Threading.Mutex($false, 'Local\<Svc>-Watchdog-Singleton')
if (-not $mutex.WaitOne(0)) { exit 0 }          # 单实例闸门，避免叠加

while ($true) {
  $ok = $false
  try {
    $r = Invoke-WebRequest "http://127.0.0.1:<PORT>/healthz" -TimeoutSec 5 -UseBasicParsing
    $ok = ($r.Headers['X-Service'] -eq '<SERVICE_ID>')   # 用服务身份标识，而不是只看端口
  } catch { }
  if (-not $ok) { & "<GATEWAY_DIR>\start.ps1" }          # start.ps1 必须幂等
  Start-Sleep -Seconds 30
}
```

Linux 侧等价物就是 `Restart=always`（1.3），不需要额外看门狗。

**注册为登录时启动 + 失败重启**（Windows 计划任务）：

```powershell
$action  = New-ScheduledTaskAction -Execute 'powershell.exe' `
  -Argument '-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File <GATEWAY_DIR>\watchdog.ps1'
$trigger = New-ScheduledTaskTrigger -AtLogOn
$set     = New-ScheduledTaskSettingsSet -RestartCount 999 -RestartInterval (New-TimeSpan -Minutes 1) `
  -ExecutionTimeLimit (New-TimeSpan -Days 0) -MultipleInstances IgnoreNew
Register-ScheduledTask -TaskName '<Svc>-Watchdog' -Action $action -Trigger $trigger -Settings $set -Force
```

> 实测有效：网关被外部原因杀掉后，看门狗在 **30 秒内**把它拉回来，
> `watchdog.log` 里会留下 `gateway NOT healthy (probe #1) -> running start.ps1` 这样的记录。

## 1.6 管理 API（脚本化的抓手）

全部用 `authorization: Bearer <API_KEY>`（`/healthz` 免鉴权）：

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/healthz` | 存活 + 账号总数（免鉴权） |
| GET | `/status` | **账号池权威状态**：每账号 `credits / cooling / until / rate_limited_models[{model,until,reset_at,reason}] / in_flight` |
| GET | `/v1/models` | OpenAI 兼容模型清单（含 `tags: badge:<文字>:#RRGGBB` 促销徽章） |
| GET | `/panel/api/overview` | 账号概览 + 版本/运行时长 |
| GET | `/panel/api/models` | 模型明细：倍率 `credits`、促销 `promo_*`、上下文/输出上限、能力位 |
| GET | `/panel/api/usage` | 用量：`totals / by_account / by_model / series / buckets` |
| GET | `/panel/api/packages` | **积分包明细**（每个包的 `remain/used/size/end_time`） |
| GET | `/panel/api/config` | 生效配置（只读，用于核对改动是否生效） |
| POST | `/panel/api/checkin_all` | 一键签到（全账号，幂等） |
| POST | `/panel/api/activity_all` | 一键活动 |
| POST | `/panel/api/travel_all` | 一键日常/出差任务 |
| POST | `/panel/api/school/run_all` | 一键学堂任务 |
| POST | `/panel/api/tasks/scan_all` → `/tasks/run_queue` | 扫任务 + 跑队列（**成长任务，主要积分来源**） |
| POST | `/panel/api/keepalive_all` | 保活 + **滚动续期 token** |
| POST | `/panel/api/balance_all` | 强制刷新余额（返回每账号 `model_costs`） |

**读密钥的正确姿势**（不要硬编码）：

```bash
API_KEY=$(python -c "import json;print(json.load(open('<GATEWAY_DIR>/config.json'))['api_key'])")
curl -s -H "authorization: Bearer $API_KEY" http://127.0.0.1:<PORT>/status
```

## 1.7 token 生命周期（决定「要不要反复登录」）

`auths/<account>.json` 里是一对 token：

```
accessToken   有效期 ~28 天
refreshToken  有效期 ~30 天
```

**关键实测**：`keepalive_all`（或 `schedule.keepalive_hours` 到点自动跑）会**滚动换发新的 refreshToken**，
到期日随之后移（例：`11-10 → 11-11`）。也就是说：

> 只要网关在跑、保活任务没关，**登录不会过期**，不需要周期性重新登录。
> 只有账号被踢下线 / 改密码 / 上游风控才需要重来一次。

验证方法（不必看明文 token，比对指纹即可）：

```python
import json, base64, hashlib, glob
def jt(t): p=t.split('.')[1]; p+='='*(-len(p)%4); return json.loads(base64.urlsafe_b64decode(p))
for f in glob.glob('<GATEWAY_DIR>/auths/*.json'):
    a = json.load(open(f, encoding='utf-8'))['auth']
    print(f, jt(a['refreshToken'])['exp'], hashlib.md5(a['refreshToken'].encode()).hexdigest()[:8])
# 跑一次 keepalive_all 再打印：exp 变新 + 指纹变 → 滚动续期成立
```

## 1.8 部署完成的自检清单

```bash
# ① 存活
curl -s http://127.0.0.1:<PORT>/healthz
# ② 账号池健康（healthy == total）
curl -s -H "authorization: Bearer $API_KEY" http://127.0.0.1:<PORT>/status | \
  python -c "import json,sys;d=json.load(sys.stdin);print('healthy',d['healthy'],'/',d['total'])"
# ③ 真打一发（别只看 /healthz）
curl -s -H "authorization: Bearer $API_KEY" -H 'content-type: application/json' \
  -d '{"model":"<MODEL_ID>","messages":[{"role":"user","content":"hi"}],"max_tokens":8}' \
  http://127.0.0.1:<PORT>/v1/chat/completions
# ④ 日志在写
tail -n 3 <GATEWAY_DIR>/logs/out.log
# ⑤ 看门狗在跑（杀掉网关进程，等 30~40 秒，应该自动回来）
```
