# vercel-custom-proxy

Vercel 上的反向代理，用于转发 DeepSeek 与 OpenCode (Zen/Go) 的请求。

## 路由

| 源路径 | 目标 |
| --- | --- |
| `/ds/*` | `https://api.deepseek.com/*` |
| `/oc/*` | `https://opencode.ai/zen/go/*` |

## 为什么 `/oc` 用 `routes` 而不是 `rewrites`

OpenCode Go 要求每个请求带上 `x-opencode-session` 请求头，缺失时返回：

```
400 {"error":{"type":"MissingSessionID","message":"... Request is missing x-opencode-session ..."}}
```

`rewrites` 的 `transforms` 只支持 `request.path`（路径改写），**不能改请求头**；
只有 `routes` 的 `transforms` 支持 `request.headers` / `request.query` / `response.headers`。

所以 `/oc` 写在 `routes` 里，用 transform 注入请求头：

```json
{
  "transforms": [
    {
      "type": "request.headers",
      "op": "set",
      "target": { "key": "x-opencode-session" },
      "args": "vercel-proxy"
    }
  ]
}
```

`routes` 与 `rewrites` 可以共存（Vercel 官方支持），因此 `/ds` 保持原来的 `rewrites` 不动。

注意：`args` 是固定值。若客户端自带真实 session id，此 transform 会把它设为 `vercel-proxy`
（`op: set` 语义），不影响可用性，只影响 opencode 侧的会话路由亲和性。
