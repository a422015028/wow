# 便捷下载 - Cloudflare Pages 部署版

基于逆向工程的"便捷下载"(com.lcw.easydownload) API 客户端，部署在 Cloudflare Pages 上。

**签名算法**: `sign = md5(md5(loginId + timestamp + token))`

## 功能

- 解析 60+ 平台视频链接，返回无水印下载地址
- YouTube 专用端点(自动路由)
- 会话自动持久化(KV 存储)，token 失效自动重新登录
- 短链自动转长链(快手等平台)
- 海外平台自动延长超时(60s)
- 网络错误自动重试 + 指数退避

## 项目结构

```
pages-project/
├── functions/
│   ├── index.js          # GET /  使用说明
│   ├── parse.js          # POST /parse  解析链接
│   ├── short2long.js     # POST /short2long  短链转长链
│   ├── health.js         # GET /health  健康检查
│   └── platforms.js      # GET /platforms  支持的平台列表
├── utils/
│   ├── md5.js            # 纯 JS MD5 实现(无依赖)
│   ├── client.js         # 核心客户端(登录/签名/解析)
│   ├── platforms.js      # 平台识别
│   └── response.js       # 响应/请求体解析工具
├── .assetsignore         # 部署时忽略文件
└── README.md             # 本文档
```

## 部署前配置(必做)

### 1. 创建 KV Namespace

在 Cloudflare Dashboard 中:
1. 进入 **Workers & Pages** → **KV** → **Create namespace**
2. 名称填 `easydownload-session`(或任意名称)
3. 创建后复制 **Namespace ID**

### 2. 设置环境变量和 KV Binding

在 Pages 项目的 **Settings** → **Variables**:

**Environment variables**(加密):
| 变量名 | 值 | 说明 |
|--------|----|------|
| `EASYDOWNLOAD_USERNAME` | 你的账号(邮箱/手机号) | 必选 |
| `EASYDOWNLOAD_PASSWORD` | 你的密码(明文,加密存储) | 必选 |
| `EASYDOWNLOAD_HOST` | `https://easydownload.flyinglife.cn` | 可选,默认即可 |
| `EASYDOWNLOAD_VERSION` | `1912` | 可选,App 版本号请求头(上游更新版本号时在此修改,无需改代码) |
| `DOWNLOAD_TOKEN_SECRET` | 任意强随机字符串 | 可选,/download 代理签名密钥(未设置时从账号密码派生) |

**KV namespace bindings**:
| Binding 名 | Namespace | 说明 |
|------------|-----------|------|
| `SESSION_KV` | `easydownload-session` | 必选,存储登录态 |

### 3. 兼容性设置(可选)

在 **Settings** → **Functions**:
- Compatibility date: `2026-06-16`(或默认)
- 无需额外 flags

## 部署方式(CloudFlareAssistant)

## 安全机制

- **下载代理签名**: `/download` 要求 `/parse` 签发的 HMAC-SHA256 token(24h 有效期),并校验 Origin/Referer 同源,防止被当作开放代理滥用
- **重登限流**: `/relogin` 每 IP 每分钟最多 3 次(KV 计数),防止刷爆上游登录接口

## 常见问题

**Q: 首次部署后解析失败?**
A: 检查 `/health` 是否返回 `loggedIn: true`。如果是 false，检查账号密码是否正确。

**Q: KV 中存储了什么?**
A: 只存储 `loginId`、`token`、`timeOffset`，不存储任何解析历史。

**Q: 可以换账号吗?**
A: 修改 Pages 环境变量中的 `EASYDOWNLOAD_USERNAME` / `EASYDOWNLOAD_PASSWORD`，然后清除 KV 中 `easydownload:session` 这个 key，下次请求会自动重新登录。
