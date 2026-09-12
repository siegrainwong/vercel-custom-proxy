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

注意：`args` 是固定值，无法按请求动态生成。官方文档对 `op: set` 的语义是「缺失时设置」，
即客户端已带该头时不覆盖；实测客户端自带值时请求同样返回 200（外部无法观测是否被覆盖）。
若将来需要动态值（如把客户端 session id 透传、或每次请求随机值），得改用 Routing Middleware（`middleware.ts`）。
