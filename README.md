<div align="center">

<img src="https://icc.gt.tc/xiaoxiangimage/logo.jpg" width="80" height="80" style="border-radius:16px" alt="小象部署 Logo">

# 小象部署

**粘贴一段 HTML，立刻得到一个 HTTPS 子域名。**

基于 Cloudflare Workers + Workers KV，零服务器、零构建、零依赖。

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Cloudflare Workers](https://img.shields.io/badge/Runtime-Cloudflare%20Workers-F38020?logo=cloudflare&logoColor=white)](https://workers.cloudflare.com/)
[![Zero Dependencies](https://img.shields.io/badge/Dependencies-0-brightgreen.svg)](worker.js)

</div>

## 功能

- 粘贴 HTML 一键部署，随机 10 位 ID 或自定义子域名（3–63 位，字母 / 数字 / `_` / `-`）
- 主链接 `https://{id}.xxhost.de5.net`，兼容链接 `https://xxhost.de5.net/s/{id}.html`
- 旧链接继续有效；旧站显示关闭页，`/s/` 保持只读
- 全球 CDN 加速、免费、无广告、无追踪

## 原理

**三种入口查同一把 KV key**，所以不需要数据迁移：

```
KV:  "test"  →  "<h1>hi</h1>"

旧站   https://host.smallelephant.de5.net/s/test.html  → id  = "test" → kv.get("test")
新站   https://test.xxhost.de5.net/                    → sub = "test" → kv.get("test")
兼容   https://xxhost.de5.net/s/test.html              → id  = "test" → kv.get("test")
```

一次请求的链路：

1. **DNS**：`A *` → `192.0.2.1`（橙云）。这是保留地址，请求在边缘被交给 Worker，不会回源。
2. **TLS**：Universal SSL 覆盖 `*.xxhost.de5.net`，证书自动生效。
3. **Route**：`*.xxhost.de5.net/*` 把请求派发给 Worker。
4. **Worker**：读 `CF-Connecting-Host`（回退 `Host`），去端口、转小写、去尾点，截取 `test` 作为 KV key。
5. **响应**：返回 HTML，带 `Cache-Control: max-age=300`。

根域用 Custom Domain（`xxhost.de5.net`），Cloudflare 自动创建指向 Worker 的 CNAME 并签发证书，覆盖根域所有路径。子域用通配 DNS + Route，因为 Custom Domain 不支持通配符、slug 是运行时生成的。含点的子域会被拒绝，因为 Universal SSL 只覆盖一级子域。

## 配置过程

**1. 域名接入**：把 `xxhost.de5.net` 以 Full setup（NS 委派）接入 Cloudflare，状态 Active。

**2. 创建 KV**：`Storage and databases → Workers KV → Create a namespace`，命名 `host`，记下 Namespace ID。

**3. 创建 Worker**：创建 Worker，在 `Settings → Bindings → KV Namespace` 添加绑定，变量名填 `host`，选择第 2 步的 namespace。把 `worker.js` 内容粘贴到 `Edit Code` 并 Deploy；也可用仓库自带的 `wrangler.jsonc` 直接 `wrangler deploy`（KV 绑定已配好）。

**4. DNS 记录**（手动只需一条，橙云 Proxied）：

| 类型 | Name | 值 | 代理 |
|---|---|---|---|
| A | `*` | `192.0.2.1` | 橙云 |

> 根域 `xxhost.de5.net` **不要**手动加 `A @`：给 Worker 添加 Custom Domain 时 Cloudflare 会自动创建指向 Worker 的 CNAME 并签发证书；手加 A/CNAME 记录会与 Custom Domain 冲突。精确记录优先于通配记录，避免 slug 与已有 DNS 名冲突。

**5. Worker 域名绑定**（都指向 `xxhost`）：

- Custom Domain：`xxhost.de5.net`（根域，Cloudflare 自动建 CNAME + 证书，覆盖全部路径）
- Route：`*.xxhost.de5.net/*`（子域，配合上面的 `A *` 通配记录）

**6. SSL/TLS**：加密模式 `Full (strict)`，开启 `Always Use HTTPS`。

**7. 验证**：

```bash
curl -I https://xxhost.de5.net/                       # 200 部署界面
curl -X POST https://xxhost.de5.net/api/deploy \
  -F action=deploy -F 'html=<h1>test</h1>' -F slug=test
curl -I https://test.xxhost.de5.net/                  # 200 部署内容
curl -I https://xxhost.de5.net/s/test.html            # 200 兼容回退
```



## 用法

**部署**

```bash
curl -X POST https://xxhost.de5.net/api/deploy \
  -F action=deploy \
  -F 'html=<h1>Hello</h1>' \
  -F slug=hello
```

```json
{
  "success": true,
  "url": "https://hello.xxhost.de5.net",
  "pathUrl": "https://xxhost.de5.net/s/hello.html",
  "filename": "hello.html",
  "size": 16,
  "isCustom": true
}
```

`slug` 留空则生成随机 ID。服务端校验正则 `/^[a-zA-Z0-9_-]{1,63}$/`、至少 3 个字符，并拦截 `__meta__` / `__deploy_meta__` 与已占用的名字。部署入口 `POST /` 与 `POST /api/deploy` 等价。

**删除**

```bash
curl -X POST https://xxhost.de5.net/api/delete -F id=hello
```

> 部署与删除接口目前无鉴权，公开使用前建议加 Turnstile、Token 或限流。

**访问**

```bash
https://hello.xxhost.de5.net/                    # 主链接
https://xxhost.de5.net/s/hello.html              # 兼容回退
https://host.smallelephant.de5.net/s/hello.html  # 旧站只读（同一 KV）
```

部署记录保存在浏览器 `localStorage`，不会上传服务端。

## License

[Apache License 2.0](LICENSE) · Copyright 2026 devxiaoxiang
