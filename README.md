# cf-sv-proxy

短视频媒体直链代理，**Cloudflare Workers 版**。解决抖音、微博解析出来的视频/图片直链 **403 防盗链** 问题。

直链（如 `douyinvod.com`、`sinaimg.cn`）会校验请求头 `Referer`，直接在浏览器或自己的服务里引用会 403。本 Worker 在 Cloudflare 边缘节点带上正确的 `Referer` / `User-Agent` 拉取上游，再**流式转发**给你——零服务器、全球边缘加速、每天 10 万次免费额度。

> 代码源自 [jiuhunwl/short_videos](https://github.com/jiuhunwl/short_videos) 的 `cf_sv_proxy/worker.js`，原作者 JH-Ahua（[www.jiuhunwl.cn](https://www.jiuhunwl.cn)）。本仓库做了工程化封装（Wrangler 配置、一键部署、测试页），核心逻辑未改动。

---

## 特性

- **302 手动跟随**：最多 10 跳，每跳重新注入 `Referer` + `UA`（`redirect: follow` 跨域会丢 Referer，防盗链场景必须手动跟）
- **双平台预设**：`type=douyin|weibo`，自动匹配 Referer 与下载文件名前缀
- **双媒体预设**：视频走 `video/*` Accept + 下载头，图片走 `image/*` Accept + 24 小时缓存
- **类型自动探测**：不传 `mediatype` 时 HEAD 探测上游 Content-Type
- **两种 URL 输入**：Base64 编码（兼容 `+` 被转成空格）或明文 http(s) 直传
- **流式透传**：不落盘、不占 Worker 内存，不受响应体大小限制
- **CORS 全开放**：带 `OPTIONS` 预检，前端可直接跨域引用
- **可选域名白名单**：一行配置限定代理目标，防止被当任意资源代理滥用

---

## 部署

### 方式一：Wrangler CLI（推荐）

```bash
git clone https://github.com/xingmihai/cf-sv-proxy.git
cd cf-sv-proxy
npm install
npx wrangler login     # 浏览器授权 Cloudflare 账号
npm run deploy
```

部署完终端会输出地址：`https://cf-sv-proxy.<你的子域>.workers.dev`

本地调试：`npm run dev`，然后访问 `http://localhost:8787`

### 方式二：Cloudflare Dashboard（零工具）

1. [Cloudflare Dashboard](https://dash.cloudflare.com/) → **Workers 和 Pages** → **创建** → **创建 Worker**
2. 全选删除在线编辑器里的默认代码，粘贴本仓库 `worker.js` 全部内容
3. 点 **部署**，拿到 `https://xxx.workers.dev`

---

## 接口

```plaintext
GET /?proxyurl={媒体直链}&type={平台}&mediatype={媒体类型}&download={是否下载}
```

### 请求参数

| 参数 | 必填 | 说明 |
|------|:---:|------|
| `proxyurl` | ✅ | 媒体直链。支持 **Base64 编码**（推荐）或**明文 http(s) URL** |
| `type` | ❌ | `douyin`（默认）/ `weibo`，决定注入的 Referer 与文件名前缀 |
| `mediatype` | ❌ | `video` / `image`；不传则自动 HEAD 探测 |
| `download` | ❌ | `true`（默认）返回 `Content-Disposition: attachment` 触发下载；`false` 内联播放/显示 |

### 响应

**成功 200**：流式返回媒体内容，关键响应头：

| 响应头 | 说明 |
|--------|------|
| `Content-Type` | 透传上游，如 `video/mp4`、`image/jpeg` |
| `Content-Disposition` | `download=true` 时为 `attachment; filename="{platform}_{media}_{hash}.{ext}"` |
| `Cache-Control` | 图片恒 `public, max-age=86400` |
| `X-Proxy` | 代理渠道标识，如 `douyin-video-proxy` |

**失败**：返回 JSON，状态码与上游一致：

```json
{"code": 403, "msg": "请求失败，HTTP状态码: 403（可能需要更新Cookie或Referer头信息）"}
```

| 状态码 | 场景 |
|:---:|------|
| 400 | `proxyurl` 缺失 / URL 无效 / Base64 解码失败 |
| 403 | 上游防盗链拦截（含白名单拒绝） |
| 404 | 上游资源不存在或直链已过期 |
| 500 | 网络错误 / 超时（30s） |

### 调用示例

```bash
# 明文 URL，代理抖音视频（默认触发下载）
curl -L -o video.mp4 "https://xxx.workers.dev/?proxyurl=https://aweme.snssdk.com/aweme/v1/play/?video_id=xxxx%26ratio=default"

# Base64 编码 URL（PHP：base64_encode($url)）
curl -L -o video.mp4 "https://xxx.workers.dev/?proxyurl=aHR0cHM6Ly9hd2VtZS5zbnNzZGsuY29tL..."

# 图片内联显示，不触发下载
curl -o cover.jpg "https://xxx.workers.dev/?proxyurl=<base64>&mediatype=image&download=false"

# 微博视频
curl -L -o weibo.mp4 "https://xxx.workers.dev/?proxyurl=<base64>&type=weibo"
```

**接在解析接口后面用**：解析出的视频 `url` 一旦 403，Base64 后拼进来即可

```php
https://xxx.workers.dev/?proxyurl=<?php echo base64_encode($videoUrl); ?>
```

---

## 配置

改 `worker.js` 顶部常量：

| 常量 | 默认值 | 说明 |
|------|--------|------|
| `ALLOWED_HOST_SUFFIXES` | `[]`（不限制） | 目标域后缀白名单。**公开部署建议填**：`['douyinvod.com','douyinpic.com','snssdk.com','weibo.com','sinaimg.cn']` |
| `REQUEST_TIMEOUT` | `30000` | 媒体拉取超时 ms（302 每跳独立计时） |
| `PROBE_TIMEOUT` | `15000` | HEAD 探测超时 ms |
| `MAX_REDIRECTS` | `10` | 302 最大跟随跳数 |

---

## 注意

1. **不是破解**：只是以合法 Referer 代为请求。平台若升级签名校验（带 `x-signature` 的链接），仍可能 403，需重新解析拿新直链。
2. **直链有时效**：抖音 CDN 直链带 `x-expires`，过期返回 404。实时解析、即时代理，别缓存直链。
3. **配额**：免费版每日 10 万次请求；每次代理消耗 1 个子请求，302 每跳另计。
4. **防滥用**：默认不限制目标域。公开部署务必配白名单，否则会被当免费图床/代理刷爆额度。

---

## License

MIT
