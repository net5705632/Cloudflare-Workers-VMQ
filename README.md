# Cloudflare Workers V免签支付系统

> 🚀 基于 Cloudflare Workers + D1 数据库实现的零成本 V免签个人收款服务端
>
> 复刻自 [szvone/Vmqphp](https://github.com/szvone/Vmqphp)，无需服务器，无需 MySQL，部署即用。

---

## 📖 项目简介

这是一个用 **Cloudflare Workers** 重新实现的 **V免签服务端**，用于个人开发者/博客搭建**免签约收款系统**。

**V免签原理**：

```
用户扫码付款 → 手机/电脑收到收款通知 → 监控端监听到通知 → 推送到服务端
                                                          ↓
                           服务端按金额匹配订单 → 回调商户系统 → 完成交易闭环
```

**核心优势**：

- ✅ **零服务器成本**：Cloudflare Workers 免费额度 10 万次请求/天，个人博客足够
- ✅ **零运维**：D1 数据库全托管、Workers 自动扩缩容
- ✅ **全球加速**：Cloudflare 边缘节点自动分发
- ✅ **开源透明**：所有代码可审计、可二次开发
- ✅ **支持多端**：微信 / 支付宝

---

## 🏗️ 系统架构

```
┌──────────────┐      ① 下单     ┌────────────────┐
│  博客系统     │ ───────────────→│  V免签 Workers  │
│ (任意 Workers) │ ←─────────────── │  (本项目)      │
│              │   ⑥ 异步回调      │                │
└──────────────┘                  └────────────────┘
                                          ↑
                                          │ ④ appPush
                                          │
                                    ┌──────────────┐
                                    │ Python 监控端 │
                                    │ (OCR识别金额) │
                                    └──────────────┘
                                          ↑
                                          │ ③ 收款通知
                                          │
                                    ┌──────────────┐
                                    │ 微信电脑版    │
                                    │ 收款助手窗口  │
                                    └──────────────┘
```

---

## 📁 目录结构

```
cloudflare-vmq/
├── src/
│   ├── index.js          # 主入口 + 路由
│   ├── admin.js          # 管理后台
│   ├── api.js            # V免签核心 API
│   ├── db.js             # D1 数据库操作
│   ├── md5.js            # 纯 JS MD5 实现
│   └── templates.js      # HTML 模板
├── schema.sql            # D1 数据库结构
├── wrangler.toml         # Wrangler 配置
├── package.json
└── README.md
```

---

## 🚀 快速开始

### 前置要求

- Node.js 18+ 和 npm
- Cloudflare 账号（免费）
- [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/install-update/)

```bash
npm install -g wrangler
wrangler login
```

### 1. 克隆项目

```bash
git clone https://github.com/yourname/cloudflare-vmq.git
cd cloudflare-vmq
npm install
```

### 2. 创建 D1 数据库

```bash
wrangler d1 create vmq-db
```

返回结果类似：

```
✅ Successfully created DB 'vmq-db' in region APAC
database_id = "bcb9c2db-8837-468b-9aff-d044a6a2c93a"
```

**记下 `database_id`**，稍后填入 `wrangler.toml`。

### 3. 配置 `wrangler.toml`

```toml
name = "vmq-workers"
main = "src/index.js"
compatibility_date = "2024-01-01"

[[d1_databases]]
binding = "DB"
database_name = "vmq-db"
database_id = "bcb9c2db-8837-468b-9aff-d044a6a2c93a"  # ← 换成你的 ID
```

> ⚠️ **注意**：`database_id` 结尾**必须有双引号**，否则 wrangler 会报 TOML 解析错误。

### 4. 初始化数据库

```bash
# 本地初始化（测试用）
wrangler d1 execute vmq-db --file=./schema.sql

# 远程初始化（部署后必须执行）
wrangler d1 execute vmq-db --file=./schema.sql --remote
```

> ⚠️ **关键**：不加 `--remote` 只初始化本地模拟库，Cloudflare 线上库是空的。**必须执行 `--remote` 版本**。

### 5. 部署

```bash
wrangler deploy
```

### 6. 访问管理后台

打开 `https://你的域名/admin/login`

**默认账号**：
- 用户名：`admin`
- 密码：`admin123`

**⚠️ 首次登录后立即修改密码！**

---

## 🔌 API 接口

### 1. 创建订单

```
GET /createOrder
```

**参数**：

| 参数 | 必填 | 说明 |
|------|------|------|
| `payId` | ✅ | 商户订单号（唯一） |
| `type` | ✅ | 支付方式：1=微信，2=支付宝 |
| `price` | ✅ | 订单金额（两位小数） |
| `sign` | ✅ | 签名：`md5(payId + param + type + price + key)` |
| `param` | ❌ | 附加参数（通常传文章 ID） |
| `notifyUrl` | ❌ | 异步回调地址 |
| `returnUrl` | ❌ | 同步跳转地址 |
| `isHtml` | ❌ | 是否直接返回 HTML 支付页（`1` 开启） |

**示例**：

```bash
curl "https://your-domain.com/createOrder?\
payId=CF1234567890&\
type=1&\
price=0.02&\
sign=abc123...&\
param=000151&\
isHtml=1&\
notifyUrl=https://your-blog.com/pay/notify&\
returnUrl=https://your-blog.com/pay/return"
```

**响应**：

```json
{
  "code": 1,
  "msg": "创建成功",
  "orderId": "V1790123456789",
  "price": 0.02,
  "payUrl": "/pay/qrcode?orderId=V1790123456789"
}
```

---

### 2. 监控端推送

```
POST /appPush
```

**参数**（form 表单）：

| 参数 | 说明 |
|------|------|
| `type` | 支付方式：1=微信，2=支付宝 |
| `price` | 实付金额 |
| `t` | 时间戳（秒） |
| `sign` | 签名：`md5(type + price + t + key)` |

**示例**：

```bash
curl -X POST "https://your-domain.com/appPush" \
  -d "type=1" \
  -d "price=0.02" \
  -d "t=1790123456" \
  -d "sign=abc123..."
```

**响应**：
- `success`：推送成功，订单匹配
- `no matching order`：未找到匹配订单
- `sign error`：签名错误

---

### 3. 监控端心跳

```
POST /appHeart
```

**参数**：

| 参数 | 说明 |
|------|------|
| `t` | 时间戳（秒） |
| `sign` | 签名：`md5(t + key)` |

---

### 4. 查询订单状态

```
GET /api/order/status?orderId=V1790123456789
```

**响应**：

```json
{
  "code": 1,
  "data": {
    "payId": "CF1234567890",
    "orderId": "V1790123456789",
    "price": 0.02,
    "reallyPrice": 0.02,
    "state": 1,
    "createTime": 1790123456,
    "payTime": 1790123457
  }
}
```

---

### 5. 异步回调（服务端 → 商户）

当订单支付成功后，V免签会主动请求商户的 `notifyUrl`：

```
GET {notifyUrl}?payId=xxx&param=xxx&type=1&price=0.02&reallyPrice=0.02&sign=xxx
```

**签名**：`md5(payId + param + type + price + reallyPrice + key)`

**商户必须返回**：纯文本 `success`（否则 V免签会反复重试）

---

## 📱 监控端配置

监控端（Python）推荐使用本项目配套的 Windows 版本：

- 仓库：[cloudflare-vmq-monitor](https://github.com/yourname/cloudflare-vmq-monitor)
- 原理：截图"微信收款助手"窗口 → OCR 识别金额 → 推送到 V免签服务端

**核心配置**：

```python
VMQ_URL = "https://zf.hiir.cn/appPush"
VMQ_HEART_URL = "https://zf.hiir.cn/appHeart"
VMQ_KEY = "changeme_xxx"  # 从管理后台复制

WINDOW_TITLE = "微信收款助手"
```

**运行环境**：
- Windows 10/11
- Python 3.8+
- Tesseract-OCR 5.x
- 微信电脑版（保持"微信收款助手"窗口可见）

---

## ⚙️ 管理后台

访问 `https://你的域名/admin/`

**功能**：

- 📊 **数据总览**：订单总数、已支付、待支付、今日收入、监控端状态
- 📋 **订单管理**：查看所有订单、筛选、手动补单
- ⚙️ **系统设置**：修改通讯密钥、回调地址、超时时间、管理员账号密码

---

## 🔧 常见问题（FAQ）

### Q1: `D1_ERROR: no such table: config`

**原因**：D1 只初始化了本地，线上库是空的。

**解决**：
```bash
wrangler d1 execute vmq-db --file=./schema.sql --remote
```

---

### Q2: `MD5 is not supported`

**原因**：Cloudflare Workers 的 `crypto.subtle` **不支持 MD5**。

**解决**：项目已内置纯 JS 版 MD5（`src/md5.js`），无需修改。

---

### Q3: `sign error`（签名错误）

**排查顺序**：

1. **确认 `vmq_key` 一致**：管理后台 → 系统设置 → 通讯密钥
2. **确认所有参数拼接前用 `String()`**：避免 JS 数字相加 `1 + 0.02 = 1.02`
3. **确认 `price` 用 `.toFixed(2)`**：确保 `8.00` 而不是 `8`
4. **打开 tail 日志对比两边的签名输入**：

```bash
wrangler tail
```

```
=== createOrder 验签 === {
  payId: 'CF1790141027822953',
  param: '000149',
  type: '1',
  price: '8.00',
  signInput: 'CF179014102782295300014918.00changeme_xxx',
  sign: '37c9ee...',
  expectSign: '37c9ee...'   ← 一致才能通过
}
```

---

### Q4: `no matching order`（未匹配到订单）

**原因**：监控端推送的金额和订单金额不一致。

**排查**：
```bash
wrangler d1 execute vmq-db --command="SELECT order_id, price, state FROM orders WHERE state = 0;" --remote
```

**常见场景**：
- 订单被**金额递增机制**加了 0.01（个人博客可关闭此逻辑）
- OCR 识别到的金额格式不同（`8.0` vs `8.00`）
- 订单已过期（超过 300 秒）

---

### Q5: `通知商户超时（8秒）`

**原因**：V免签 Worker 用 `fetch` 请求博客域名时，走 DNS 解析到了**优选 IP（CDN 中转服务器）**，请求打不到博客 Worker。

**解决**：**Worker 之间 fetch 必须用 Cloudflare 原生域名**。

```javascript
// notifyUrl 用原生域名
const notifyUrl = 'https://cfblog.777171.xyz/pay/notify';

// returnUrl 给用户浏览器用，保持优选 IP 域名
const returnUrl = url.origin + '/pay/return?payId=' + payId;
```

**或者用 Service Bindings**（推荐，不走公网）：

```toml
[[services]]
binding = "BLOG_WORKER"
service = "cfblog"
```

```javascript
const resp = await env.BLOG_WORKER.fetch(fullUrl, { method: 'GET' });
```

---

### Q6: 支付页面一直"等待支付确认"

**排查顺序**：

1. **监控端在线吗**：管理后台 → 数据总览 → 监控端状态
2. **监控端推送成功了吗**：看 Python 控制台有没有 `推送成功`
3. **订单状态变了没**：查询 D1
   ```bash
   wrangler d1 execute vmq-db --command="SELECT state FROM orders WHERE order_id='V1790141028590260';" --remote
   ```
4. **订单状态是 1 但页面没跳转**：说明前端轮询超时，可能是 V免签 → 博客的通知超时

---

### Q7: 监控端 OCR 识别不到金额

**原因**：截图裁剪区没包含金额。

**解决**：打开 Python 脚本同目录下的 `debug_crop.png`，看截图内容是否包含完整的 `¥X.XX`。

**调整裁剪比例**：

```python
CROP_LEFT_RATIO   = 0.10
CROP_RIGHT_RATIO  = 0.90
CROP_TOP_RATIO    = 0.05
CROP_BOTTOM_RATIO = 0.75   # ← 金额在窗口 75% 位置，别切掉
```

---

## 📊 数据库结构

### `orders` 表

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | INTEGER | 主键 |
| `pay_id` | TEXT | 商户订单号（唯一） |
| `order_id` | TEXT | 云端订单号（唯一） |
| `type` | INTEGER | 1=微信，2=支付宝 |
| `price` | REAL | 订单金额 |
| `really_price` | REAL | 实际支付金额 |
| `state` | INTEGER | 0=待支付，1=已支付，2=通知失败，3=已过期，4=补单 |
| `param` | TEXT | 附加参数 |
| `notify_url` | TEXT | 异步回调地址 |
| `return_url` | TEXT | 同步跳转地址 |
| `create_time` | INTEGER | 创建时间戳 |
| `pay_time` | INTEGER | 支付时间戳 |
| `notify_time` | INTEGER | 通知时间戳 |
| `notify_count` | INTEGER | 通知次数 |

### `config` 表

| key | value 示例 | 说明 |
|-----|----------|------|
| `vmq_key` | `changeme_xxx` | 通讯密钥 |
| `admin_user` | `admin` | 管理员账号 |
| `admin_pass` | `admin123` | 管理员密码 |
| `site_name` | `V免签` | 站点名称 |
| `notify_url` | `https://...` | 默认回调地址 |
| `order_timeout` | `300` | 订单超时（秒） |

### `monitor_status` 表

| 字段 | 说明 |
|------|------|
| `last_heartbeat` | 最后心跳时间 |
| `version` | 监控端版本 |
| `status` | 在线状态 |

---

## 🛠️ 开发调试

### 本地开发

```bash
wrangler dev
```

访问 `http://localhost:8787`

### 实时日志

```bash
wrangler tail
```

### 数据库查询

```bash
# 查询最近 10 条订单
wrangler d1 execute vmq-db --command="SELECT * FROM orders ORDER BY create_time DESC LIMIT 10;" --remote

# 清理过期订单
wrangler d1 execute vmq-db --command="UPDATE orders SET state = 3 WHERE state = 0 AND create_time < strftime('%s','now') - 300;" --remote

# 导出备份
wrangler d1 export vmq-db --remote --output=vmq-backup.sql
```

---

## 🔐 安全建议

1. **首次登录立即改密码**：管理后台 → 系统设置 → 修改密码
2. **定期更换 `vmq_key`**：管理后台 → 系统设置 → 通讯密钥
3. **`vmq_key` 不要泄露**：这是通讯密钥，泄露后别人可以伪造推送
4. **生产环境推荐用 Service Bindings**：避免走公网，更安全
5. **定期备份 D1 数据**：`wrangler d1 export`
6. **只用于个人开发者场景**：V免签是个人方案，不适合商用多用户

---

## 📝 版本历史

### v1.0.0

- ✅ 实现 V免签核心协议（createOrder / appPush / appHeart / notify）
- ✅ 管理后台（订单管理 / 系统设置 / 数据统计）
- ✅ 支持手动补单
- ✅ 兼容 POST form 和 GET query 两种推送格式
- ✅ 金额匹配支持微小差异兜底

---

## 📄 许可证

MIT License

Copyright (c) 2026

---

## 🙏 致谢

- [szvone/Vmqphp](https://github.com/szvone/Vmqphp) - V免签原版 PHP 服务端
- [szvone/VmqApk](https://github.com/szvone/VmqApk) - V免签原版安卓监控端
- [Cloudflare Workers](https://workers.cloudflare.com/) - 提供的免费 Workers 服务
- [Cloudflare D1](https://developers.cloudflare.com/d1/) - 提供的免费 SQLite 数据库

---

## ⚠️ 免责声明

本项目仅供**个人开发者学习、测试、个人博客收款**使用。请勿用于非法用途，商用请申请官方商户接口。使用本项目所产生的一切后果由使用者自行承担。

---

## 📮 反馈与交流

- 提交 [Issue](https://github.com/yourname/cloudflare-vmq/issues)
- 提交 [Pull Request](https://github.com/yourname/cloudflare-vmq/pulls)

---

**⭐ 如果这个项目对你有帮助，欢迎 Star 支持！**
