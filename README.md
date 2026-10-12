# 反代网关 + 号池看板 + 模型降级链 + 每日领取：可复现操作手册

把「一个上游 AI 服务」变成**本地 OpenAI 兼容网关**，再接入 DSH（DeepSeek Harness），
配上**账号池轮询**、**模型降级链**、**每日自动领积分**和**可视化看板**。

> 本仓库只写**方法**。所有本地路径、API Key、账号昵称/UID、会话 ID、Cookie 全部以占位符表示
> （`<GATEWAY_DIR>`、`<API_KEY>`、`<ACCOUNT>`、`<SESSION_ID>` …）。照抄时替换成你自己的值。

---

## 这套东西解决什么问题

| 痛点 | 解法 |
|---|---|
| 上游只给网页/客户端，没有 API | 本地**反代网关**把它包成 OpenAI 兼容接口（`/v1/chat/completions`） |
| 单账号会被限流（HTTP 429，按「账号 × 模型」独立计） | **多账号池** + 加权轮询 + 熔断 + 冷却 |
| 主力模型被限流时整个工作流停摆 | **模型降级链**：自动切到下一档，一恢复立刻切回 |
| 每天有签到/活动/成长任务能领积分，但手点很累 | **定时任务**批量领取（幂等，可手动补跑） |
| 池子里谁在冷却、谁在挨限流、今天漏领了没有，看不见 | **DSH 插件看板**：账号额度、按模型分摊、轮询分布、领取状态、降级链 |

## 架构

```
┌──────────────┐   OpenAI 兼容 HTTP   ┌────────────────────┐   上游协议    ┌──────────────┐
│ DSH / 任意   │ ───────────────────► │ 反代网关            │ ───────────► │ 上游 AI 服务  │
│ OpenAI 客户端│  /v1/chat/completions│ <GATEWAY_DIR>       │              │ (CodeBuddy 系)│
└──────┬───────┘                      │  · 多账号池 + 轮询   │ ◄─────────── └──────────────┘
       │                              │  · 限流/熔断/冷却    │
       │ provider 配置                 │  · 定时任务(签到等)  │
       │                              │  · 管理 API + 面板   │
       ▼                              └─────────┬──────────┘
┌──────────────────┐  读日志/读 /status          │
│ DSH 插件看板      │ ◄──────────────────────────┘
│ dsh-<name>-pool  │   + 调 sessionController 切会话模型
└──────────────────┘
```

## 六条成功路径

| # | 路径 | 文档 | 关键产物 |
|---|---|---|---|
| 1 | 部署反代网关并常驻守护 | [docs/gateway.md](docs/gateway.md) | 网关进程 + 看门狗 + 健康检查 |
| 2 | 账号池：加号、轮询、限流/冷却机制 | [docs/pool-and-fallback.md](docs/pool-and-fallback.md) | 多账号加权轮询 |
| 3 | DSH 接入 + 模型选择器上架 | [docs/dsh-provider.md](docs/dsh-provider.md) | `provider` 配置 + 模型清单 |
| 4 | 插件看板（账号额度/按模型分摊/领取状态） | [docs/panel-plugin.md](docs/panel-plugin.md) | 侧边栏面板 + 会话内工具 |
| 5 | 模型降级链（自动切档 + 自动切回） | [docs/pool-and-fallback.md#模型降级链](docs/pool-and-fallback.md) | 可编辑的 `fallback-config.json` |
| 6 | 每日签到/领积分（定时 + 手动） | [docs/daily-claim.md](docs/daily-claim.md) | 领取脚本 + 计划任务 |
| 7 | 踩坑与验证清单（含 7 个真实故障） | [docs/pitfalls.md](docs/pitfalls.md) | 故障 → 根因 → 修法 |

## 最短路径（30 分钟跑通）

```bash
# 1) 网关：配置 + 启动（Windows 用计划任务/看门狗常驻；Linux 用 systemd）
cd <GATEWAY_DIR>
<GATEWAY_EXE> -config config.json          # 首次会自动生成推荐配置
curl -s http://127.0.0.1:<PORT>/healthz    # 期望: {"healthy":N,"total":N,...}

# 2) 加账号（每个账号登录一次，token 会自动滚动续期，不必反复登录）
./add-account.sh --realm cn

# 3) 冒烟：打一发
curl -s http://127.0.0.1:<PORT>/v1/chat/completions \
  -H "authorization: Bearer <API_KEY>" -H 'content-type: application/json' \
  -d '{"model":"<MODEL_ID>","messages":[{"role":"user","content":"hi"}],"max_tokens":8}'

# 4) 接入 DSH：在 profile 配置里声明 provider（api / baseURL / models 必须显式写）
#    详见 docs/dsh-provider.md

# 5) 装看板插件（可选但强烈推荐）
#    详见 docs/panel-plugin.md

# 6) 定时领取
#    详见 docs/daily-claim.md
```

## 三个「不看文档就会踩」的坑（先看这三条）

1. **限流是「账号 × 模型」独立的**，不是总额度问题。
   池子里还有别的账号时，网关只会冷却「那个账号 + 那个模型」的组合。
   → 所以**加号**和**降级链**是两个正交的解法，都要有。

2. **余额是 5 分钟粒度的缓存值**。
   任何「打几发看余额掉多少」的成本测量都必须等过一个刷新周期，
   否则你会得出完全相反的结论（详见 [docs/pitfalls.md](docs/pitfalls.md) 的成本账陷阱）。

3. **`only_reasoning`（只能推理、不能关思考）的模型不适合当日常主力**。
   实测：TTFB 15~60 秒、吞吐 1~4 tok/s、单轮可达 5 分钟以上；
   而且**思考 token 吃满输出预算时，返回的 `content` 会是空的**——界面表现是
   「已完成，但什么都没有」。要么别当主力，要么把它放在降级链的靠后档位。

## 目录

```
.
├── README.md                     ← 你在这里
├── LICENSE
└── docs
    ├── gateway.md                反代网关：部署 / 守护 / 配置 / 管理 API
    ├── pool-and-fallback.md      账号池轮询 + 模型降级链（含实现要点）
    ├── dsh-provider.md           DSH 接入：provider 配置与模型上架
    ├── panel-plugin.md           插件看板：结构、数据源、slot 契约、踩坑
    ├── daily-claim.md            每日签到 / 活动 / 成长任务领取
    └── pitfalls.md               7 个真实故障的根因与修法 + 验证清单
```

## 脱敏声明

本手册**不含**任何真实密钥、账号、UID、Cookie、会话 ID 或本机绝对路径。
示例里的 `<API_KEY>` 请从你自己的配置文件读，不要写进版本库：

```bash
# 正确姿势：从配置读，不要硬编码
API_KEY=$(python -c "import json;print(json.load(open('config.json'))['api_key'])")
curl -s -H "authorization: Bearer $API_KEY" http://127.0.0.1:<PORT>/status
```

## License

MIT（见 [LICENSE](LICENSE)）
