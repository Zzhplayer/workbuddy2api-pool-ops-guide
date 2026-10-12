# 6. 每日签到 / 领积分：定时 + 手动

---

## 6.1 先弄清「积分从哪来」

积分不是只有一个来源。实测下来**大头不是签到**：

| 来源 | 单次量级 | 频率 | 说明 |
|---|---|---|---|
| 签到 `checkin` | 很小 | 每天 | 幂等（重复调用返回「今天已签到」） |
| 活动 `activity` | 小 | 每天 | 有连续天数 `streak` |
| 出差/日常 `travel` | 小（`reward=N`） | 每天 | |
| **成长任务队列 `tasks`** | **+100 / +300** | 每个任务一次 | **主要来源**，见 6.3 |
| 学堂 `school` | 代金券 | 活动期 | `/school/status` + `/school/vouchers` |

## 6.2 网关内置定时（第一道）

`/panel/api/config` 里能看到生效的定时表：

```json
{"checkin_hours":[9,21],"travel_hours":[9,21],"activity_hours":[10],
 "keepalive_hours":[22],"blackcat_hours":[23],"balance_refresh_minutes":5}
```

执行痕迹写在 `logs/err.log`：

```
2026/10/12 09:00:00 travel  <ACCOUNT>(<UID8>): depart ok location=4
2026/10/12 10:00:00 activity <ACCOUNT>(<UID8>): streak days=3
2026/10/12 09:53:03 checkin  <ACCOUNT>(<UID8>): 今天已签到（幂等）
```

## 6.3 成长任务队列（第二道，也是真正的大头）

```
POST /panel/api/tasks/scan_all     # 扫描各账号可做的任务 → 入队
POST /panel/api/tasks/run_queue    # 逐个执行并自动领奖
GET  /panel/api/tasks/queue        # 队列状态 {running, started, total, seq, conc, items[]}
```

执行痕迹：

```
2026/10/12 10:08:13 panel: 队列 growth uid=<UID> code=chat_5: 已补报 5 条对话事件；自动领奖 +100 分 +5 能（进度 5/5）
2026/10/12 10:09:09 panel: 队列 growth uid=<UID> code=create_canvas: 已上报 ...；自动领奖 +300 分 +5 能（进度 1/1）
```

> **实测一次触发**：队列 68 个任务逐个跑，单账号 **+1100 分**。
> 队列是**串行**的（`conc=1`），按账号排队，跑完需要几分钟——所以定时任务触发后别急着看结果。

## 6.4 外部定时任务（第三道：兜底 + 可回看）

网关自身定时够用，但外部计划任务能解决两件事：
① 网关重启/漏跑时补上；② 留下**可回看的日志**。

### 脚本（`<GATEWAY_DIR>/claim-all.sh` / `.ps1` 要点）

```bash
#!/usr/bin/env bash
set -u
BASE="http://127.0.0.1:<PORT>"
# 密钥从网关配置读，不硬编码
KEY=$(python3 -c "import json;print(json.load(open('<GATEWAY_DIR>/config.json'))['api_key'])")

# 网关不在就跳过，别把错误写成一堆 FAIL
curl -sf "$BASE/healthz" >/dev/null || { echo "SKIP gateway down"; exit 0; }

for p in checkin_all activity_all travel_all school/run_all tasks/scan_all tasks/run_queue keepalive_all; do
  code=$(curl -s -o /tmp/r.json -w '%{http_code}' -X POST -H "authorization: Bearer $KEY" "$BASE/panel/api/$p")
  case "$code" in
    200)  echo "OK   $p  $(head -c 120 /tmp/r.json)";;
    409)  echo "OK   $p  409 already running (幂等)";;   # ← 队列已在跑，算成功
    *)    echo "FAIL $p  HTTP $code";;
  esac
done

# 余额快照，便于回看每日趋势
curl -s -X POST -H "authorization: Bearer $KEY" "$BASE/panel/api/balance_all" \
  | python3 -c "import json,sys;[print(a['nickname'],a['credits']) for a in json.load(sys.stdin)['accounts']]"
```

**顺序有讲究**：先签到/活动/出差 → 再学堂 → 再任务队列 → 最后保活（续期 token）+ 余额快照。

### Windows 计划任务

```powershell
$action = New-ScheduledTaskAction -Execute 'powershell.exe' `
  -Argument '-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File <GATEWAY_DIR>\claim-all.ps1'
$t1 = New-ScheduledTaskTrigger -Daily -At 09:05
$t2 = New-ScheduledTaskTrigger -Daily -At 21:05
$set = New-ScheduledTaskSettingsSet -StartWhenAvailable -ExecutionTimeLimit (New-TimeSpan -Minutes 15) `
  -MultipleInstances IgnoreNew -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
Register-ScheduledTask -TaskName '<Svc>-DailyClaim' -Action $action -Trigger $t1,$t2 -Settings $set -Force
```

### Linux cron

```cron
5 9,21 * * *  /bin/bash <GATEWAY_DIR>/claim-all.sh >> <GATEWAY_DIR>/logs/claim.log 2>&1
```

## 6.5 三个实现细节（都踩过）

1. **409 要当成功**：`run_queue` 在上一次还没跑完时返回 409 Conflict —— 这是幂等保护，不是失败。
2. **PowerShell 5.1 会把 UTF-8 响应解成乱码**（响应头没带 charset 时按 Latin-1 解）。
   用 `Invoke-WebRequest` 取原始字节再显式解码：

   ```powershell
   $r = Invoke-WebRequest -Method Post -Uri $uri -Headers $hdr -UseBasicParsing
   $json = [System.Text.Encoding]::UTF8.GetString($r.RawContentStream.ToArray()) | ConvertFrom-Json
   ```

3. **脚本文件存成 UTF-8 with BOM**（Windows PowerShell 5.1 读无 BOM 的中文注释会乱码）。

## 6.6 登录/续期与「要不要反复登录」

- 领取全程**不需要打开客户端、不需要扫码**：网关用 `auths/` 里的 token 直接调上游接口。
- **扫码只在第一次加号时发生一次**。
- `keepalive_all`（或到点的保活任务）会**滚动换发 refreshToken**，到期日随之后移 →
  只要网关在跑，**登录不会过期**（详见 [gateway.md#17-token-生命周期](gateway.md)）。

## 6.7 验证

```bash
# ① 今天到底领了什么（权威来源是网关日志）
grep -E 'checkin|activity|travel|panel: 队列 growth' <GATEWAY_DIR>/logs/err.log | tail -n 20

# ② 队列状态
curl -s -H "authorization: Bearer $API_KEY" http://127.0.0.1:<PORT>/panel/api/tasks/queue

# ③ 余额有没有涨（注意：余额是 5 分钟粒度的缓存，别用秒级差值下结论）
curl -s -X POST -H "authorization: Bearer $API_KEY" http://127.0.0.1:<PORT>/panel/api/balance_all

# ④ 计划任务是否真的跑了（Windows）
Get-ScheduledTaskInfo -TaskName '<Svc>-DailyClaim' | Select LastRunTime, LastTaskResult, NextRunTime
#   LastTaskResult = 0 即成功
```

**观测建议**：把「今日领取 + 近 7 天漏领」做进看板（见 [panel-plugin.md](panel-plugin.md) 的每日领取区块），
比翻日志快得多——漏领那天一眼就能看出来。
