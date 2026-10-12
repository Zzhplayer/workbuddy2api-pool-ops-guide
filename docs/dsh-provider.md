# 4. 接入 DSH：provider 配置与模型上架

把本地网关当成一个 provider 接进 DSH，并让它的模型出现在**模型选择器**里。

---

## 4.1 两层配置（改错层会「看起来生效其实没生效」）

DSH 的 profile 目录里通常有**两层**：

| 文件 | 角色 | 说明 |
|---|---|---|
| `<PROFILE_DIR>/settings.yaml` | **用户层**（覆盖层） | 你手写的最终值，优先级最高 |
| `<PROFILE_DIR>/cordis.patch.yml` | **组合层**（patch） | 参与组合；插件/装配也会往这里写 |
| `<PROFILE_DIR>/cordis.yml` | **生成物** | 运行时合成结果，**不要手改**（会被覆盖） |

> 实测坑：只改一层时，另一层可能把它覆盖回去。
> **两层的 provider 段都要改**，改完用「运行时真值」验证（见 4.5）。

## 4.2 provider 段骨架

```yaml
llm-<driver>:
  providers:
    <PROVIDER_ID>:
      apiKeyEnv: <ENV_VAR_NAME>        # 从环境变量取密钥，别把 key 写进文件
      displayName: <显示名>
      api: openai-completions          # 网关是 OpenAI 兼容
      baseURL: http://127.0.0.1:<PORT>/v1
      defaultContextWindow: 1000000
      defaultMaxTokens: 131072
      compat:
        supportsStore: false
        supportsDeveloperRole: false
        supportsReasoningEffort: true
        maxTokensField: max_tokens
        thinkingFormat: openai         # 思考内容走 reasoning_content
      defaultInput: [text, image]
      models:
        - id: cn:<MODEL_A>             # ← 这个 id 会原样发给网关
          name: <显示名> (Local Gateway)
          contextWindow: 1000000
          maxTokens: 393216
          input: [text, image]
          reasoningEfforts:            # DSH 的思考档位 → 上游取值
            off: null                  # null = 不发送 reasoning_effort
            low: low
            high: high
            max: max
```

## 4.3 模型条目的字段含义

| 字段 | 作用 | 注意 |
|---|---|---|
| `id` | **原样发给网关的模型名** | 必须与 `/v1/models` 里的一致 |
| `name` | 选择器里显示的名字 | 建议带上来源后缀，便于区分同 id 不同后端 |
| `contextWindow` / `maxTokens` | 上下文与**输出上限** | `maxTokens` 太小会导致「思考吃满预算 → content 为空」 |
| `input` | `text` / `image` | 只写上游真支持的 |
| `reasoningEfforts` | 档位映射 | 见 4.4 |

## 4.4 思考档位：两个真实陷阱

**陷阱 A：`only_reasoning` 模型不能「关思考」。**
上游标了 `can_disable_thinking: false` 的模型，你映射 `off: null` 也能跑（网关会把 `off` 兜到 `low`），
但**别指望它真的不思考**：实测单轮 85~326 秒、TTFB 15~60 秒、吞吐 1~4 tok/s。

**陷阱 B：思考 token 吃满输出预算时，`content` 会是空的。**
实测：某主力模型在 `max_tokens=4000` 时 100% 复现 —— `reasoning` 6388 字、`content` **0 字**、
`finish_reason=length`。客户端表现是「**已完成，但什么都没有**」。

对策：
1. 给这类模型留**足够大的 `maxTokens`**；
2. 需要稳出内容时，把档位映射到更低的思考强度（例如 `high: medium`）；
3. 慢/只推理的模型**不要放在自动链首位**（见 [pool-and-fallback.md](pool-and-fallback.md)）。

## 4.5 上架与验证

```bash
# ① 网关侧确实有这个模型（拿真 id）
curl -s -H "authorization: Bearer $API_KEY" http://127.0.0.1:<PORT>/v1/models \
  | python -c "import json,sys;[print(m['id'], m.get('name')) for m in json.load(sys.stdin)['data']]"

# ② 真打一发（别只看清单）
curl -s -H "authorization: Bearer $API_KEY" -H 'content-type: application/json' \
  -d '{"model":"cn:<MODEL_A>","messages":[{"role":"user","content":"1+1=?"}],"max_tokens":32}' \
  http://127.0.0.1:<PORT>/v1/chat/completions
```

**③ 运行时真值**（最关键的一步）—— 让宿主自己报「我认到哪些模型」：

```
modelCatalog()  →  { default: {provider, model}, routableProviders: [...], groups: [{id, name, models:[...]}] }
```

看 `groups` 里你那个 provider 的 `models` 数组：**顺序就是选择器里的顺序**，新增的模型应该在预期位置。
> 这一步能立刻区分「配置写了」和「运行时真的认了」——两者经常不一致。

**④ 默认模型**（新建会话用哪个）：

```
currentSelection()  →  { provider, model, reasoningEffort? }
saveSelection({ provider, model, reasoningEffort })   // 与 GUI 选择器同一条路径，即时生效
```

**⑤ 顺手核对**：把 provider 的模型列表顺序当作「默认档位暗示」来看 ——
实测把某个模型插到列表**第一位**后，新建会话的默认模型也跟着变成了它。
所以**上架顺序会改变默认行为**，别随手插到最前面。

## 4.6 安全

- 密钥走 `apiKeyEnv`（环境变量），配置文件里只留变量名；
- 本手册与仓库**不含**任何真实密钥；脚本里统一从配置读：

```bash
API_KEY=$(python -c "import json;print(json.load(open('<GATEWAY_DIR>/config.json'))['api_key'])")
```

- 网关只绑 `127.0.0.1`；确需跨机访问时再加鉴权与网络层限制。
